# Semantic Search Validation Guide

This document explains how to exercise the two semantic PR branches locally before
handing them to reviewers. It covers the classic text-index branch
(`feature/semantic-text-index`) and the embeddings branch (`feature/semantic-embeddings`).

The steps assume you can run Kodi from source and issue JSON-RPC requests via `curl`.

---

## 1. Environment Preparation

1. **Build Kodi** (debug is preferred so you can watch logging):

   ```bash
   cmake -B build -S . -DCMAKE_BUILD_TYPE=Debug
   cmake --build build -j$(sysctl -n hw.ncpu)
   ```

2. **Create a sample video library** with at least one movie and one TV episode so the
   SemanticIndexService has data to ingest. Make sure the files include either subtitles
   or metadata (plot, tags, etc.).

3. **Enable JSON-RPC** for localhost inside Kodi if it is not already enabled:

   ```
   Settings → Services → Control → Allow remote control via HTTP (localhost)
   ```

---

## 2. Validating `feature/semantic-text-index`

### 2.1 Enable Semantic Search UI Settings

```
Settings → System → Services → Semantic Search
  - Enable Semantic Search           : ON
  - Processing Mode                  : Idle (or Background)
  - Auto-transcribe content          : OFF (initially)
  - Index subtitles / metadata       : ON
```

Setting the processing mode to *Idle* or *Background* will automatically queue
all unindexed media thanks to the new seeding logic.

### 2.2 Watch the Service Start

Start Kodi from the build folder:

```bash
./build/kodi-x11 2>&1 | tee semantic.log
```

Look for logs similar to:

```
SemanticIndexService: Started successfully (mode: idle)
SemanticIndexService: Queueing all unindexed media...
SemanticIndexService: Found X unindexed items
```

If you switch the setting back to *Manual*, the service will stay running but will not
process queued items until you manually call `Semantic.QueueIndex`.

### 2.3 Exercise JSON-RPC

Use `curl` to hit the new endpoints:

```bash
curl -s -X POST http://localhost:8080/jsonrpc \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"Semantic.GetIndexState",
       "params":{"media_id":1,"media_type":"movie"}}' | jq
```

Useful commands to try while indexing is running:

* `Semantic.GetStats`
* `Semantic.QueueIndex` (force a specific media ID)
* `Semantic.QueueTranscription`
* `Semantic.Search` (`params: {"query":"robot uprising"}`)

Make sure the commands succeed whether or not Kodi’s GUI is open, because the service
is now completely wired through `ServiceManager`.

### 2.4 Validate Auto-Seeding

1. Remove an item from the library, then rescan.
2. Watch the logs for:

   ```
   SemanticIndexService: Removing chunks for deleted movie 42
   SemanticIndexService: Queueing all unindexed media...
   ```

3. Query `Semantic.GetIndexState` for the re-added item and confirm a new row exists
   without using JSON-RPC to seed it manually.

### 2.5 Run the Local Test Script

```bash
./xbmc/semantic/test-local.sh
```

`cppcheck` will now run automatically if installed. The script reports remaining TODOs
and provides code statistics so you can copy/paste them into review notes.

---

## 3. Validating `feature/semantic-embeddings`

This branch includes the text-index fixes above *plus* optional embedding/vector code.

### 3.1 Optional Dependencies

#### sqlite-vec
1. Download the amalgamation files from
   https://github.com/asg017/sqlite-vec/releases/latest:
   * `sqlite-vec.c`
   * `sqlite-vec.h`
2. Place them in `lib/sqlite-vec/`.

If the files are missing, Kodi will still build but vector search is disabled and you’ll
see a log warning at startup. With the files present, you’ll get `HAVE_SQLITE_VEC=1` and
semantic vectors will be stored/queried through sqlite-vec.

#### ONNX Runtime
1. Install ONNX Runtime 1.16+ (system package or binary download).
2. Make sure `cmake` can find it (`ONNXRuntime_DIR` or pkg-config). When found,
   the semantic embedding module defines `HAS_ONNXRUNTIME=1` and links to it.
