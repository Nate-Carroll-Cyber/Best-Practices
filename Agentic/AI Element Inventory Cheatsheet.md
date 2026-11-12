# AI Element Inventory

Compiled 12 November 2026.

Every element below carries a definition, and the discovery indicators for an element sit directly under it. The inventory breaks down an artificial intelligence (AI) system, which NIST AI 100-3 describes as a machine-based system that produces predictions, recommendations, or decisions for a given set of objectives, affects real or virtual environments, and runs with varying degrees of autonomy.

Three markers are used.

- A double dagger (‡) marks an element that uses the NIST AI 100-3 term, with a paraphrase of the glossary's definition. Every other definition is a working definition written for this inventory. Where the Securing Agentic AI framework glossary defines the element, the working definition follows it.
- An asterisk (\*) marks an element that the Securing Agentic AI framework does not name.
- A dagger (†) marks an indicator that comes from recall and was not checked against a source on the compile date. Confirm these before they go into a detection rule.

---

## Models

### model ‡

A function that maps input features to predicted outputs.

### language model ‡

An approximate description of the patterns and regularities in natural language, used to make assumptions about language it has not seen.

### large language model (LLM) ‡

A language model that uses deep learning and is trained on very large text datasets. The glossary splits LLMs into generative models that output text and discriminatory models that classify it.

### SLM\*

A language model small enough to run on a single GPU or an end-user device, usually specialized for a narrow task. No standard sets the size cutoff, and published glossaries range from a few billion parameters to 10 billion.

### Fine-tuned model

A model whose weights were trained further from a base model on task-specific or organization-specific data.

### Adapter and checkpoint

An adapter is a small set of added weights that modifies a base model without replacing it. A checkpoint is a saved snapshot of model weights taken during or after training.

| Indicator | Examples |
|---|---|
| Adapter files † | `adapter_model.safetensors`, `adapter_config.json` |

### Embedding model\*

A model that converts text or other content into vectors for search and retrieval.

### Guardrail or moderation model

A model or service that screens prompts and outputs against policy and returns a verdict used to block, warn, or route.

### Auxiliary model

A secondary model that processes the primary model's content, such as a summarizer, reranker, classifier, translator, or redactor.

### Model weights

The numeric values inside an AI model that shape what the model returns for a given input. This follows NIST SP 800-218A.

| Indicator | Examples |
|---|---|
| Weight files † | `.safetensors`, `.gguf`, `.ggml`, `.bin`, `.pt`, `.pth`, `.ckpt`, `.onnx`, `.tflite`, `.h5`, `.keras`, `.pb`, `.mlmodel`, `.mlpackage`, `.engine` |
| Companion files † | `config.json`, `tokenizer.json`, `tokenizer_config.json`, `generation_config.json`, `Modelfile` |
| Local model stores † | `~/.cache/huggingface/hub/models--<org>--<name>`, `~/.ollama/models/blobs/sha256-*`, `~/.lmstudio/models` |

---

## Model access

### Frontier model API keys

A static secret issued by a commercial model provider that authenticates requests to its hosted models and bills them to the holder's account. "Frontier model" is not a NIST term.

| Provider | Prefix | Environment variable |
|---|---|---|
| OpenAI | `sk-proj-`, `sk-svcacct-`, `sk-admin-`, and legacy `sk-`, all carrying the `T3BlbkFJ` marker | `OPENAI_API_KEY` |
| Anthropic | `sk-ant-api03-` and `sk-ant-admin01-`, plus `sk-ant-oat01-` for subscription OAuth tokens | `ANTHROPIC_API_KEY` |
| Google Gemini | `AIza` | `GEMINI_API_KEY`, `GOOGLE_API_KEY` † |
| Amazon Bedrock | `ABSK` for long-term keys (132 characters), and `bedrock-api-key-` for short-term keys | `AWS_BEARER_TOKEN_BEDROCK` |
| AWS IAM | `AKIA` for long-term keys and `ASIA` for STS keys | `AWS_ACCESS_KEY_ID` |
| Hugging Face | `hf_`, and `api_org_` for org tokens | `HF_TOKEN` † |
| OpenRouter | `sk-or-v1-` plus 64 hex characters | `OPENROUTER_API_KEY` † |
| Groq | `gsk_` | `GROQ_API_KEY` † |
| Replicate | `r8_` | `REPLICATE_API_TOKEN` † |
| Databricks | `dapi` plus 32 hex characters | `DATABRICKS_TOKEN` † |
| Others † | `xai-` (xAI), `pplx-` (Perplexity), `nvapi-` (NVIDIA), `fw_` (Fireworks), `lsv2_` (LangSmith), `pcsk_` (Pinecone) | |

