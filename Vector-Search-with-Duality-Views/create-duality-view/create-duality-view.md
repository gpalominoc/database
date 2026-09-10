# Lab 4: Expose JSON and Vectors with a Duality View

## Introduction

Imagine a movie-discovery application that accepts a natural-language request such as “a team of heroes working together to protect others.” The application wants complete movie documents in JSON, while the database team wants to keep the source data relational, maintain embeddings in a native `VECTOR` column, and use an HNSW index to answer the semantic search efficiently. Maintaining separate JSON and search copies would create synchronization work and make model or index changes harder to manage.

Create a JSON-relational duality view to bridge those needs. The view exposes the movie JSON and the migrated E5-base vector as one document, while the underlying `MOVIES` table remains the system of record. The application can consume the document shape, and SQL can still search the underlying vector column and `SUMMARY_BASE_VEC_IDX`. This pattern is useful for catalog search, recommendations, and retrieval applications that need document-style responses without giving up relational integrity, indexing, or a gradual embedding-model migration.

Estimated Time: 30 minutes

### Objectives

- Create a duality view over the movie JSON and vector metadata.
- Query the duality view as JSON.
- Run semantic search through the duality view.
- Use an execution plan to verify the vector index access path.
- Combine semantic ranking with JSON filters and keyword search.

## Task 1: Create the movie duality view

1. Create a JSON-relational duality view that maps the movie identifier, JSON fields, and `embedding_e5_base` vector into a single document.

    ```sql
    CREATE OR REPLACE JSON RELATIONAL DUALITY VIEW movies_dv AS
    SELECT JSON {
        '_id': id,
        'movie': data,
        'embedding': embedding_e5_base
    }
    FROM movies
    WITH UPDATE INSERT DELETE;
    ```

2. Query the view and inspect one movie document.

    ```sql
    SELECT JSON_SERIALIZE(data returning CLOB PRETTY) AS movie_document
    FROM movies_dv
    FETCH FIRST 1 ROW ONLY;
    ```

    The returned document contains the `_id`, the original movie JSON under `movie`, and the E5-base vector under `embedding`. The duality view does not copy or re-ingest the source JSON.

## Task 2: Run semantic search through the duality view

1. Use the duality view in a semantic SQL query and return the top five matching movie documents.

    The duality view is the application-facing document surface, but the vector index is defined on the relational `MOVIES.EMBEDDING_E5_BASE` column. Rank the movie rows from `MOVIES` first, then join the five selected identifiers to `MOVIES_DV` to return document-shaped results. This keeps the vector search in the query block that can use `SUMMARY_BASE_VEC_IDX` while the application still receives documents from the duality view.

    ```sql
    WITH nearest_movies AS (
      SELECT id
      FROM movies
      ORDER BY VECTOR_DISTANCE(
                 embedding_e5_base,
                 VECTOR_EMBEDDING(
                   MULTILINGUAL_E5_BASE
                   USING 'a team of heroes working together to protect others' AS data
                 ),
                 COSINE
               )
      FETCH FIRST 5 ROWS ONLY
      WITH TARGET ACCURACY 90
    )
    SELECT JSON_SERIALIZE(d.data PRETTY) AS movie_document
    FROM nearest_movies n
    JOIN movies_dv d
      ON JSON_VALUE(d.data, '$._id' RETURNING NUMBER) = n.id;
    ```

    The result can be consumed as one JSON document while the nearest-neighbor step uses the relational vector column and its HNSW index. Compare these documents with the E5-small results from Lab 2.

## Task 3: Inspect the execution plan

1. Ask the optimizer to describe the expected execution path for the indexed relational search followed by the duality-view lookup.

    `EXPLAIN PLAN FOR` does not execute the query. It records the estimated operations so you can verify that the `NEAREST_MOVIES` query block uses the underlying E5-base vector column and selects `SUMMARY_BASE_VEC_IDX` before the result is joined to the duality view.

2. Generate the execution plan for the same query.

    ```sql
    EXPLAIN PLAN FOR
    WITH nearest_movies AS (
      SELECT id
      FROM movies
      ORDER BY VECTOR_DISTANCE(
                 embedding_e5_base,
                 VECTOR_EMBEDDING(
                   MULTILINGUAL_E5_BASE
                   USING 'a team of heroes working together to protect others' AS data
                 ),
                 COSINE
               )
      FETCH FIRST 5 ROWS ONLY
      WITH TARGET ACCURACY 90
    )
    SELECT JSON_SERIALIZE(d.data PRETTY) AS movie_document
    FROM nearest_movies n
    JOIN movies_dv d
      ON JSON_VALUE(d.data, '$._id' RETURNING NUMBER) = n.id;
    ```

