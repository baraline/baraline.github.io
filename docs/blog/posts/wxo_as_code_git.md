---
date: 2026-02-09
authors:
  - baraline
categories:
  - Python
  - IBM watsonx
  - AI Agents
  - CI/CD
---


# Git-Driven Agent Management with IBM watsonx Orchestrate ADK

Managing AI agents through a web UI works fine for quick experiments.
It falls apart the moment multiple people need to edit the same agent, roll back a broken prompt, or pusing configurations for multiples agents from staging to production without having to manually do it for every agent.

This post walks through a simple solution: **storing every agent, tool and knowledge-base definition in a Git repository** and using the watsonx Orchestrate Agent Development Kit (ADK) CLI to synchronize them with your Orchestrate instance. 

[**Update : As a few poeple asked for it, I've setup a GitHub template repository that include evertyhing discussed**](https://github.com/baraline/wxo_git_template)

Feel free to raise issues or PR on this template if things are missing.

---

## Why put Orchestrate artefacts in Git?

| Pain point (UI-only) | Git-based solution |
|---|---|
| "Who changed the prompt last Friday?" | `git log --oneline agents/` |
| Simultaneous edits overwrite each other | Branches + pull-request reviews |
| Rolling back means manual re-typing | `git revert` + re-import |
| Promoting mutliple agents to production needs mutliple of manual steps | Release tag triggers automated deploy for all agents |

In short, you get **versioning, collaboration, code review, automated testing for your tools and deterministic deployments** — the same things every other piece of software gets from Git.

---

## Repository structure

We structure the repository as follow :

```
.
├── .github/workflows/ # define the CI/CD actions
├── agents/
│   ├── My_Agent_abc123/
│   │   └── agents/
│   │       └── native/
│   │           └── My_Agent_abc123.yaml    # agent specification
│   └── Another_Agent_xyz789/
│       └── agents/
│           └── native/
│               └── Another_Agent_xyz789.yaml
├── tools/
│   ├── extract_markdown_from_file/
│   │   ├── extract_markdown_from_file.py   # @tool-decorated function
│   │   ├── requirements.txt                # tool-specific dependencies
│   │   ├── __init__.py
│   │   └── tests/
│   │       ├── __init__.py
│   │       └── test_extract_markdown_tool.py
│   └── produce_word_file_from_string/
│       ├── produce_word_file_from_string.py
│       ├── requirements.txt
│       ├── __init__.py
│       └── tests/
│           └── test_produce_word_file_from_string.py
├── scripts/
│   ├── import_agents_from_orchestrate.py   # pull agents from WXO → repo
│   ├── export_agents_to_orchestrate.py     # push agents from repo → WXO
│   ├── import_tools_from_orchestrate.py    # pull tools from WXO → repo
│   └── export_tools_to_orchestrate.py      # push tools from repo → WXO
├── pyproject.toml
└── .gitignore
```

We'll only focus on what "Native" agents for now, which are agents defined in watsonx Orchestrate.

### Key conventions

* **`agents/<agent_name>/agents/native/<agent_name>.yaml`** — mirrors the folder layout that `orchestrate agents export` produces, so import/export scripts are zero-config.
* **`tools/<tool_name>/`** — one directory per tool. The Python file has the same name as the directory. Each tool ships its own `requirements.txt` to be executed on the orches
* **`scripts/`** — automation helpers that wrap the ADK CLI.

---

## Setting up the ADK

### Prerequisites

* Python 3.12 (Only version supported for exporting Python tools)
* A watsonx Orchestrate account (You can setup a [free 30-day trial](https://www.ibm.com/account/reg/us-en/signup?formid=urx-52753) if you don't have an account)

### Create a python environment

You can use your favorite python environment manager (conda/uv/...) for this. Just make sure to install the `ibm-watsonx-orchestrate` package, as it exposes the `orchestrate` CLI inside the virtual environment.

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install --upgrade ibm-watsonx-orchestrate
```

### Register your environments

Most setups need at least two environments — one for testing (draft) and one for production (live):

```bash
# Test / staging instance
orchestrate env add \
  -n wxo_test \
  -u https://api.<region>.watson-orchestrate.cloud.ibm.com/instances/<test-instance-id> \
  --type ibm_iam

# Production instance
orchestrate env add \
  -n wxo_prod \
  -u https://api.<region>.watson-orchestrate.cloud.ibm.com/instances/<prod-instance-id> \
  --type ibm_iam
```

In the following we'll assume your are using a IBM Cloud instance. For AWS or Cloud Park, some commands might differ (for exemple here AWS-hosted instances use `--type mcsp` and on-premises Cloud Park for Data use `--type cpd`).

### Activate and authenticate

```bash
orchestrate env activate wxo_test
# You will be prompted for your IBM IAM API key
```

For non-interactive use (CI pipelines), pass the key directly:

```bash
orchestrate env activate wxo_test --api-key "$WXO_API_KEY"
```

In long-running pipelines, be sure to re-activate before each batch of commands to avoid token expiration.

### Storing secrets safely

| Secret | Where to store | **Never do** |
|---|---|---|
| IBM IAM API key | CI/CD secret variables (`WXO_API_KEY`) | **Hard-code in scripts** |
| `.env` with LLM keys (for auto-discover) | Local machine only, listed in `.gitignore` | **Push to remote** |

The ADK stores its own session state in two files:

* `~/.config/orchestrate/config.yaml` — environment list
* `~/.cache/orchestrate/credentials.yaml` — cached JWT

**Both are local to the machine and should never be shared or committed.**

---

## Import / export scripts

The four scripts in `scripts/` wrap the ADK CLI so you can synchronize your repository with Orchestrate in a single command. They all support a `--env` flag to target a specific environment (test or production) and `--deploy` to push agents from draft into the live state after import.

### Pulling agents from Orchestrate into Git

```bash
python scripts/import_agents_from_orchestrate.py --env wxo_test --verbose
```

Under the hood, the script:
1. Calls `orchestrate agents list --kind native -v` to discover agents.
2. Exports each agent's YAML with `orchestrate agents export -n <name> -k native -o <path> --agent-only`.
3. Enriches the YAML with `llm_config` metadata if present in the API response.
4. Saves the file into `agents/<name>/agents/native/<name>.yaml`.

These script basicaly are wrapers around the orchestaret CLI, for exemple `import_agents_from_orchestrate.py` defines:

```python
def export_and_extract_agent(agent_name: str, project_root: Path,
                             max_retries: int = 3, agent_data: dict = None):
    agents_dir = project_root / "agents" / agent_name / "agents" / "native"
    agents_dir.mkdir(parents=True, exist_ok=True)
    output_path = agents_dir / f"{agent_name}.yaml"

    command = [
        "orchestrate", "agents", "export",
        "-n", agent_name,
        "-k", "native",
        "-o", str(output_path),
        "--agent-only",
    ]

    for attempt in range(1, max_retries + 1):
        try:
            result = subprocess.run(command, capture_output=True,
                                    text=True, timeout=300)
            if result.returncode == 0:
                logger.info("Exported %s → %s", agent_name, output_path)
                break
        except subprocess.TimeoutExpired:
            if attempt < max_retries:
                time.sleep(5 * attempt)  # exponential backoff
                continue
            raise
```

We can run these script manualy when needed, but we'll mainly use them in the CI/CD pipeline.

### Pushing agents from Git to Orchestrate

```bash
# Import to draft (test environment)
python scripts/export_agents_to_orchestrate.py --env wxo_test

# Import AND deploy to live (production)
python scripts/export_agents_to_orchestrate.py --env wxo_prod --deploy
```

The script iterates over every `*.yaml` file in the `agents/` tree and calls `orchestrate agents import -f <path>`. When `--deploy` is passed, it additionally runs `orchestrate agents deploy --name <agent_name>` for each successfully imported agent.

The core logic is similar to the previous one, use subprocess to run orchestrate CLI commands:

```python
def import_agent_file(agent_file: Path, deploy: bool = False) -> bool:
    result = subprocess.run(
        ["orchestrate", "agents", "import", "-f", str(agent_file)],
        capture_output=True, text=True, check=False, timeout=120,
    )
    if result.returncode != 0:
        logger.error("Import failed for %s: %s", agent_file.name, result.stderr)
        return False

    if deploy:
        agent_name = agent_file.stem
        dep_result = subprocess.run(
            ["orchestrate", "agents", "deploy", "--name", agent_name],
            capture_output=True, text=True, check=False, timeout=120,
        )
        if dep_result.returncode != 0:
            logger.error("Deploy failed for %s: %s", agent_name, dep_result.stderr)
            return False
        logger.info("Deployed %s to live", agent_name)

    return True
```

### Pushing tools from Git to Orchestrate

```bash
python scripts/export_tools_to_orchestrate.py --env wxo_prod
```

For each folder in `tools/`, the script runs:

```bash
orchestrate tools import -k python \
  -f tools/<tool_name>/<tool_name>.py \
  -p tools/<tool_name> \
  -r tools/<tool_name>/requirements.txt
```

The `-p` flag tells the ADK to include the entire directory as a package — essential when your tool spans multiple files.

### Targeting test vs. live environments

Every script accepts an `--env` flag. Before executing any `orchestrate` command, the script activates the target environment:

```python
def activate_environment(env_name: str, api_key: str | None = None):
    """Activate a WXO environment before running CLI commands."""
    cmd = ["orchestrate", "env", "activate", env_name]
    if api_key:
        cmd.extend(["--api-key", api_key])
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=60)
    if result.returncode != 0:
        raise RuntimeError(f"Failed to activate env '{env_name}': {result.stderr}")
    logger.info("Activated environment: %s", env_name)
```

This keeps the choice of environment explicit and auditable. In CI, the API key comes from a secret variable:

```bash
python scripts/export_agents_to_orchestrate.py \
  --env wxo_prod \
  --api-key "$WXO_API_KEY" \
  --deploy
```

---

## Agent YAML: what goes in Git

A typical agent specification looks like this:

```yaml
spec_version: v1
kind: native
name: My_Support_Agent
display_name: Support Agent
description: |
  Answers customer questions using the internal knowledge base.
  Routes complex issues to the escalation collaborator.
llm: watsonx/meta-llama/llama-3-2-90b-vision-instruct
style: default
hide_reasoning: false
instructions: |
  You are a support agent. Use the search_kb tool to find relevant
  documentation before answering. Always cite your sources.
  If the question is outside your scope, delegate to Escalation_Agent.
tools:
  - search_kb
collaborators:
  - Escalation_Agent
knowledge_base:
  - support_docs_kb
guidelines:
  - condition: "The user asks to speak to a human."
    action: "Acknowledge the request and transfer to Escalation_Agent."
    tool: ""
context_access_enabled: true
context_variables:
  - wxo_email_id
  - wxo_user_name
restrictions: editable
```

Every field is declarative and diff-friendly. Prompt changes show up clearly in pull-request reviews.

---

## Custom dependencies in tools

Each Python tool runs in an **isolated container with its own uv virtual environment**. You declare dependencies in a `requirements.txt` sitting next to the tool file.

### Single-file tool with dependencies

```
tools/
  extract_markdown_from_file/
    extract_markdown_from_file.py
    requirements.txt          # docling==2.62.0
```

```python
# extract_markdown_from_file.py
from ibm_watsonx_orchestrate.agent_builder.tools import tool
from docling.document_converter import DocumentConverter

_CONVERTER = DocumentConverter()

@tool()
def extract_markdown_from_file(file_url: str) -> str:
    """Extract Markdown content from a document at the given URL.

    Args:
        file_url (str): HTTP(S) URL pointing to the document.

    Returns:
        str: Markdown representation of the document.
    """
    result = _CONVERTER.convert(file_url)
    return result.document.export_to_markdown()
```

Import command with dependencies:

```bash
orchestrate tools import -k python \
  -f tools/extract_markdown_from_file/extract_markdown_from_file.py \
  -r tools/extract_markdown_from_file/requirements.txt
```

### Multi-file tool packages

When your tool spans multiple files (helper modules, configs, etc.), use the `-p` flag to upload the whole directory:

```bash
orchestrate tools import -k python \
  -f tools/my_tool/my_tool.py \
  -p tools/my_tool \
  -r tools/my_tool/requirements.txt
```

The ADK packages everything under `-p` into the tool's container.

### Pinning versions

Always pin exact versions in `requirements.txt`. Orchestrate tools run on **Python 3.12 only** (as of 2026). Keep your local development environment on the same version to catch compatibility issues early:

```
# tools/word_from_string/requirements.txt
spire-doc==13.8.0
```

### Local packages and private dependencies

The `requirements.txt` follows the standard [pip requirements file format](https://pip.pypa.io/en/stable/reference/requirements-file-format/), so you might be tempted to use local paths (`./libs/my_lib`) or private Git URLs. In practice, **tools run inside an isolated container** — local filesystem paths on your machine are meaningless at runtime, and the container has no access to your SSH keys or Git tokens.

Here are the two main strategies:

**1. Add the dependency into the tool package**

This is the simplest (but ugly) approach for **SMALL packages that you only need for one tool**. Place the dependency source code inside the tool directory and use the `-p` flag to upload the entire package:

```
tools/
  my_tool/
    my_tool.py
    requirements.txt
    libs/
      my_shared_lib/
        __init__.py
        helpers.py
```

In your tool code, use relative imports:

```python
from .libs.my_shared_lib.helpers import my_function
```

Import with the package flag:

```bash
orchestrate tools import -k python \
  -f tools/my_tool/my_tool.py \
  -p tools/my_tool \
  -r tools/my_tool/requirements.txt
```

This works for any code you control — internal libraries, forks of open-source packages, or generated code. The max compressed package size is **50 MB**.

**2. Use a public Git URL in `requirements.txt`**

If the dependency lives in a **public** Git repository, you can reference it directly:

```
# requirements.txt
requests==2.32.4
git+https://github.com/your-org/your-public-lib.git@v1.2.0
```

This works because the container has outbound internet access (on SaaS and Developer Edition). Pin to a specific tag or commit hash to keep builds reproducible.

**What does NOT work:**

| Approach | Why it fails |
|---|---|
| `./libs/my_lib` in requirements.txt | Path doesn't exist inside the container |
| `git+ssh://git@github.com/private-repo` | No SSH key available in the runtime |
| `git+https://<token>@github.com/private-repo` | Token would be committed to Git — security risk |

For **private Git** dependencies, I still need to work on a reliable plan (option 1 is ugly). If you must keep the dependency in a separate repository, consider publishing it to a private PyPI registry that the container can reach over HTTPS, or build a wheel and include it in the package directory.

For more details on all dependency features, see the official [Authoring Python Tools — Adding Dependencies](https://developer.watson-orchestrate.ibm.com/tools/create_tool) documentation.

---

## Other tool types

This post focuses on **Python tools** because they offer the most flexibility and fit naturally into a Git-based workflow. However, watsonx Orchestrate supports other tool types that may be a better fit depending on your use case.

### OpenAPI tools

If you already have a REST API with an **OpenAPI 3.0 specification**, you can expose its endpoints directly as Orchestrate tools — no Python code required. The ADK imports the spec and creates one tool per `operationId`:

```bash
orchestrate tools import -k openapi -f my-api-spec.yaml
```

Keep in mind that OpenAPI specs designed for developers often lack the detailed descriptions that agents need to use tools effectively. You might want to enrich your spec with agent-friendly descriptions.

One good option for hosting the OpenAPI backend is to [use IBM Code Engine functions](https://cloud.ibm.com/docs/codeengine?topic=codeengine-fun-work), this allows automatic scalling and deployments can also be automated by using the IBM Cloud CLI.

### Langflow tools

For teams using [Langflow](https://www.langflow.org/) to build visual workflows, watsonx Orchestrate can import Langflow flows as tools. This lets you orchestrate complex chains or RAG workflows built visually and expose them to your agents.

**The main interest is that Orchestrate will manage the runtime, so you don't need to host the langflow MCP to run the tools yourself**. Note that, depending on your specifics (e.g. using custom langflow component defined from a privately hosted python packages), this might result in slower runtimes compared to a dedicated MCP server. Choose based on your budget and constraints.

Langflow tools are supported on Developer Edition, IBM Cloud, and AWS, but can only be imported via the ADK CLI. Note that flows using the Langflow Agent component or invoking external agents are not yet supported. See [Langflow tools — Prerequisites and limitations](https://developer.watson-orchestrate.ibm.com/langflow/prerequisistes_and_limitations) for the current constraints.

---

## Local development workflow

## Code quality with pre-commit hooks

To catch formatting and linting issues before they reach CI, set up pre-commit hooks that run ruff automatically on every commit:

```bash
# Install pre-commit (if not already in your dev dependencies)
pip install pre-commit

# Install git hooks
pre-commit install
```

The repository includes a `.pre-commit-config.yaml` that runs ruff on all staged Python files:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.9.2
    hooks:
      # Run the linter
      - id: ruff
        args: [--fix]
      # Run the formatter
      - id: ruff-format
```

Now every commit automatically:
- Fixes auto-correctable linting issues
- Formats code to match project style
- Blocks the commit if unfixable issues remain

To manually run ruff on all files:

```bash
# Check and fix linting issues
ruff check . --fix

# Format all Python files
ruff format .

# Run all pre-commit hooks without committing
pre-commit run --all-files
```

Ruff configuration lives in `pyproject.toml`, where you can adjust rules, line length, and per-file ignores to match your team's standards.

---

## CI/CD automation

The real payoff comes when Git events trigger automated workflows. Below is a GitHub Actions setup, but the pattern ports to GitLab CI, Azure DevOps, or Jenkins without changes to the scripts.

*While we don't discuss it here, it is also a good practice to setup a code quality actions with pre-commit and stuff like ruff, isort, etc.*

### Run tool tests on every PR and push

The goal is to catch broken tools *before* they reach any Orchestrate environment. Note that this suppose that tools `requirements.txt` don't create dependencies issues or overwrite versions, if that's the case, each test run should be independent. Here is a **simple baseline** for such action, calling pytests to execute the tests defined by each tool.

```yaml
# .github/workflows/test-tools.yml
name: Test Tools

on:
  push:
    branches: [main, develop]
    paths: ["tools/**", "pyproject.toml"]
  pull_request:
    paths: ["tools/**", "pyproject.toml"]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.12"]

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install base dependencies
        run: |
          python -m pip install --upgrade pip
          pip install pytest ibm-watsonx-orchestrate
          # Install each tool's dependencies
          for req in tools/*/requirements.txt; do
            pip install -r "$req"
          done

      - name: Run tests
        run: pytest tools/ -v --tb=short
```

### Deploy agents to draft on push to `main`

Next, we want to deploy agents on the draft environment after merging a PR, the action initialise the orchestrate CLI using github secrets to load the API key. It then execute the script to export the agent and tools. The `WXO_TEST_URL` env var should point to your draft Orchestrate instance.

```yaml
# .github/workflows/deploy-draft.yml
name: Deploy Agents (Draft)

on:
  push:
    branches: [main]
    paths: ["agents/**", "tools/**"]

jobs:
  deploy-draft:
    runs-on: ubuntu-latest
    environment: wxo-test

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install ADK
        run: pip install --upgrade ibm-watsonx-orchestrate

      - name: Deploy tools to test
        env:
          WXO_API_KEY: ${{ secrets.WXO_TEST_API_KEY }}
        run: |
          orchestrate env add -n wxo_test \
            -u "${{ vars.WXO_TEST_URL }}" --type ibm_iam
          orchestrate env activate wxo_test --api-key "$WXO_API_KEY"
          python scripts/export_tools_to_orchestrate.py

      - name: Deploy agents to test (draft)
        env:
          WXO_API_KEY: ${{ secrets.WXO_TEST_API_KEY }}
        run: |
          python scripts/export_agents_to_orchestrate.py
```

### Promote to live on release

Finally, we want to deploy agents on the live environment after a release is published, the action initialise the orchestrate CLI using github secrets to load the API key. It then execute the same script as the draft action to export the agent and tools, but this time using `WXO_PROD_URL` env var, which should point to your live Orchestrate instance.

```yaml
# .github/workflows/deploy-live.yml
name: Deploy Agents (Live)

on:
  release:
    types: [published]

jobs:
  deploy-live:
    runs-on: ubuntu-latest
    environment: wxo-production

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install ADK
        run: pip install --upgrade ibm-watsonx-orchestrate

      - name: Deploy tools to production
        env:
          WXO_API_KEY: ${{ secrets.WXO_PROD_API_KEY }}
        run: |
          orchestrate env add -n wxo_prod \
            -u "${{ vars.WXO_PROD_URL }}" --type ibm_iam
          orchestrate env activate wxo_prod --api-key "$WXO_API_KEY"
          python scripts/export_tools_to_orchestrate.py

      - name: Deploy agents to production (live)
        env:
          WXO_API_KEY: ${{ secrets.WXO_PROD_API_KEY }}
        run: |
          python scripts/export_agents_to_orchestrate.py --deploy
```

## Limitations of ADK-imported Python tools

While the ADK and Orchestrate are evolving rapidly, there is still some limitations (as of ADK v2.1). Be sure to check if what you want to build don't fall into these :

### Read-only filesystem

Tools execute in a **read-only container**. You cannot write files to disk during execution. If your tool needs to produce a file (e.g., generate a Word document), return the content as `bytes` or a base64 string rather than a file path.

### Python version

The only supported runtime is **Python 3.12**. There is no way to select another version at import time. If your pinned dependency doesn't support 3.12, it won't work.

### Host networking

| Deployment | Localhost means… | Can access your intranet? |
|---|---|---|
| Developer Edition (Docker) | The container, not your host. Use `docker.host.internal`. | Yes (via Docker network) |
| SaaS (IBM Cloud / AWS) | The Orchestrate pod. | **No** — external internet only |
| On-premises | The execution node. Firewall rules must allow outbound. | Yes, within your network |

If your tool needs to call an internal API, prototype with the Developer Edition and plan for on-premises deployment or an API gateway.

### Dependency isolation

Each tool gets its own virtual environment — great for isolation, but it means large dependencies are re-installed per tool. Keep `requirements.txt` lean when possible.

### Deprecation policy

When a Python version is deprecated by Orchestrate, you get a 24-month window to re-import the tool. After that, the platform will attempt automatic migration to the nearest supported version, which may break things. Re-import proactively after any deprecation notice.

---

## Wrapping up

Treating your watsonx Orchestrate agents and tools as code gives you the same safety nets software teams rely on everywhere else: version control, peer review, automated testing, and push-button deployments. The setup cost is minimal — a folder structure, a handful of CLI wrappers, and a CI pipeline. The return is a workflow where changes are traceable, reversible, and testable before they ever touch production.

At some point, I'll probably try to benchmark the different options for tools, between a dedicated langflow MCP, Orchestrate tools (or langflow imports!), or a function based OpenAPI in code engine. This could help pinpoint the exact price to performance winner between these options, as it is currently unclear for me.