3. Without ONNX Runtime, the embedding engine builds as a stub and returns default
   responses (useful for developers without the SDK installed).

### 3.2 Multilingual Settings

A new group appears under *Semantic Search → Multilingual*:

* Enable Multilingual Search
* Default Multilingual Model (defaults to `paraphrase-multilingual-MiniLM-L12-v2`)

Enable the switch, provide a model name, and restart Kodi. The multilingual engine will
download the model (if missing), initialize it via the embedding engine, and log
the active model. Because downloads can be large, verify that the model paths under
`special://home/semantic/models/multilingual/` contain both `model.onnx` and `vocab.txt`.

### 3.3 Hybrid Search Smoke Test

1. Verify the `Semantic.FindSimilar` JSON-RPC call now returns a stub error until actual
   embeddings are generated.
2. Once ONNX Runtime and sqlite-vec are configured, trigger a transcription/embedding job
   (future work may add an explicit RPC; for now inspect logs for embedding status).
3. Use GUI search or `Semantic.Search` with the `options` field to exercise both the BM25
   text path and the vector similarity path. Watch for logging such as:

   ```
   HybridSearchEngine: Query embedding generated in XX ms
   HybridSearchEngine: Combined RRF results (keyword + vector)
   ```

### 3.4 Tests

Run the same `./xbmc/semantic/test-local.sh` script; it is branch-aware and will include
embedding/vector sources when present. For deeper validation, run Kodi’s GoogleTest suite
if available (`cmake --build build --target semantic_test` followed by the gtest binary).

---

## 4. Translating New Settings

The new settings use IDs 39530–39534. Add human-readable strings to each language file,
e.g., `addons/resource.language.en_gb/resources/strings.po`.

Example POT entries:

```
msgctxt "#39530"
msgid "Multilingual"
msgstr ""

msgctxt "#39531"
msgid "Enable multilingual search"
msgstr ""

msgctxt "#39532"
msgid "Use cross-language embeddings to search media in any language."
msgstr ""

msgctxt "#39533"
msgid "Default multilingual model"
msgstr ""

msgctxt "#39534"
msgid "Model identifier (e.g., paraphrase-multilingual-MiniLM-L12-v2)."
msgstr ""
```

---

## 5. Preparing Pull Requests

1. Push each branch (`feature/semantic-text-index` and `feature/semantic-embeddings`)
   to your fork.
2. Update PR descriptions with:
   * Summary of behavior changes
   * Testing performed (section 2 & 3 above)
   * Outstanding TODOs or optional dependencies (sqlite-vec/ONNX runtime)
3. Link to this validation document so reviewers know how to reproduce your results.
4. Monitor CI: focus on verifying that Linux/macOS builds without optional dependencies,
   and that Windows builders see the expected “vector disabled” logging unless they
   provide sqlite-vec manually.

---

## 6. Troubleshooting Checklist

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| `Semantic.*` RPC returns `FailedToExecute` immediately | Service not enabled or hasn’t seeded the index | Enable Semantic Search and wait for queue seeding; watch logs |
| Index state never transitions past `pending` | Media files lack subtitles/metadata or parsing failed | Check Kodi log for parsing errors; confirm file paths |
| Vector search logs “sqlite-vec support not available” | sqlite-vec sources missing | Download and place `sqlite-vec.c/.h` under `lib/sqlite-vec/`, then rebuild |
| Embedding engine logs “ONNX Runtime support not available” | ONNXRuntime not found during configure | Install ONNX Runtime and re-run CMake |
| Multilingual switch does nothing | Model files missing or download failed | Inspect `special://home/semantic/models/multilingual/`; re-run with network access |

---

Following this checklist before submitting ensures the reviewers can simply pull the PR,
follow the same steps, and verify behavior without additional back-and-forth. Update
this document whenever new semantic features land so it stays in sync with reality.
