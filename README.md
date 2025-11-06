# GitHub Test Case Generator - Dual Mode (Batch & Individual)

This repository contains an n8n workflow `github-test-case-generator-dual-mode.json` that generates executable unit tests for repository source files using an LLM (via AWS Bedrock in the provided nodes). The workflow supports two modes: `batch` (group files into prompts) and `individual` (analyze each file separately).

## Purpose

- Automatically generate production-ready unit tests for source files in a GitHub repository.
- Provide a repeatable workflow that can run in bulk (batch) or per-file (individual).

## Configuration parameters (what they do & where used)

All of these live in the `Configuration` node in the workflow (node name: `Configuration`). The node sets initial variables used across the flow.

- `owner` (string)

  - Description: GitHub repository owner/org (e.g., `wizeline`).
  - Where used: GitHub API requests to form repository URLs and fetch content.
  - Default in example: `wizeline`.

- `repo` (string)

  - Description: Repository name (e.g., `BytescribeTeam`).
  - Where used: GitHub API requests to fetch tree and file contents.
  - Default in example: `BytescribeTeam`.

- `branch` (string)

  - Description: Source branch to read files from (e.g., `main`).
  - Where used: API calls to read repository tree and file content.
  - Default in example: `main`.

- `processingMode` (string) — `batch` or `individual`

  - Description: Choose processing mode.
    - `batch`: groups multiple files into a single prompt (fewer AI calls, larger prompts).
    - `individual`: sends each file in its own prompt (more AI calls, more precise results per file).
  - Where used: `Route by Processing Mode` node to pick the route.
  - Default in example: `batch`.

- `batchSize` (number)

  - Description: Number of files included in each batch when `processingMode` is `batch`.
  - Where used: `Create Batches` node slices `sourceFiles` into groups of `batchSize`.
  - Effect: Controls prompt size and number of AI calls.
  - Example default in example: `5`.
  - Recommendation: 5–10 as a safe starting point (depends on token limits and file sizes).

- `fetchFileContent` (boolean)

  - Description: When `true` the workflow will fetch raw file content from GitHub (used for detailed generation or individual mode).
  - Where used: `Fetch File Content` and `Prepare Individual File` nodes.
  - Default in example: `true`.

- `maxFilesToProcess` (number)

  - Description: Maximum number of matching source files the workflow will process overall.
  - Where used: `Process File Structure` node uses `.slice(0, maxFilesToProcess)` when building the `sourceFiles` list.
  - Effect: Caps work done (useful for large repos and cost control).
  - Example default in example: `10`.

- `testBranch` (string)
  - Description: Branch name where auto-generated tests will be committed (e.g., `auto-generated-tests`).
  - Where used: Branch creation and commit nodes (`Prepare Branch Creation Data`, `Create Test Branch`, commit HTTP request).
  - Default in example: `auto-generated-tests`.

## Where parameters appear in the workflow

Key nodes (by name) and roles:

- `Process File Structure` (code node)

  - Reads the repository tree response and builds `sourceFiles` list (applies `maxFilesToProcess`).

- `Create Batches` (code node)

  - Uses `batchSize` to slice `sourceFiles` into batches (each batch becomes one prompt to the AI in batch mode).

- `Prepare Batch Prompt` (set node)

  - Builds the text prompt for a batch including a list of files.

- `AI - Batch Analysis` / `AI - Individual File Analysis` (language model nodes)

  - Send prompts and receive AI outputs.

- `Format Final Output` (code node)

  - Parses AI responses, aggregates test file metadata and contents, and returns a consolidated JSON structure.

- `Export to JSON File` (convertToFile node)

  - Writes the resulting JSON of generated tests to disk.

- `Prepare Git Commit` / `Create/Update File on GitHub` (code + httpRequest nodes)
  - Create/commit test files to the `testBranch` branch.

## General flow (step-by-step)

1. Trigger: `When clicking 'Test workflow'` — start the flow.
2. `Configuration` sets runtime variables (owner, repo, branch, processingMode, batchSize, fetchFileContent, maxFilesToProcess, testBranch).
3. `Get Branch Info` and `Get Repository Tree` fetch repository metadata and full file tree.
4. `Process File Structure` filters potential source files (by extension and excluded folders), sets `sourceFiles` and `sourceFilesCount`, and applies `maxFilesToProcess`.
5. `Route by Processing Mode`:
   - If `individual`: split into single-file items and analyze each file separately (`Fetch File Content` -> `Prepare Individual File Prompt` -> `AI - Individual File Analysis`).
   - If `batch`: `Create Batches` groups files into batches of `batchSize` and runs `AI - Batch Analysis`.