3. Display the execution plan.

    ```sql
    SELECT *
    FROM TABLE(DBMS_XPLAN.DISPLAY);
    ```

    Look for `VECTOR INDEX HNSW SCAN` with the index name `SUMMARY_BASE_VEC_IDX` in the `NEAREST_MOVIES` query block. This confirms that the semantic search uses the relational vector index before the matching documents are read from the duality view. The exact plan can vary with table size, statistics, and optimizer settings, so verify the operation and index name in the plan output.

## Task 4: Combine semantic search with filters

1. Add metadata filters to the semantic search. This query finds movies about a team of heroes, while restricting the results to Action movies released in 2000 or later.

    Apply the structured filters to the source rows before the indexed vector search, then use the matching identifiers to return fields from the duality-view document. The vector expression references the relational `EMBEDDING_E5_BASE` column directly, so this query can use `SUMMARY_BASE_VEC_IDX`.

    ```sql
    WITH nearest_movies AS (
      SELECT m.id
      FROM movies m
      WHERE JSON_VALUE(m.data, '$.year' RETURNING NUMBER) >= 2000
        AND JSON_EXISTS(m.data, '$.genre[*]?(@ == "Action")')
      ORDER BY VECTOR_DISTANCE(
                 m.embedding_e5_base,
                 VECTOR_EMBEDDING(
                   MULTILINGUAL_E5_BASE
                   USING 'a team of heroes working together to protect others' AS data
                 ),
                 COSINE
               )
      FETCH FIRST 5 ROWS ONLY
      WITH TARGET ACCURACY 90
    )
    SELECT JSON_VALUE(d.data, '$.movie.title'
                      RETURNING VARCHAR2(200)) AS title,
           JSON_VALUE(d.data, '$.movie.year'
                      RETURNING NUMBER) AS year,
           JSON_QUERY(d.data, '$.movie.genre'
                      RETURNING VARCHAR2(1000)) AS genre
    FROM nearest_movies n
    JOIN movies_dv d
      ON JSON_VALUE(d.data, '$._id' RETURNING NUMBER) = n.id;
    ```

## Task 5: Combine text and vector results with INTERSECT

1. Return movies that appear in both the title search for `avengers` and the 20 nearest semantic matches.

    `JSON_TEXTCONTAINS` uses the `TITLE_IDX` JSON search index on `MOVIES.DATA`. The vector branch ranks the relational `EMBEDDING_E5_BASE` column so it can use `SUMMARY_BASE_VEC_IDX`, then joins the matching identifiers to `MOVIES_DV` for the document fields.

    ```sql
    -- Text results INTERSECT vector results
    WITH text_results AS (
      SELECT JSON_VALUE(data, '$.title' RETURNING VARCHAR2(200)) AS title,
             JSON_VALUE(data, '$.year' RETURNING NUMBER) AS year,
             JSON_QUERY(data, '$.genre' RETURNING VARCHAR2(1000)) AS genre
      FROM movies
      WHERE JSON_TEXTCONTAINS(data, '$.title', 'avengers', 1)
    ),
    vector_results AS (
      SELECT JSON_VALUE(d.data, '$.movie.title' RETURNING VARCHAR2(200)) AS title,
             JSON_VALUE(d.data, '$.movie.year' RETURNING NUMBER) AS year,
             JSON_QUERY(d.data, '$.movie.genre' RETURNING VARCHAR2(1000)) AS genre
      FROM (
        SELECT id
        FROM movies
        ORDER BY VECTOR_DISTANCE(
                   embedding_e5_base,
                   VECTOR_EMBEDDING(
                     MULTILINGUAL_E5_BASE
                     USING 'a team of heroes working together to protect others' AS data
                   ),
                   COSINE
                 )
        FETCH FIRST 20 ROWS ONLY
        WITH TARGET ACCURACY 90
      ) n
      JOIN movies_dv d
        ON JSON_VALUE(d.data, '$._id' RETURNING NUMBER) = n.id
    )
    SELECT title, year, genre
    FROM text_results
    INTERSECT
    SELECT title, year, genre
    FROM vector_results
    ORDER BY year, title
    FETCH FIRST 3 ROWS ONLY;
    ```

    `INTERSECT` returns distinct title, year, and genre values found by both searches. The vector branch uses `VECTOR_DISTANCE` only to rank its 20 nearest candidates; the distance is not returned. The final query sorts the shared results by year and title and returns at most three rows.

## Acknowledgements

* **Author** - Gael Palomino
* **Last Updated By/Date** - Gael Palomino, August 2026
