# pal-skills

Custom [pi](https://github.com/earendil-works/pi) skills for everyday dev tasks.

## Skills

| Skill | Trigger phrases | What it does |
|-------|----------------|--------------|
| **beautiful-tui** | "make it beautiful", "add a TUI", "terminal dashboard" | Scaffolds a pretty terminal UI using [ink](https://github.com/vadimdemedes/ink) + [termcn](https://github.com/termcn/termcn) — spinners, progress bars, tables, themes |
| **build-deploy** | "build image", "push to registry", "deploy to K8s" | Full pipeline: Dockerfile → Docker build → registry push → K8s secret upsert → `kubectl apply` |
| **build-package** | "build the package", "build installer", "package the app" | PyInstaller EXE → Inno Setup installer → smoke-test, produces a distributable Windows `.exe` |
| **concept-explainer** | any named technical concept (e.g. "what is CQRS", "explain attention mechanism") | Single-file HTML visual explainer with interactive diagrams, purposeful typography, zero AI-slop |
| **momentum-finder** | "find momentum stocks", "find dip stocks", "stocks like TICKER" | Scans for momentum stocks in a dip with 10-20%+ 1-month upside; supports pattern-matching against a reference ticker |

## Usage

Skills are picked up automatically by pi from this directory. They activate on matching trigger phrases — no explicit invocation needed.

To use a skill explicitly, reference it by name in your prompt, e.g.:

```
use build-deploy to push my app to staging
```

## Adding a skill

Drop a new `skills/<name>/SKILL.md` following the [pi skills spec](https://github.com/earendil-works/pi/blob/main/docs/skills.md). Run `graphify update` after adding.
