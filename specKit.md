# Spec Kit

A step-by-step guide to installing and initialising [Spec Kit](https://github.com/github/spec-kit) — a CLI for spec-driven development with optional AI agent integrations.

---

## Prerequisites

Spec Kit is distributed via [uv](https://github.com/astral-sh/uv), a fast Python package manager.

### 1. Install uv

```bash
brew install uv
```

Verify the installation:

```bash
uv --version
```

---

## Step 1 — Install Spec Kit CLI

```bash
uv tool install specify-cli
```

> This installs the `specify` command globally via uv's tool layer.

---

## Step 2 — Initialise a Project

### Basic init (no agent integration)

```bash
specify init my-project
cd my-project
```

### Init with an agent integration

Pass `--integration` to wire up an AI agent at init time:

```bash
# GitHub Copilot
specify init my-project --integration copilot

# Cline
specify init my-project --integration cline

cd my-project
```

> The `--integration` flag generates the agent-specific configuration files alongside the standard spec structure.

---

## Step 3 — Install an Agent Integration (if not done at init)

If you initialised without `--integration`, or if the integration did not apply correctly, run the following from inside your project directory:

```bash
cd my-project
specify integration install copilot
```

### Verify the integration files were created

List hidden files to confirm the agent configuration was written:

```bash
ls -a
```

You should see integration-specific files such as `.github/` (Copilot) or `.clinerules` (Cline) alongside the standard spec files.

---

## Step 4 — Init into an Existing Directory

To initialise Spec Kit inside a directory that already exists (without creating a new subdirectory), use `.` as the target:

```bash
cd my-project2
specify init .
```

Then install the agent integration:

```bash
specify integration install copilot
```

Confirm all files were written:

```bash
ls -a
```

---

## Next Step — Create the Constitution

With the project initialised and the agent integration in place, the next step is to create the project **constitution** — a foundational specification document that defines the project's goals, constraints, and conventions.

> See the constitution authoring guide (coming next).

---

## Quick Reference

| Command | Purpose |
| --- | --- |
| `brew install uv` | Install the uv package manager |
| `uv --version` | Verify uv installation |
| `uv tool install specify-cli` | Install the Spec Kit CLI |
| `specify init <name>` | Create a new project directory |
| `specify init <name> --integration copilot` | Init with GitHub Copilot integration |
| `specify init <name> --integration cline` | Init with Cline integration |
| `specify init .` | Init inside the current directory |
| `specify integration install copilot` | Add Copilot integration post-init |
| `ls -a` | Verify hidden integration files are present |
