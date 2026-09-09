# Lab 5: Build a Movie Agent

## Introduction

The duality view now gives an application one document-shaped interface to the movie JSON and the E5-base search metadata. In this lab, you place a natural-language interface on top of that same view. A user asks a question such as “What movie is about a rescue mission?”, a database function turns the question into an E5-base query embedding, and semantic search returns the most relevant movie documents. Select AI Agent then uses those documents as context for its response.

This separates retrieval from generation. `MULTILINGUAL_E5_BASE` is the embedding model used to find relevant movies; the generative model configured in the Select AI profile is a separate model used to reason over the retrieved context and write the answer. The function is deliberately narrow: it returns movie documents from `MOVIES_DV`, so the agent has a controlled source of movie facts instead of unrestricted access to the database.

Select AI setup depends on the provider, generative model, credential, database privileges, and network configuration approved for your environment. The SQL below uses placeholders for those environment-specific values and keeps secrets out of the workshop source. If your administrator already created an approved Select AI profile, use that profile name and skip the profile-creation template.

Estimated Time: 30 minutes

### Objectives

- Configure an approved Select AI profile.
- Create a movie-search function that queries the duality view.
- Register the function as a Select AI tool.
- Create an agent and task that call the movie-search tool.
- Ask a natural-language question and inspect the grounded response.

## Task 1: Prepare Select AI

1. Confirm that the required Select AI privileges are available.

    A database administrator may need to grant the following privileges to the workshop schema. Do not run these statements as the workshop user unless you have administrator privileges.

    ```sql
    GRANT EXECUTE ON DBMS_CLOUD TO <WORKSHOP_USER>;
    GRANT EXECUTE ON DBMS_CLOUD_AI TO <WORKSHOP_USER>;
    GRANT EXECUTE ON DBMS_CLOUD_AI_AGENT TO <WORKSHOP_USER>;
    ```

2. Create a credential only if the credential required by your approved Select AI profile does not already exist.

    Replace the placeholders with the credential format approved for your provider. Never commit a real secret to the workshop files.

    ```sql
    BEGIN
      DBMS_CLOUD.CREATE_CREDENTIAL(
        credential_name => 'SELECT_AI_CRED',
        username        => '<PROVIDER_USER_OR_OCID>',
        password        => '<PROVIDER_API_KEY_OR_AUTH_TOKEN>'
      );
    END;
    /
    ```

3. Create the Select AI profile used by the movie agent.

    This OCI Generative AI template identifies the profile, credential, model, region, compartment, and duality view that the agent can use. Replace `<APPROVED_GENERATIVE_MODEL>`, `<APPROVED_OCI_REGION>`, `<OCI_COMPARTMENT_OCID>`, `<WORKSHOP_SCHEMA>`, and any provider-specific values before running it. If your environment uses another approved provider, keep the profile name but use that provider's profile attributes.

    ```sql
    BEGIN
      DBMS_CLOUD_AI.CREATE_PROFILE(
        profile_name => 'MOVIE_SELECT_AI_PROFILE',
        attributes   => '{
          "provider": "oci",
          "credential_name": "SELECT_AI_CRED",
          "model": "<APPROVED_GENERATIVE_MODEL>",
          "region": "<APPROVED_OCI_REGION>",
          "oci_compartment_id": "<OCI_COMPARTMENT_OCID>",
          "object_list": [
            {"owner": "<WORKSHOP_SCHEMA>", "name": "MOVIES_DV"}
          ],
          "temperature": 0
        }'
      );
    END;
    /
    ```

    The profile's generative model is used for the final response. It is independent of the E5-base ONNX model used by the movie-search function.

4. Set the profile for the current session and test that Select AI can call its configured generative model.

    ```sql
    EXEC DBMS_CLOUD_AI.SET_PROFILE('MOVIE_SELECT_AI_PROFILE');

    SELECT AI "Tell me about movies with superheroes saving others";
    ```

    Continue when Select AI returns a response from the configured model. If the profile or credential is not available, ask your database administrator to provide the approved profile name and required privileges.

## Task 2: Create the movie-search tool