Azure OpenAI, Cohere, and Mistral keys have no fixed prefix, so they need context matching on the variable name. †

A bare `sk-` match is noisy because several vendors use it. The `T3BlbkFJ` marker is base64 for "OpenAI", so it is the reliable anchor for OpenAI keys.

Wiz recommends matching Bedrock long-term keys on the `ABSK` prefix alone, because the keys can attach to IAM users with any username.

| Indicator | Examples |
|---|---|
| Persisted keys in the Windows registry † | `HKCU\Environment` and `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment`, which is where `setx` writes the variables above |

### application programming interface (API) ‡

A software contract between an application and a client, expressed as a set of functions the client can call. Hosted model endpoints are the instances inventoried here.

| Indicator | Examples |
|---|---|
| Hosted API hostnames | `api.openai.com`, `api.anthropic.com`, `generativelanguage.googleapis.com`, `aiplatform.googleapis.com`, `bedrock*.amazonaws.com`, `*.openai.azure.com`, `api.mistral.ai`, `api.groq.com`, `openrouter.ai` |
| Local API paths | `/v1/chat/completions` for OpenAI-compatible servers, `/api/chat` and `/api/tags` for Ollama |

### Model routing layer or LLM gateway

A gateway component that selects among providers or models, holds their credentials, and mediates every prompt and response.

| Indicator | Examples |
|---|---|
| Default port | LiteLLM Proxy 4000 |

### inference ‡

The stage of machine learning in which a trained model is applied to a task. Local inference runtimes are the instances inventoried here.

| Indicator | Examples |
|---|---|
| Local inference servers | Ollama 11434, LM Studio 1234, LocalAI 8080, llama.cpp 8080, Jan 1337, GPT4All 4891, text-generation-webui 7860 for the UI and 5000 for the API, vLLM 8000 |
| Front ends | Open WebUI 3000 |
| Model serving † | Triton 8000, 8001, 8002. TF Serving 8500, 8501. TorchServe 8080, 8081. |
| Installed apps in the Windows registry † | `HKCU` and `HKLM\Software\Microsoft\Windows\CurrentVersion\Uninstall\*`, matched on `DisplayName` for Ollama, LM Studio, Cursor, Claude |

### Model hub and internal mirror

A model hub is a public or private repository that distributes model weights and datasets. An internal mirror is the organization-operated registry through which every dependency, image, and model artifact is resolved.

---

## Agents

### Agent

An AI-enabled system that can plan or select actions and invoke tools.

### Sub-agent

An agent started by another agent to carry out a delegated part of a task.

### Orchestrator

An agent or service that decomposes tasks, routes them to other agents, and handles retry and recovery.

### Agent framework and runtime

The software that runs the agent loop, including model calls, tool dispatch, memory, and state.

### Agent registry

The authoritative record of every approved agent and tool server, with its identity, owner, endpoint, version, and lifecycle state.

### A2A Agent Card

The JSON document an A2A agent publishes to describe its identity, skills, capabilities, and endpoints.

| Indicator | Examples |
|---|---|
| Discovery path | `/.well-known/agent-card.json`. Older deployments use `/.well-known/agent.json`. |
| Transports | JSON-RPC, gRPC, or HTTP+JSON |

### Skill and plugin

A packaged set of instructions, scripts, or tool bindings that extends what an agent can do.

| Indicator | Examples |
|---|---|
| Skill and bundle files † | `SKILL.md`, `.mcpb` bundles |

---

## Tools

### MCP server

A program that exposes tools, resources, and prompts to MCP clients over the Model Context Protocol, either as a local process or over HTTP.

| Indicator | Examples |
|---|---|
| Transports | stdio with no port, or Streamable HTTP and legacy SSE |
| Typical local URL | `http://localhost:3000/mcp` |
| MCP Inspector | 6274 for the UI and 6277 for the proxy |

### MCP client or host

The application that connects to MCP servers and passes their tools and context to a model. The host is the application, and the client is its connector to one server.

| Indicator | Examples |
|---|---|
| Claude Desktop policy in the Windows registry | `HKLM\SOFTWARE\Policies\Claude` with `isLocalDevMcpEnabled`, `isDesktopExtensionEnabled`, `isDesktopExtensionDirectoryEnabled`, `isClaudeCodeForDesktopEnabled`. The same path exists under HKCU. |

