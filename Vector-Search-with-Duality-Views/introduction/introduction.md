# Vector Search with Duality Views

## Introduction

You have already searched the movie sample through the MongoDB API. This Part 2 livelab runs those same search capabilities with SQL while keeping the original JSON document separate from its semantic-search metadata. Oracle AI Database lets you add, rebuild, or drop vector columns and their vector indexes without rewriting the application JSON.

Estimated Workshop Time: 60–90 minutes

### Prerequisites

- Autonomous AI Database 26ai with AI Vector Search enabled.
- SQL worksheet or SQLcl access.
- Either an ONNX embedding model already available to the database, or access to an Oracle Cloud Infrastructure Object Storage bucket where you can upload the provided models.

### Objectives

- Keep JSON movie data independent from its embedding columns.
- Run text and semantic searches with SQL.
- Use a duality view to expose the JSON data and vectors as one document.
- Explore an experimental Select AI agent grounded in movie search.

## About this workshop

Search answers a simple question: which movie best matches a user's request? Keyword search looks for the same words in the movie data. It works well when the user knows the title or uses terms that appear in the document. It can miss related results when the user describes an idea with different words.

Semantic search compares meaning instead of exact words. An embedding model converts a movie summary into a vector, which is a numeric representation of the meaning in that text. The same model converts the user's request into another vector. Oracle AI Database compares the two vectors with a distance metric and returns the closest matches. The vector column stores the embeddings, and a vector index helps the database find those matches efficiently.

Embedding models do not represent language in exactly the same way. Models can differ in training data, language coverage, dimensions, and how well they capture a particular type of question. You can keep the original movie JSON in `DATA` and add one vector column and index for each model. This workshop starts with E5-small, then adds E5-base as the migration target. You can compare the results on the same movies and migrate search gradually without re-ingesting or rewriting the source documents.

A JSON-relational duality view provides the application-facing layer for this design. It maps selected relational columns into a JSON document, so an application can read the movie fields and its search metadata through one document shape. The database still owns the underlying JSON and vector columns, and the view materializes the document when you query it; it does not require a second copy of the data. You will use the view to return movie documents, run semantic search, and confirm that the database can still use the underlying vector index.

By the end of the workshop, you will have a search design with two complementary retrieval methods, independent embedding models over shared movie data, and a JSON document interface over the relational storage and search structures.

## Learn More

- [Oracle AI Vector Search User's Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/index.html)
- [JSON-Relational Duality Developer's Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/jsnvu/index.html)
- [Generate Embeddings](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/generate-embeddings.html)

## Acknowledgements

* **Author** - Gael Palomino
* **Last Updated By/Date** - Gael Palomino, August 2026
