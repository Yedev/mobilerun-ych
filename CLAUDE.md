# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Install dependencies** (uses `uv`):
```bash
uv sync
uv sync --extra anthropic    # include Anthropic SDK
uv sync --extra langfuse     # include Langfuse tracing
uv sync --extra dev          # dev tools (black, ruff, mypy, bandit)
```

**Run tests:**
```bash
uv run python -m pytest tests/
uv run python -m pytest tests/test_fast_agent_xml_parser.py  # single test file
uv run python -m pytest tests/test_fast_agent_xml_parser.py::FastAgentXmlParserTest::test_drops_adjacent_exact_duplicate_tool_calls  # single test
```

**Lint and format:**
```bash
uv run black .                          # format
uv run black --check --diff .           # check only (CI mode)
uv run ruff check mobilerun/            # lint
uv run ruff check --fix mobilerun/      # lint + autofix
```

**Run the CLI:**
```bash
uv run mobilerun --help
uv run mobilerun configure              # set up LLM provider
uv run mobilerun run "open Chrome"      # run command on device
```

## Architecture

### Multi-Agent Orchestration

`MobileAgent` (`mobilerun/agent/droid/droid_agent.py`) is the top-level `llama-index` `Workflow` that orchestrates everything. It operates in two modes:

- **Reasoning mode** (`reasoning=True`, default): `ManagerAgent` generates a plan and current subgoal → `ExecutorAgent` executes one action → loops back to Manager until complete or `max_steps` reached.
- **Direct mode** (`reasoning=False`): `FastAgent` generates Python-like XML tool calls and executes them directly without a planning step.

The workflow uses typed `Event` objects (defined in `mobilerun/agent/droid/events.py`) to pass data between `@step`-decorated methods. The step graph is: `start_handler` → `run_manager` ↔ `run_executor` → `finalize` (reasoning mode), or `start_handler` → `execute_task` → `finalize` (direct mode).

### Driver / StateProvider Split

Device interaction is split into two concerns:

- **`DeviceDriver`** (`mobilerun/tools/driver/base.py`): Raw I/O — tap, swipe, type, screenshot, app launch. Concrete implementations: `AndroidDriver` (ADB), `IOSDriver` (HTTP to Portal), `VisualRemoteDriver`, `StealthDriver` (wrapper), `RecordingDriver` (wrapper that logs actions for macro replay).
- **`StateProvider`** (`mobilerun/tools/ui/provider.py`): Reads device UI state and returns a filtered, formatted `UIState`. `AndroidStateProvider` uses the accessibility tree; `IOSStateProvider` uses iOS Portal; `ScreenshotOnlyStateProvider` skips tree parsing (vision-only mode).

`start_handler` constructs the driver and state provider, then wires them into an `ActionContext` that is passed to all sub-agents.

### Tool Registry

`ToolRegistry` (`mobilerun/agent/tool_registry.py`) is the single source of truth for tools available to agents. `build_tool_registry` (`mobilerun/agent/utils/signatures.py`) registers all standard tools (click, swipe, type, etc.) at startup. User custom tools and MCP tools are added on top. The registry filters out tools unsupported by the current driver+provider combination via a `supported` capability set.

### ActionContext

`ActionContext` (`mobilerun/agent/action_context.py`) is the dependency bag passed to every action function — it holds the driver, current UI state, shared state, state provider, credential manager, and LLM for app-opening. Action functions in `mobilerun/agent/utils/actions.py` receive this instead of individual tool instances.

### MobileAgentState (Shared State)

`MobileAgentState` (`mobilerun/agent/droid/state.py`) is the mutable coordination object shared across all agents within a single run. It tracks: step number, action/summary/outcome history, error flags, visited packages/activities, remembered facts (`fast_memory`), and queued external user messages (for mid-run injection via `send_user_message`).

### Configuration

`MobileConfig` and its nested dataclasses live in `mobilerun/config_manager/config_manager.py`. The config is loaded from a YAML file by `ConfigManager` (`mobilerun/config_manager/config_manager.py`). `LLMProfile` entries in `llm_profiles` are resolved to actual `llama-index` LLM instances by `load_agent_llms` (`mobilerun/agent/utils/llm_loader.py`). Each agent role (`manager`, `executor`, `fast_agent`, `app_opener`, `structured_output`) can have its own LLM profile.

### Prompts

All agent prompts are Jinja2 templates under `mobilerun/config/prompts/` (subdirectories per agent: `executor/`, `fast_agent/`, `manager/`). `PromptResolver` (`mobilerun/agent/utils/prompt_resolver.py`) loads them, with user-supplied custom prompts taking priority over defaults.

### App Cards

App cards (`mobilerun/config/app_cards/` + `mobilerun/app_cards/`) are JSON files that provide app-specific instructions injected into agent prompts (e.g., how to navigate a particular app). The `AppCardProvider` looks up cards by package name.

### Telemetry & Tracing

- **Telemetry**: `posthog`-based usage events fired via `capture()`/`flush()` in `mobilerun/telemetry/`.
- **Tracing**: OpenTelemetry traces via Arize Phoenix (default) or Langfuse (optional extra). Setup happens in `setup_tracing` (`mobilerun/agent/utils/tracing_setup.py`).

### Compat Layer

`compat/droidrun/` provides backward-compatible imports for the legacy `droidrun` package name. `DroidAgent = MobileAgent` alias is maintained in `droid_agent.py` until v0.8.0.

## Key Conventions

- **Python 3.11–3.13** required (3.14 not supported).
- **Package manager**: `uv`. Do not use `pip` directly.
- **Formatter**: `black` (line length 100 via `ruff`, but black uses its own defaults). CI enforces black formatting.
- **Linter**: `ruff` with E/W/F/I/B rules; E501 (line length) is ignored.
- **Async throughout**: All device I/O, agent steps, and tool calls are `async`. Use `asyncio.run()` or `await` at the top level.
- **Drivers declare capabilities**: Never check `isinstance(driver, AndroidDriver)`. Use `driver.supported` and `state_provider.supported` sets to determine available capabilities.
- **Documentation**: `docs/v5/` contains MDX docs. `SKILL.md` maps topics to doc files for quick reference.