### Tool definition and manifest

The name, description, and input schema a tool advertises to the model, together with the versioned record of its owner, permissions, and artifact digest.

### Third-party connector

A vendor-built integration that gives an agent access to an external application or data source under granted scopes.

### Tool execution broker

The deterministic policy decision and enforcement point that every consequential agent action passes through.

### Code execution sandbox

An isolated, disposable environment in which model-generated or untrusted code runs.

### Cached fetch service

A controlled intermediary that retrieves external content on behalf of workloads that have no direct internet reach.

---

## Developer surface

### copilot ‡

An AI-powered program that assists users with tasks and automates features inside compatible applications, using language models and a conversational interface. AI-enabled IDEs and AI features embedded in SaaS platforms are the instances inventoried here.

| Indicator | Examples |
|---|---|
| VS Code extension IDs † | `GitHub.copilot`, `GitHub.copilot-chat`, `saoudrizwan.claude-dev`, `rooveterinaryinc.roo-cline`, `Continue.continue`, `anthropic.claude-code` |
| Windows Copilot policy | `SOFTWARE\Policies\Microsoft\Windows\WindowsCopilot` with `TurnOffWindowsCopilot`, user scope |
| Windows Recall policy | `SOFTWARE\Policies\Microsoft\Windows\WindowsAI` with `DisableAIDataAnalysis` and `AllowRecallEnablement` |

Microsoft states that `TurnOffWindowsCopilot` does not apply to the newer Copilot experience. It is not a complete Copilot control.

### Coding agent CLI\*

A command-line agent that reads and edits a codebase and runs commands on the developer's machine.

### AI browser extension\*

A browser add-on that sends page content to a model or lets a model act in the browser.

| Indicator | Examples |
|---|---|
| Chrome policy and host keys † | `HKLM\SOFTWARE\Policies\Google\Chrome\ExtensionInstallForcelist`, `HKCU\Software\Google\Chrome\NativeMessagingHosts\*` |

### Agent and MCP configuration files

Files that register MCP servers, set agent permissions, and supply standing instructions. They are active input and not inert data.

