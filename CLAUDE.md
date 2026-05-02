# molecule-ai-workspace-template-autogen

A Molecule AI workspace template that runs the **Microsoft AutoGen** (`autogen-agentchat`) framework as a workspace runtime. The template ships an `Adapter` class that the platform's `molecule-runtime` ENTRYPOINT discovers and loads.

This is **not** a plugin — there is no `plugin.yaml` and no `rules/` directory. It is a runtime container image, published per `RUNTIME_VERSION` to the Molecule registry by the reusable `Molecule-AI/molecule-ci` workflow.

---

## Files

| File | Role |
|---|---|
| `adapter.py` | The single source of runtime behavior. Defines `AutoGenAdapter` (subclass of `BaseAdapter`) and `AutoGenA2AExecutor` (subclass of `a2a.server.agent_execution.AgentExecutor`). |
| `config.yaml` | Workspace metadata: `name`, `runtime: autogen`, default `model`, model picker (`models:`), required env (`OPENAI_API_KEY`), `template_schema_version: 1`. |
| `system-prompt.md` | Default system prompt — used only when the workspace config does not override it. |
| `__init__.py` | Re-exports `AutoGenAdapter` as `Adapter`. The runtime resolves it via `ENV ADAPTER_MODULE=adapter` (set in the Dockerfile). |
| `Dockerfile` | `python:3.11-slim`, runs as user `agent` (uid 1000), entrypoint is `molecule-runtime`. Honors a `RUNTIME_VERSION` build-arg that pins the wheel installed on top of `requirements.txt` — that ARG is load-bearing because it busts the pip-install cache layer for cascade-triggered builds (see Dockerfile comment, 2026-04-27 incident). |
| `requirements.txt` | `molecule-ai-workspace-runtime>=0.1.0`, `autogen-agentchat>=0.4.0`, `autogen-ext[openai]>=0.4.0`. |
| `.github/workflows/ci.yml` | Delegates to `Molecule-AI/molecule-ci/.github/workflows/validate-workspace-template.yml@main`. |
| `.molecule-ci/scripts/validate-workspace-template.py` | Local copy of the validator the CI runs. |

---

## BaseAdapter integration (adapter.py)

`AutoGenAdapter(BaseAdapter)` implements four hooks the platform expects:

- `name()` → `"autogen"` — must match `runtime:` in `config.yaml`.
- `display_name()`, `description()`, `get_config_schema()` — surfaced in the canvas template picker / config editor.
- `setup(config)` — async; imports `AssistantAgent` to fail fast if `autogen-agentchat` is missing, then calls inherited `self._common_setup(config)` (provided by `BaseAdapter`) which returns the resolved system prompt and the platform's LangChain tool list. The tools are wrapped via `_langchain_to_autogen` and stashed on `self.autogen_tools`.
- `create_executor(config)` → returns `AutoGenA2AExecutor` (the A2A protocol object the molecule-runtime serves).

`_common_setup`, `build_task_text`, `brief_task`, `extract_history`, `extract_message_text`, `set_current_task` come from `molecule_runtime.adapters.shared_runtime` — the shared layer that owns delegation, memory, sandbox, and approval tools across all adapter templates. Do not reimplement these.

---

## AutoGen specifics

- The runtime uses **only** `AssistantAgent` (single-agent shape). There is no `UserProxyAgent` and no `GroupChat` / `Swarm` wiring in this template.
- A fresh `AssistantAgent` + `OpenAIChatCompletionClient` is constructed **per `execute()` call** inside `AutoGenA2AExecutor.execute` — there is no long-lived agent instance held on `self`. The system prompt and the tool list are reused, but the chat client and agent are not pooled.
- Model resolution: the config string (e.g. `openai:gpt-4.1-mini`) is split on `":"` and only the suffix is passed to `OpenAIChatCompletionClient(model=...)`. The provider prefix (`openai:`) is stripped silently — non-OpenAI prefixes will compile but fail at request time.
- Reply extraction: `agent.run()` returns a result with `messages`. The executor walks `messages` in reverse and returns the first item whose `content` is a `str`. If none qualifies, it falls back to `str(result)`.

---

## Tool wrapping (LangChain → AutoGen)

`_langchain_to_autogen(lc_tool)` wraps each platform LangChain `BaseTool` as an `autogen_core.tools.FunctionTool` with a single typed parameter `input: str`. The wrapper:

1. Tries `json.loads(input)` and, if the result is a `dict`, calls `await lc_tool.ainvoke(parsed_dict)` — preserves structured-input tools.
2. Otherwise falls back to `await lc_tool.ainvoke(input)`.

This is the bridge AutoGen requires because **AutoGen tools must have typed signatures (no `**kwargs`)** while LangChain tools accept opaque string-or-dict input. Keep the bridge function-shape — replacing it with a multi-arg signature breaks every platform tool that ships structured args (`delegate_task`, `commit_memory`, etc.).

---

## Async boundaries

- `setup`, `create_executor`, `execute`, and `cancel` are all `async def` — AutoGen's whole API is async-first; do not call any of them from synchronous code.
- `OpenAIChatCompletionClient` and `AssistantAgent` are constructed inside the async `execute()` body. If you ever hoist them to `__init__` for reuse, do it lazily inside an async method — not in a sync constructor.
- `cancel()` is a no-op (`pass`). The platform may call it on disconnect; until it does something, an in-flight `agent.run()` will keep running until the OpenAI call finishes. Do not assume canceling the A2A task halts model spend.
- `set_current_task` is wrapped in `try / finally` so the heartbeat label is cleared even on exception. Preserve that pattern when adding new error paths.

---

## Conventions / what NOT to do

- **Do not** hold workspace-mutable state on `AutoGenA2AExecutor`. Each `execute()` call rebuilds the agent. Conversation history comes from `extract_history(context)`, not from any in-process memory.
- **Do not** add provider-specific clients here. If multi-provider routing is needed, route inside `_common_setup` upstream (in `molecule_runtime`); keep this template OpenAI-only to match `requirements.txt` and `config.yaml`'s declared models.
- **Do not** edit `adapter.py` to bypass `_common_setup` — the platform expects every adapter template to expose the same delegation / memory / sandbox / approval tool set, and that set is built there.
- **Do not** bump `template_schema_version` without coordinating with the platform team — `config.yaml` declares `template_schema_version: 1` and the canvas template picker reads it.
- **Do not** modify the Dockerfile's `ARG RUNTIME_VERSION=` line or its position above the `pip install` layer — it is the cache-bust trigger that fixes cascade-build staleness.
- **Do not** add a `CLAUDE.md`-driven runtime config; everything the runtime reads is in `config.yaml`. `CLAUDE.md` is for repo contributors only.

---

## Where to make changes

| Want to... | Edit... |
|---|---|
| Add or remove an OpenAI model | `config.yaml` `models:` block + bump `version:` |
| Change the default system prompt | `system-prompt.md` |
| Pick up a newer `molecule-ai-workspace-runtime` | Bump pin in `requirements.txt`; the cascade publish workflow handles the image |
| Change tool-bridge behavior | `_langchain_to_autogen` in `adapter.py` |
| Switch to multi-agent (`GroupChat` / `Swarm`) | `AutoGenA2AExecutor.execute` — re-wire the agent construction; keep the LangChain bridge intact |