6. `Merge All Results` collects AI outputs (from individual or batch paths).
7. `Format Final Output` parses and normalizes AI responses into a well-structured JSON with fields such as `repository`, `processingMode`, `summary`, and `testFiles` (each file includes `testFileContent`, `testFile`, `runCommand`, etc.).
   - If parsing issues are encountered, `errors` are returned in the summary (the workflow includes robust parsing code to handle template literals and common malformed responses).
8. `Export to JSON File` writes consolidated results locally.
9. (Optional) The flow then enriches each `testFile` with repo context, creates test files locally, and pushes them to the configured `testBranch` on GitHub.

## Recommended defaults and why

- `processingMode`: `batch` for quick overviews and fewer API calls; `individual` for precise per-file tests.
- `batchSize`: 5 (safe default). Larger sizes reduce the number of AI calls but increase prompt/token usage and parsing complexity. If files are small, you can push to 8–10; if files are large, reduce to 1–3.
- `maxFilesToProcess`: 50 (start conservatively). This limits cost and runtime. Increase as needed.
- `fetchFileContent`: `true` (if you want accurate tests). If you set `false`, batch prompts will rely only on path/heuristic inference.

Calculate expected AI calls: ceil(min(totalSourceFiles, maxFilesToProcess) / batchSize)

Example: 50 source files, `maxFilesToProcess = 50`, `batchSize = 5` → 10 batch prompts to the AI.

## Troubleshooting tips

- Parsing errors in `Format Final Output`:

  - Cause: AI returned JavaScript template literals (backticks) or Python triple-quoted strings inside JSON fields. The node includes normalization to convert those into valid JSON; if errors persist, inspect the `errors` array returned by the node for the `output` snippet and `parseAttempts`.
  - Fixes: Reduce `batchSize` (smaller prompts produce simpler outputs), or switch to `individual` mode for problematic files.

- Token limits / truncated outputs:

  - Symptom: AI response truncated or missing test content.
  - Fixes: Reduce `batchSize` or provide smaller `maxFilesToProcess`. Also consider lowering temperature in AI node options to reduce variance.

- Too many API calls / cost high:

  - Symptom: High number of AI calls when processing large repos.
  - Fixes: Increase `batchSize` (if files are small) and set a lower `maxFilesToProcess`.

- Generated tests failing to run:
  - Inspect `testFileContent` exported JSON, verify imports/mocks match your project setup, and adapt the prompt templates to include project-specific import paths or setup steps.

## Suggested small improvements (optional)

- Add runtime validation in the `Configuration` node or early code node to enforce:
  - `batchSize` >= 1 (integer)
  - `maxFilesToProcess` >= 1 (integer)
  - Reasonable upper bounds (e.g., `maxFilesToProcess <= 500`) to prevent runaway runs
- Add a summary/logging node after `Create Batches` to log the number of batches and estimated token size for each batch.
- Add unit tests for the `Format Final Output` parsing code (mock a few AI outputs including template literals, triple quotes, and malformed JSON) to ensure stability.

## How to run (n8n)

- Import `github-test-case-generator-dual-mode.json` into your n8n instance.
- Configure credentials for GitHub and AWS Bedrock (or your LM provider) in n8n.
- Update the `Configuration` node values as needed.
- Trigger the workflow manually or via a scheduled trigger.

## Where to edit prompts

- Batch prompt: node `Prepare Batch Prompt`
- Individual prompt: node `Prepare Individual File Prompt`
- If you want to add project-specific instructions (test conventions, import paths, common mocks), update these prompt templates.

## Contact & next steps

If you want, I can:

- Add validation code to the `Configuration` node to enforce sensible bounds.
- Add a small logging node that prints the number of batches and files per batch before the AI call.
- Add example unit tests and a small harness to validate `Format Final Output` parsing logic.

---

README created for `github-test-case-generator-dual-mode.json`. If you want this content changed, or a one-page printable summary instead, tell me which sections to tighten or expand.
