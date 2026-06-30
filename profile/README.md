<p align="center">
  <img src="apptim-header.webp" alt="Apptim" />
</p>

**Catch performance and UX regressions in your mobile app before users feel them.**

Apptim helps developers, QA engineers, and product teams find performance and UX problems in their Android and iOS apps. Instead of a single session average, it pinpoints exactly *when* and *where* the experience degrades, and tracks whether each release gets better or worse.

## Why "Apptim"

Apptim is the two-faced Roman god of doorways, beginnings, and transitions, who looks backward and forward at once. That is the product: every run is compared against the previous version across the same flow transitions, seeing the old build and the new build at the same time. January, the month of "what changed, what's next," is named after him.

## What Apptim does

- **Device metrics**: CPU, memory, FPS, energy, network and more, shown as timeline graphs with aggregations.
- **Perceived UX Score**: AI video analysis of a session that produces a Lighthouse-style score, comparable release over release.
- **Intelligent Reports**: a written, senior-engineer-style narrative generated from the run's metrics.

Supporting AI actions round it out: **Intelligent Fix Suggestions** (why a failed Script run failed), the **AI-Driven** runner (executes a natural-language goal), and the **Live Assistant** (real-time voice and chat Q&A about a test).

## Why Apptim is different

Per-flow-segment metrics, not session averages:

- ❌ "Average session CPU was 60%."
- ✅ "CPU spiked to 94% during the checkout transition, lasted 2.1s, and did not happen in the previous version."

Scores are **relative to the project's own history**: a 63 means "11 points below your recent best," not "bad in absolute terms." A change counts as a regression only when it clears the project's measured noise band and holds across consecutive runs. Scores anchor to deterministic device counters wherever possible, so they stay reproducible.

## Who it's for

- **Developers**: no visibility into UX degradation between versions.
- **QA teams**: manual performance testing is inconsistent and does not scale.
- **Product teams**: cannot quantify the UX impact of a feature.

## How it's built

Two **solutions** sit on three shared **agents**:

- **Solutions** are how you run Apptim: **Desktop** (human-in-the-loop capture, on every plan) and the **Apptim CLI** (the ENTERPRISE CI/CD pipeline, with an interactive terminal UI for local runs).
- **Agents** are the engine underneath: **Profiler** (capture), **Intelligence** (the AI actions), and **Studio** (the bridge to the cloud).

## Repositories

| Repo | What it handles |
| ---- | --------------- |
| [**domain**](https://github.com/janusperf/domain) | Shared domain models, metric catalog, and threshold definitions used across agents and Studio API. |
| [**agent**](https://github.com/janusperf/agent) | Multi-agent engine on the test machine (Profiler, Intelligence, Studio): capture, AI actions, and cloud bridge. |
| [**studio**](https://github.com/janusperf/studio) | The cloud and Studio Web: runs, Perceived UX Scores, Intelligent Reports, workspaces, billing, and alerts. |
| [**desktop**](https://github.com/janusperf/desktop) | Desktop solution for Manual, Script, and AI-Driven tests on connected Android and iOS devices. |
| [**cli**](https://github.com/janusperf/cli) | Apptim CLI: React Ink TUI for local runs and headless CI/CD pipeline integration. |
| [**docs**](https://github.com/janusperf/docs) | Product documentation site: Desktop, CLI, pipeline integration, Studio, metrics reference, and plans. |
| [**landing**](https://github.com/janusperf/landing) | Public marketing site. |

## Running it

To bring up the whole stack locally or in production, see **[RUNNING.md](RUNNING.md)**.
