# Vector Search with Duality Views

## Short Description

Continue the MongoDB API movie-search workshop with SQL, independent embedding columns, JSON-relational duality views, and an experimental Select AI agent.

## Long Description

This Part 2 workshop starts with the same Autonomous AI Database and ONNX model preparation used in the MongoDB API workshop, without installing or using MongoDB Shell. It stores the source movie document as JSON while keeping embedding columns and their vector indexes independent from that JSON data.

Learners build a text index and an HNSW vector index, compare text and semantic searches in SQL, and then migrate from multilingual E5-small to E5-base by adding a second embedding column and index. They use a JSON-relational duality view to expose each movie and its searchable vectors as one document, then finish with an experimental Select AI agent grounded in the duality view.

## Workshop Outline

1. Introduction
2. Get Ready - Prepare the database and ONNX model
3. Lab 1 - Create JSON and vector tables, then run text search
4. Lab 2 - Run semantic search with SQL
5. Lab 3 - Compare embedding models on the same JSON data
6. Lab 4 - Expose JSON and vectors with a duality view
7. Lab 5 - Build a Movie Agent

## Workshop Prerequisites

- Autonomous AI Database 26ai with AI Vector Search enabled
- SQL worksheet or SQLcl access
- Permission to create the sample objects, Oracle Text and vector indexes, and JSON-relational duality view
- Either an ONNX embedding model already available to the database, or access to an Oracle Cloud Infrastructure Object Storage bucket where you can upload the provided models

Estimated Time: 60–90 minutes

## Acknowledgements

* **Author** - Gael Palomino
* **Last Updated By/Date** - Gael Palomino, August 2026

## Notes

- Keep this file aligned with the final manifest and lab titles.
- Keep the long description learner-focused. Do not say the workshop was created from a blog, prompt, or source format.