1. Create a database function that accepts a natural-language question, generates an E5-base query embedding, and searches the duality view.

    The function returns the five closest movie documents as newline-delimited JSON. It searches the top-level `embedding` field exposed by `MOVIES_DV`, which maps to the relational `MOVIES.EMBEDDING_E5_BASE` column created in Lab 3. It returns only the original `movie` object as context, leaving the 768-dimensional vector out of the prompt because the vector is retrieval metadata rather than movie content.

    ```sql
    CREATE OR REPLACE FUNCTION search_movies (
      p_query IN CLOB
    ) RETURN CLOB
    AUTHID DEFINER
    IS
      l_context CLOB;
    BEGIN
      DBMS_LOB.CREATETEMPORARY(l_context, TRUE);

      FOR movie_row IN (
        SELECT JSON_QUERY(m.data, '$.movie' RETURNING CLOB) AS movie_document
        FROM movies_dv m
        ORDER BY VECTOR_DISTANCE(
                   JSON_VALUE(
                     m.data,
                     '$.embedding'
                     RETURNING VECTOR(768, FLOAT32)
                   ),
                   VECTOR_EMBEDDING(
                     MULTILINGUAL_E5_BASE
                     USING p_query AS data
                   ),
                   COSINE
                 )
        FETCH FIRST 5 ROWS ONLY
        WITH TARGET ACCURACY 90
      ) LOOP
        DBMS_LOB.APPEND(l_context, movie_row.movie_document);
        DBMS_LOB.WRITEAPPEND(l_context, 1, CHR(10));
      END LOOP;

      RETURN l_context;
    END;
    /
    ```

2. Register the function as a Select AI Agent tool.

    The tool instruction tells the agent what the function returns and which argument to populate. The agent can discover the function argument schema through the tool definition.

    ```sql
    BEGIN
      DBMS_CLOUD_AI_AGENT.CREATE_TOOL(
        tool_name   => 'MOVIE_SEARCH_TOOL',
        attributes  => '{
          "instruction": "Retrieve the five most relevant movie documents from MOVIES_DV. Pass the complete natural-language movie request in the p_query argument.",
          "function": "SEARCH_MOVIES"
        }',
        description => 'Semantic movie retrieval over the MOVIES_DV duality view.'
      );
    END;
    /
    ```

3. Inspect the registered tool before the agent uses it.

    This verifies the retrieval function metadata before the agent is introduced. The agent will invoke the tool with the user's question in Task 4.

    ```sql
    DECLARE
      l_tool_description CLOB;
    BEGIN
      l_tool_description := DBMS_CLOUD_AI_AGENT.DESCRIBE_TOOL(
                              tool_name => 'MOVIE_SEARCH_TOOL'
                            );
      DBMS_OUTPUT.PUT_LINE(l_tool_description);
    END;
    /
    ```

    The next task connects the tool to an agent and task.

## Task 3: Create the movie agent and team

1. Create an agent that uses the approved Select AI profile and is responsible for grounded movie answers.

    ```sql
    BEGIN
      DBMS_CLOUD_AI_AGENT.CREATE_AGENT(
        agent_name  => 'MOVIE_ANALYST',
        attributes  => '{
          "profile_name": "MOVIE_SELECT_AI_PROFILE",
          "role": "You are a movie search assistant. Use MOVIE_SEARCH_TOOL to ground every movie fact. Do not invent titles, years, genres, or subjects. If the retrieved documents do not answer the question, say that no matching movie was found."
        }',
        description => 'Answers movie questions using semantic retrieval from MOVIES_DV.'
      );
    END;
    /
    ```

2. Create a task that calls the retrieval tool before writing the answer.

    ```sql
    BEGIN
      DBMS_CLOUD_AI_AGENT.CREATE_TASK(
        task_name  => 'ANSWER_MOVIE_TASK',
        attributes => '{
          "instruction": "Answer the user movie question: {query}. First call MOVIE_SEARCH_TOOL with p_query set to the complete user question. Use only the returned movie documents as factual context. State the best matching title and briefly explain why it matches. If no movie is relevant, say that no matching movie was found.",
          "tools": ["MOVIE_SEARCH_TOOL"],
          "enable_human_tool": "false"
        }'
      );
    END;
    /
    ```

