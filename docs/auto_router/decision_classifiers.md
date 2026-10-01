---
title: Decision model classifiers
sidebar_label: Jev, Nimble and Laya
description: Route requests with hosted Jev or self-hosted Nimble and Laya using the Auto Router's decision-model classifier.
---

Use Jev, Nimble or Laya to choose which model answers each request. The classifier picks a complexity tier; the Auto Router then calls a model configured for that tier. Clients keep using one router model name through the normal chat completions API.

Jev is available through TypeSafe's hosted API. Nimble and Laya are open models you can run on your own infrastructure.

| Classifier | Where it runs | Classifier model | LiteLLM configuration |
| --- | --- | --- | --- |
| [Jev](https://docs.typesafe.ai/) | TypeSafe's hosted API | `jev-latest` | `classifier_type: jev` |
| [Bespoke Nimble](https://github.com/bespokelabsai/nimble) | Self-hosted, including Ollama | `nimble` on Ollama; `nimble-latest` on Bespoke's System One server | `classifier_type: jev` with a custom endpoint |
| [Laya](https://github.com/NandhaKishorM/laya) | Self-hosted | `english`, `multilingual` or `typed-decisions` | `classifier_type: jev` with `provider: laya` |

All three use the System One decision protocol. LiteLLM sends a `choice` question describing your tiers to `POST /v1/systemone`. Set `api_base` to the server's base URL without `/v1/systemone`; an endpoint that only exposes `/v1/evaluate` or chat completions is not sufficient.

:::info Laya availability

Laya's provider setting requires a gateway build containing [backend #43626](https://github.com/BerriAI/litellm/pull/43626). Its dashboard selector also requires [UI #43768](https://github.com/BerriAI/litellm/pull/43768). These changes are pending; existing releases can reject `provider: laya`. Jev and Nimble use the existing Jev-compatible client.

:::

## Configure the router

Start with this complete configuration, then choose the classifier connection below. `small-solver` and `large-solver` are public deployment names; replace their underlying models with models your gateway can access:

```yaml title="config.yaml"
model_list:
  - model_name: small-solver
    litellm_params:
      model: openai/{{openai_small}}
      api_key: os.environ/OPENAI_API_KEY
  - model_name: large-solver
    litellm_params:
      model: openai/{{openai_large}}
      api_key: os.environ/OPENAI_API_KEY
  - model_name: decision-router
    litellm_params:
      model: auto_router/complexity_router
      complexity_router_default_model: large-solver
      complexity_router_config:
        classifier_type: jev
        jev_classifier_config:
          model: jev-latest
          timeout_ms: 3000
        tiers:
          SIMPLE: small-solver
          MEDIUM: small-solver
          COMPLEX: large-solver
          REASONING: large-solver
        classifier_fallback: default_model
        classifier_context_window_size: 0
```

This example routes classifier failures to `large-solver` and sends no prior conversation turns to the classifier. The current request and selected system text are still sent. The default fallback, when omitted, is `heuristic`; the default history window is three prior user turns within an 8,000-character prior-turn budget.

### Jev: hosted TypeSafe API

Set `TYPESAFE_API_KEY` in the gateway's environment using your secret manager. Leave the example's `jev_classifier_config` unchanged. `TYPESAFE_API_BASE` is optional and defaults to `https://api.typesafe.ai`.

For a router-specific endpoint, supply both `api_base` and its matching `api_key` inside `jev_classifier_config`. An explicit endpoint does not inherit the TypeSafe environment key. See [Jev setup](/docs/auto_router/setup#jev-classifier-typesafe-ai) for dashboard configuration and the [Jev reference](/docs/proxy/auto_routing#jev-classifier) for the full settings.

### Nimble: self-hosted with Ollama

Install [Ollama 0.35 or later](https://ollama.com/library/nimble) and start its server. If the Ollama app is not already serving requests, run `ollama serve` in a separate terminal. Download the decision model:

```bash
ollama pull nimble
```

Replace the example's `jev_classifier_config` with this block:

```yaml
jev_classifier_config:
  model: nimble
  api_base: http://localhost:11434
  api_key: ollama
  timeout_ms: 30000
```

Local Ollama ignores the API key. `ollama` is a non-secret placeholder because LiteLLM's Jev-compatible client requires a nonempty key when an endpoint is supplied. If your endpoint has authentication, use that endpoint's real credential instead. There is no `provider: nimble` setting.

The longer timeout allows more time for local inference; warm the model before measuring latency. For Bespoke's own System One server, use its base URL and `model: nimble-latest` instead. See the upstream [serving guide](https://github.com/bespokelabsai/nimble/blob/main/docs/MODAL_SERVING.md) and [model preparation guide](https://github.com/bespokelabsai/nimble#quickstart).

### Laya: self-hosted HTTP server

Install Laya in a separate environment from LiteLLM and start the English checkpoint:

```bash
python -m venv .venv-laya
source .venv-laya/bin/activate
python -m pip install 'laya[serve]==0.3.21'
LAYA_HOST=127.0.0.1 LAYA_PORT=8000 LAYA_MODELS=english laya-serve
```

In the gateway process, set its reachable server URL:

```bash
export LAYA_API_BASE="http://127.0.0.1:8000"
```

Replace the example's `jev_classifier_config` with this block:

```yaml
jev_classifier_config:
  provider: laya
  model: english
  timeout_ms: 15000
```

Choose `english`, `multilingual` or `typed-decisions` explicitly. When `api_base` is omitted, the gateway uses `LAYA_API_BASE` and optional `LAYA_API_KEY`. To require authentication, configure the same key on the Laya server and gateway.

An administrator can instead set `api_base` and optional `api_key` on this router. An explicit Laya endpoint without a key connects without authentication and does not inherit the environment key. See the [Laya HTTP guide](https://github.com/NandhaKishorM/laya/blob/main/docs/http-api.md) for server options.

For either self-hosted server, `localhost` refers to the gateway's own host or container. Use a hostname reachable from the gateway when the classifier runs elsewhere.

## Test routing and send a request

Start LiteLLM from its own environment after configuring the completion-provider credentials and your chosen classifier:

```bash
litellm --config config.yaml
```

In **Models + Endpoints → Auto Router**, open the create or edit form and use **Test Routing** to inspect the chosen tier and model without calling the completion model. A successful fallback does not prove the classifier worked: inspect the routing cause and classifier model. A decision-model result uses `cause: jev_classifier`; Laya reports a classifier model such as `laya/english`.

Use a LiteLLM virtual key with access to `decision-router` to make a completion:

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "decision-router",
    "messages": [{"role": "user", "content": "Explain what an HTTP status code means"}]
  }'
```

In dashboards containing the Laya UI integration, select **Decision Model** in the auto-router form, then **TypeSafe Jev** or **Laya**. Nimble uses the TypeSafe Jev-compatible path with a custom model and endpoint; YAML is the simplest setup. Older dashboards label the classifier **JEV Classifier**.

Administrators can configure connection overrides. Team members cannot submit classifier endpoint or credential overrides through the management API. Saved connection fields are hidden on edit; leaving them untouched preserves them. Use the explicit reset controls to clear a saved key or return to the gateway connection.

## Evaluate quality and cost

Compare tier choices on representative prompts before changing production routing. Probabilities describe the supplied choices; confidence scores from different model families are not interchangeable accuracy estimates. Measure downstream answer quality, classifier latency, fallback frequency and total cost using the [evaluation guide](/docs/auto_router/evaluate).

Jev calls can incur TypeSafe charges. Self-hosting Nimble or Laya has compute costs even without a hosted inference fee. Laya's built-in catalog rates are zero; custom Nimble model names may have no registered rate, so missing classifier cost does not establish zero cost. Nimble routed through the Jev-compatible client retains `typesafe/<model>` classifier log naming; Laya uses `laya/<checkpoint>`.

If the classifier falls back, check the endpoint, checkpoint name, credentials, model warm-up and timeout. A timeout can also open the classifier circuit breaker, which defaults to a 30-second recovery interval. Verify that the server accepts `/v1/systemone`, rather than adding that path to `api_base`.
