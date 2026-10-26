# Learn Application Monitoring

A practical path from your first application metric to useful dashboards and alerts—with **Prometheus, Grafana, Python, and FastAPI**.

[Read the course](https://rohit1024.github.io/learn-monitoring/) · [Browse the lessons](https://rohit1024.github.io/learn-monitoring/lessons/) · [Project goals](docs/mission.md)

## What you'll learn

Build a local monitoring stack, instrument a FastAPI application, collect its metrics, and investigate its behavior. The curriculum progresses from metric types and bounded labels to PromQL, Grafana dashboards, and a Golden Signals monitoring project.

Lessons combine explanations, diagrams, guided exercises, deliberate failures and recovery, and active-recall questions. Each authored lesson includes prerequisites, verification steps, and restart notes so you can resume without conversation history.

## Curriculum

| Module | Lessons | Focus |
| --- | --- | --- |
| Set up the stack | 0001–0002 | Docker Compose, persistence, scrape configuration, target health |
| Collect application metrics | 0003–0005 | Metric types, label cardinality, FastAPI instrumentation, scraping diagnostics |
| Investigate with PromQL | 0006–0010 | UI and HTTP API, selectors, rates, aggregation, latency histograms |
| Build Grafana dashboards | 0011–0014 | Data sources, panels, variables, transformations |
| Integrate monitoring and alerting | 0015–0016 | Golden Signals project, provisioning, alerting |

**Current status:** Lessons **0001–0006** are fully authored. Lessons **0007–0016** contain detailed outlines for future expansion. Authored content is separate from demonstrated learner completion.

Start with [lesson 0001](https://rohit1024.github.io/learn-monitoring/lessons/0001-setting-up-prometheus-and-grafana/) or use the [lesson index](docs/lessons/index.md) to pick up where you left off.

## Run the documentation locally

You need [uv](https://docs.astral.sh/uv/) and Python 3.14 or later. uv can install the required Python version. The monitoring exercises additionally require a container engine with Docker Compose support; individual lessons specify their prerequisites and version pins.

``` bash
git clone https://github.com/Rohit1024/learn-monitoring.git
cd learn-monitoring
uv sync --locked
uv run zensical serve
```

Open the local address printed by Zensical. To build the static site:

``` bash
uv run zensical build --clean
```

The generated site is written to `site/` and is not committed.

## Repository layout

``` text
docs/
  lessons/           Sequential lessons and curriculum index
  cheatsheet/        Quick-reference section
  debugging/         Diagnostic playbook section
  interview/         Interview practice section
  references/        Official sources and glossary
  mission.md         Learning goals and scope
  lesson-authoring.md Shared lab and authoring conventions
overrides/           Site icons and theme customizations
.agents/skills/teach/ Teaching workflow and diagram validator
learning-records/    Records of demonstrated understanding
zensical.toml        Documentation site configuration
```

The reference, debugging, cheatsheet, and interview sections grow with the course. Learner-created application files and evidence live in a local `monitoring-lab/` directory as instructed by the lessons.

## Improve or extend a lesson

Follow the [authoring guide](docs/lesson-authoring.md), preserve the shared metric schema, and verify technical claims against the [official references](docs/references/resources.md).

For agents supporting local skills, the included `teach` skill can guide a session or expand an outline:

``` text
$teach Expand lesson 0007 using the outline in docs/lessons/0007-promql-selectors-and-vectors.md and docs/lesson-authoring.md.
```

Before submitting changes, build the site and validate its Mermaid diagrams. Diagram validation requires Node.js/npm and uses the Mermaid CLI through `npx`.

``` bash
python3 .agents/skills/teach/scripts/validate_mermaid.py
uv run zensical build --clean
```

## Publishing

[GitHub Actions](https://github.com/Rohit1024/learn-monitoring/actions/workflows/docs.yml) builds the site from the locked dependencies and publishes it to **GitHub Pages** on pushes to `main`. The workflow can also be run manually.

**Course website:** https://rohit1024.github.io/learn-monitoring/