3. Create a sequential team that assigns the task to the movie agent.

    ```sql
    BEGIN
      DBMS_CLOUD_AI_AGENT.CREATE_TEAM(
        team_name  => 'MOVIE_AGENT_TEAM',
        attributes => '{
          "agents": [
            {"name": "MOVIE_ANALYST", "task": "ANSWER_MOVIE_TASK"}
          ],
          "process": "sequential"
        }'
      );
    END;
    /
    ```

4. Confirm that the team is discoverable.

    ```sql
    SELECT DBMS_CLOUD_AI_AGENT.LIST_TEAMS() AS teams
    FROM dual;
    ```

## Task 4: Ask the movie agent

1. Set the agent team and ask a natural-language question with Select AI.

    The quoted prompt follows the Select AI form. The unqualified form is used for a profile-level Select AI prompt; adding the `AGENT` action routes this prompt through `MOVIE_AGENT_TEAM`, allowing the task to apply `MOVIE_SEARCH_TOOL` before the response is generated.

    ```sql
    EXEC DBMS_CLOUD_AI_AGENT.SET_TEAM('MOVIE_AGENT_TEAM');

    SELECT AI AGENT "Tell me about movies with superheroes saving others";
    ```

    The response is generated by the profile's generative model after the task calls `MOVIE_SEARCH_TOOL`. Your wording and response may vary, but verify that it names a movie from the retrieved documents and explains the match without inventing unsupported facts.

    A response might look like this:

    ```text
    The movie that best matches your request is Saving Private Ryan (1998). It follows a rescue mission during the invasion of Normandy.
    ```

## Task 5: See the movie agent in action

1. Ask the agent questions that require semantic retrieval rather than an exact title match.

    Run each query separately after setting the team in Task 4. Observe how the same `MOVIE_SEARCH_TOOL` handles different natural-language requests.

    ```sql
    SELECT AI AGENT "Which movie is about a rescue mission during an invasion?";
    ```

    ```sql
    SELECT AI AGENT "Which movie features a group of heroes working together against an alien threat?";
    ```

    The first prompt should retrieve movie context related to a rescue mission, and the second should retrieve context related to heroes and an alien threat. Exact rankings and wording depend on the E5-base model and the configured generative model.

2. Inspect the tool history to confirm that the agent applied the created retrieval tool.

    ```sql
    SELECT tool_name,
           agent_name,
           task_name,
           task_order,
           start_date,
           end_date,
           input,
           output
    FROM user_ai_agent_tool_history
    WHERE tool_name = 'MOVIE_SEARCH_TOOL'
    ORDER BY start_date DESC
    FETCH FIRST 5 ROWS ONLY;
    ```

    The `INPUT` column shows the natural-language value passed to `p_query`, and `OUTPUT` contains the movie JSON returned by the function. This is the point where the agent applies the custom tool over the duality view.

3. Inspect the team and task history for the same runs.

    ```sql
    SELECT team_exec_id,
           team_name,
           start_date,
           end_date,
           state
    FROM user_ai_agent_team_history
    WHERE team_name = 'MOVIE_AGENT_TEAM'
    ORDER BY start_date DESC
    FETCH FIRST 5 ROWS ONLY;
    ```

    ```sql
    SELECT team_exec_id,
           agent_name,
           task_name,
           task_order,
           state,
           result
    FROM user_ai_agent_task_history
    WHERE task_name = 'ANSWER_MOVIE_TASK'
    ORDER BY start_date DESC
    FETCH FIRST 5 ROWS ONLY;
    ```

    Together, these history views show the workflow: the team receives the question, the task calls `MOVIE_SEARCH_TOOL`, the function retrieves E5-base matches through `MOVIES_DV`, and the agent uses that context to produce the final response.

## Acknowledgements

* **Author** - Gael Palomino
* **Last Updated By/Date** - Gael Palomino, August 2026
