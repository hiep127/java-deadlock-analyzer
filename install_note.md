# Installing JCA as a GitHub Copilot Skill in VS Code

JCA (Java Concurrency Analyzer) detects concurrency defects — deadlocks, race conditions, lock inversions, and thread-safety violations — in **any Java codebase**. Point it at any directory containing `.java` files.

> The `aosp-framework-base/` submodule included in this repo is a reference source (AOSP Audio Framework) you can use to try the tool. You are not required to use it.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| VS Code | Latest stable |
| GitHub Copilot extension | `GitHub.copilot` |
| GitHub Copilot Chat extension | `GitHub.copilot-chat` |
| Copilot plan | Individual, Business, or Enterprise (agent mode required) |
| Git | For cloning |

---

## Step 1 — Clone this repository

```bash
git clone <this-repo-url> Java-deadlock-analyzer
cd Java-deadlock-analyzer
```

If you want to use the bundled AOSP reference source, also run:

```bash
git submodule update --init --recursive
```

Otherwise skip it — JCA works against any Java source you point it at.

---

## Step 2 — Open the folder in VS Code

Open `Java-deadlock-analyzer/` as your workspace root:

```
File → Open Folder → select Java-deadlock-analyzer/
```

GitHub Copilot automatically reads `.github/agents/*.agent.md` and `.github/skills/` from the workspace root. No manual registration needed.

---

## Step 3 — Enable Copilot Agent Mode

1. Open VS Code Settings (`Ctrl+,`)
2. Search for `github.copilot.chat.agent.enabled`
3. Set it to **true**

Or add this to your `settings.json`:

```json
{
  "github.copilot.chat.agent.enabled": true
}
```

---

## Step 4 — Verify the skill is loaded

1. Open Copilot Chat (`Ctrl+Alt+I`)
2. Switch to **Agent mode** using the mode selector at the top of the chat panel
3. Type `/jca-help` and press Enter

You should see the JCA usage guide. If you get an unknown command error, confirm the `.github/` folder is present at the workspace root and reload VS Code (`Ctrl+Shift+P` → `Developer: Reload Window`).

---

## Step 5 — Run an analysis

Pass any relative path to a Java source directory:

```
/jca-analyze src/main/java/com/example/service/
```

```
/jca-analyze lib/core/src/java/
```

Using the bundled AOSP reference source as a demo:

```
/jca-analyze aosp-framework-base/services/core/java/com/android/server/audio/
```

Results are written to `concurrency_analysis/` in the workspace root:

| File | What it contains |
|---|---|
| `concurrency_analysis/report.md` | Full findings with File:Line references |
| `concurrency_analysis/report.json` | Machine-readable findings |
| `concurrency_analysis/lock-dependency.dot` | Lock dependency graph (open with a Graphviz extension) |
| `concurrency_analysis/pipeline.log` | Agent execution trace |

---

## Available Commands

| Command | What it does |
|---|---|
| `/jca-analyze <path>` | Run the full 7-phase analysis pipeline on any Java path |
| `/jca-publish` | Publish the latest report to Confluence |
| `/jca-help` | Show usage and command reference |

---

## Configuration

Edit `jca-config.json` to adjust pipeline behavior:

```json
{
  "pipeline": {
    "parallelWorkers": 4,
    "maxPartitionSizeKB": 512
  },
  "analysis": {
    "minSeverityToReport": "LOW"
  }
}
```

Set `minSeverityToReport` to `"HIGH"` to reduce noise on a first run.

---

## Troubleshooting

**Slash command not recognized**
- Confirm agent mode is enabled (Step 3)
- Confirm `.github/agents/` and `.github/skills/` exist at the workspace root
- Reload VS Code (`Ctrl+Shift+P` → `Developer: Reload Window`)

**No findings / empty report**
- Make sure the path you passed contains `.java` files
- Check `concurrency_analysis/pipeline.log` for which agent failed

**Submodule folder is empty**
```bash
git submodule update --init --recursive
```