| Client | Paths |
|---|---|
| Claude Desktop | `%APPDATA%\Claude\claude_desktop_config.json`, `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Claude Code | `~/.claude.json` for user scope, `.mcp.json` for project scope |
| Cursor | `~/.cursor/mcp.json`, `.cursor/mcp.json` |
| VS Code | `.vscode/mcp.json`, `%APPDATA%\Code\User\mcp.json`, with a `servers` key where the others use `mcpServers` |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` |
| Cline and Roo Code | `...\globalStorage\saoudrizwan.claude-dev\settings\cline_mcp_settings.json`, `~/.roo/mcp.json` |
| Other CLIs | `~/.codex/config.toml`, `~/.gemini/settings.json`, `~/.config/zed/settings.json` |
| Instruction files † | `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, `.cursor/rules/`, `.windsurfrules`, `.github/copilot-instructions.md` |

---

## Prompts and context

### System prompt and developer prompt

Instructions set by the operator or developer that the model receives ahead of user input.

### Context window

The bounded span of tokens a model can read in one call, holding instructions, conversation, retrieved content, and tool output.

### Agent memory

State that an agent writes during one run and reads in a later run.

### word embedding ‡

A representation of words as vectors, trained on word co-occurrence, in which words with similar meanings sit close together.

### Vector store

A database that stores embeddings and returns the nearest matches to a query vector.

| Indicator | Examples |
|---|---|
| Default ports † | Qdrant 6333 and 6334, Milvus 19530, Weaviate 8080 and 50051, Chroma 8000, pgvector 5432 |

### Retrieval index and knowledge sources

The indexed documents and data an agent retrieves at run time to ground its answers.

---

## Data

### training data ‡

The dataset from which a model is learned.

### Fine-tuning dataset

The task-specific or organization-specific examples used to adapt a base model.

### Evaluation dataset and results

The held-out examples used to measure a model, and the scores they produce.

### Prompt and response logs

Stored records of what was sent to a model and what it returned.

### Reasoning traces

The model's recorded intermediate reasoning, where the platform exposes it.

---

## Credentials and identity

### Agent workload identity

The cryptographically verifiable identity of the process an agent runs as.

### User access token and exchanged OBO token

The token a signed-in user presents, and the downstream token issued in exchange for it that names the user as subject and the agent as actor.

### Standing delegation grant

A refresh token or provider equivalent that lets an agent act for a user who is not present.

### Tool and MCP server credentials

The secrets a tool or MCP server uses to reach the systems behind it.

### Mirror and registry credentials

The secrets that let a workload read from artifact mirrors and registries, or let a service write to them.

### Signing keys

The private keys that sign audit records, attestations, artifacts, and AIBOMs.

---

## Pipeline and infrastructure

### model training ‡

The phase in which practitioners fit weights and bias to a machine learning algorithm to minimize a loss function. Training and fine-tuning pipelines are the instances inventoried here.

| Indicator | Examples |
|---|---|
| Notebooks and trackers † | Jupyter 8888, Gradio 7860, Streamlit 8501, MLflow 5000 |

### Evaluation harness

The tooling that runs a model against evaluation datasets and records the results.

### graphical processing unit (GPU) ‡\*

A specialized chip built for highly parallel processing and well suited to running machine learning and deep learning workloads.

### AIBOM

A scoped, machine-readable record of an AI system comprising a header, components and services, directed data flows, and evidence references.

### model card ‡

A short document that accompanies a trained model and discloses its intended use, its evaluation procedure, and its benchmarked performance across relevant conditions and groups.

### model inventory ‡

A database that aggregates information about the quantitative models an organization has in use.

---

## Source notes

The NIST AI 100-3 terms and definitions were read from the published glossary sheet on the compile date. Its title line says it was updated on August 4, 2024, and it holds 447 terms. It has no entry for agent, MCP, tool, prompt, context window, fine-tuning, weights, guardrail, sandbox, API key, or bill of materials, which is why those elements carry working definitions.

Every indicator without a dagger was checked against a source on the compile date. Most of those sources are secret-scanner documentation and third-party guides. Vendor documentation was the source for the Anthropic key prefix, the Bedrock environment variable, the Claude Desktop registry policy, and the Windows Copilot and Recall policy keys.

Three other vocabularies define terms that AI 100-3 lacks. ISO/IEC 22989:2022 defines AI agent, and its draft Amendment 1 adds AI model, generative AI, and foundation model. Article 3 of the EU AI Act defines general-purpose AI model. NIST COSAiS plans SP 800-53 overlays for single-agent and multi-agent systems, and those overlays are unpublished.

## Sources

- [NIST AI 100-3 glossary, published sheet](https://docs.google.com/spreadsheets/d/e/2PACX-1vTRBYglcOtgaMrdF11aFxfEY3EmB31zslYI4q2_7ZZ8z_1lKm7OHtF0t4xIsckuogNZ3hRZAaDQuv_K/pubhtml)
- [NIST AI RMF glossary landing page](https://airc.nist.gov/glossary/)
- [NIST CSRC glossary, model weight](https://csrc.nist.gov/glossary/term/model_weight)
- [NIST AI 100-2e2025, Adversarial Machine Learning taxonomy](https://nvlpubs.nist.gov/nistpubs/ai/nist.ai.100-2e2025.pdf)
- [ISO/IEC 22989:2022/DAmd 1](https://committee.iso.org/standard/88145.html?browse=ics)
- [NIST COSAiS project](https://csrc.nist.gov/Projects/cosais)
- [EU AI Act Article 3](https://ai-act-service-desk.ec.europa.eu/en/ai-act/article-3)
- [OpenRouter secret formats](https://openrouter.ai/docs/guides/features/guardrails/secret-formats.md)
- [Bagel AI service detectors](https://boostsecurityio.github.io/bagel/detectors/ai-service)
- [Anthropic API key documentation](https://platform.claude.com/docs/en/get-api-key)
- [Wiz on Bedrock API keys](https://www.wiz.io/blog/a-new-type-of-long-lived-key-on-aws-bedrock-api-keys)
- [AWS Bedrock API key usage](https://docs.aws.amazon.com/bedrock/latest/userguide/api-keys-use.html)
- [Claude Desktop enterprise configuration](https://support.claude.com/en/articles/12622667-enterprise-configuration)
- [Microsoft WindowsAI policy CSP](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-windowsai)
- [MCP Inspector](https://github.com/wei/inspector)
- [A2A agent cards](https://www.specification.website/spec/agent-readiness/a2a-agent-cards/)
- [MCP client config locations](https://glama.ai/mcp/servers/@polaz/gitlab-mcp/blob/897bef9821fbf89738e6533d22bf9a28960194a1/docs/clients/index.md)
- [Local LLM default ports](https://localtonet.com/blog/expose-llms-to-the-internet)
