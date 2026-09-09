# Proposal: Vector Search with Duality Views

Estimated Time: 60–90 minutes

## Positioning

This is Part 2 of the movie-search workshop that begins with the MongoDB API. Part 1 teaches vector and text search from MongoDB Shell. Part 2 teaches the same search concepts with SQL, then shows how to make the JSON document and its semantic-search metadata evolve independently.

## Why this design matters

- Keep the original JSON movie document independent from embeddings and vector indexes.
- Add, rebuild, or drop a vector index without changing the application data.
- Add multiple vector columns for different embedding models over the same JSON document.
- Migrate to a new model by adding its column and index, moving search traffic, then retiring the old vector.
- Expose JSON data and its vectors as one document through a duality view while retaining indexed vector search.

## Audience and duration

Database developers who completed the MongoDB API workshop. Target duration: 60–90 minutes.

## Workshop sequence

1. **Get Ready** — Repeat the OCI, Autonomous AI Database, and ONNX Object Storage preparation from Part 1. Do not install or use MongoDB Shell.
2. **Lab 1: Create JSON and vector data, then run text search** — Create the table with `id`, JSON `data`, and an embedding vector; load movie documents; create the Oracle Text index; run a lexical search.
3. **Lab 2: Run semantic search with SQL** — Load multilingual E5-small, generate embeddings, create the HNSW index, run nearest-neighbor search, and introduce the model-migration pattern.
4. **Lab 3: Compare embedding models on the same movie data** — Add multilingual E5-base in a new vector column, create its independent index, search with it, and compare relevance with E5-small.
5. **Lab 4: Expose JSON and vectors with a duality view** — Create the duality view, query it as a document, run semantic search through it, and confirm vector-index access with an execution plan.
6. **Lab 5: Build a Movie Agent** — Configure Select AI, register a duality-view movie-search function as a tool, query the grounded agent, and inspect the tool and team history.

## Scope notes

- SQL is the only client interface in this workshop; MongoDB API setup, `mongosh`, and the `mongo.preview` property are outside its scope.
- The workshop starts with 384-dimensional multilingual E5-small and then adds 768-dimensional multilingual E5-base. Learners who choose another model must add a separate appropriately sized column and index.
- The Select AI lab remains experimental until the approved provider, model, credentials, privileges, and sandbox behavior are confirmed.

## Acknowledgements

* **Author** - Gael Palomino
* **Last Updated By/Date** - Gael Palomino, August 2026
