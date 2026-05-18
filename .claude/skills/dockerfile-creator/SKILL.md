---
name: dockerfile-creator
description: >
  Generates a production-ready Dockerfile and .dockerignore for the current project
  following current best practices fetched via context7. Automatically detects the
  project stack from repo files (package.json → Node.js, requirements.txt → Python,
  go.mod → Go, pom.xml/build.gradle → Java, Cargo.toml → Rust, etc.).

  Use this skill whenever the user wants to create, generate, or write a Dockerfile,
  containerize their application, or asks how to dockerize a project. Also trigger
  when the user mentions "docker build", "Docker image", "container", "containerize",
  "preciso de um Dockerfile", "criar Dockerfile", "gerar Dockerfile", or similar —
  even if they don't use the exact word "Dockerfile".
---

# Dockerfile Creator

Generates a `Dockerfile` and `.dockerignore` for the current project, grounded in
current best practices fetched via context7. The two files are always created
together — a Dockerfile without a `.dockerignore` is incomplete.

## Step 1 — Detect the source folder

Check whether the project has a dedicated source subfolder (e.g., `src/`, `app/`,
`backend/`) that contains the stack's entry-point indicators. If the indicators are
in the repo root, the root is the source folder.

Indicators by stack:
- Node.js: `package.json`
- Python: `requirements.txt`, `pyproject.toml`, `setup.py`
- Go: `go.mod`
- Java/Kotlin: `pom.xml`, `build.gradle`
- Rust: `Cargo.toml`
- Ruby: `Gemfile`
- PHP: `composer.json`
- .NET: `*.csproj`, `*.sln`

**If multiple folders each contain indicators (monorepo), ask the user which one
to use before proceeding.**

## Step 2 — Detect the stack

Derive the stack from the indicator files found. Read the main indicator file
(e.g., `package.json`) to extract:
- Runtime version if specified (e.g., `engines.node`, Python version from
  `.python-version` or `pyproject.toml`)
- Entry point / start command
- Build step if any (e.g., `scripts.build` in `package.json`)

**If no indicator is found, ask the user for the stack before continuing.**

## Step 3 — Fetch current best practices via context7

Use context7 to get up-to-date Dockerfile guidance. Query format:

```
dockerfile {stack} best practices
```

Examples:
- `dockerfile nodejs best practices`
- `dockerfile python best practices`
- `dockerfile go best practices`

Extract from the results:
- Recommended base image and tag (prefer slim/alpine variants for interpreted
  stacks)
- Correct layer ordering for cache efficiency (dependencies before source code)
- Non-root user setup specific to the image
- Any stack-specific gotchas (e.g., `npm ci` vs `npm install`, virtual envs
  for Python)

## Step 4 — Choose Dockerfile strategy

**Default: single-stage Dockerfile.**

Use multi-stage **only** for stacks where compilation produces a self-contained
binary and the build toolchain would significantly bloat the final image:
- Go
- Rust
- .NET (publish → runtime image)
- Java (build with Maven/Gradle → JRE runtime image)

For Node.js, Python, Ruby, PHP — single-stage. These runtimes are needed at
runtime too, so multi-stage adds complexity without meaningful size reduction.

## Step 5 — Generate the files (show draft first, don't save yet)

### Dockerfile rules (always apply)

1. **Non-root user** — always. Create a dedicated user and switch to it before
   the final `CMD`/`ENTRYPOINT`. Don't run as root.
2. **Dependency layer before source copy** — copy only the dependency manifest
   first (`package.json`, `requirements.txt`, `go.mod`+`go.sum`, etc.), install,
   then copy the rest of the source. This keeps the dependency layer cached when
   only source changes.
3. **Explicit tag on base image** — never use `latest`. Pin to a specific minor
   version (e.g., `node:20-slim`, `python:3.12-slim`).
4. **WORKDIR** — always set an explicit workdir (e.g., `/app`).
5. **CMD vs ENTRYPOINT** — use `CMD` for the default start command unless the
   image is meant to be used as an executable, in which case use `ENTRYPOINT`.

### .dockerignore rules

Always include:
```
.git
.gitignore
node_modules/        # if Node.js
__pycache__/         # if Python
*.pyc                # if Python
.env
.env.*
*.log
README.md
docs/
.DS_Store
```

Add stack-specific entries based on what the project has.

### Draft presentation

Show both files in full in the chat with a brief explanation of the key decisions
made (base image choice, user setup, cache layering). Do **not** write any files
yet.

End with: "Quer que eu salve assim em `{source_folder}/Dockerfile` e
`{source_folder}/.dockerignore`, ou prefere ajustar algo antes?"

## Step 6 — Iterate if needed

If the user requests changes, apply them in the chat draft and show again.
Don't save until explicit approval.

## Step 7 — Save on approval

When the user approves (e.g., "pode salvar", "tá bom", "salva", "yes", "go
ahead"), write:
- `{source_folder}/Dockerfile`
- `{source_folder}/.dockerignore`

Confirm the paths to the user after saving.
