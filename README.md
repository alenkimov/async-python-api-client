# async-python-api-client

A reusable Codex skill for creating and maintaining clean, handwritten-style Python API clients from OpenAPI YAML or JSON specifications.

It guides Codex to build strictly async Python 3.12+ clients with a reused `httpx.AsyncClient`, Pydantic v2 models, `better-proxy.Proxy` support, typed authentication, pagination and API errors, explicit async lifecycle management, tests, and Ruff. It does not use OpenAPI code generators or create a generated-code layer.

The skill supports two workflows:

- create a compact client from scratch with the OpenAPI document as the source of truth;
- update an existing client with minimal changes while preserving its public API where reasonable.

## Install

Clone the repository into your personal Codex skills directory:

```shell
git clone https://github.com/alenkimov/async-python-api-client.git ~/.codex/skills/async-python-api-client
```

If `CODEX_HOME` points somewhere else, clone it into that directory's `skills/async-python-api-client` path instead. Restart Codex if the skill is not discovered immediately.

## Use

Invoke the skill explicitly and provide an OpenAPI file or URL plus the target repository. For example:

```text
Use $async-python-api-client to create a Python client from openapi.yaml.
```

```text
Use $async-python-api-client to update this client for the changes in openapi-v2.json. Keep unrelated public APIs unchanged.
```

Codex reads the focused workflow in `SKILL.md` and loads the detailed Python conventions from `references/python-client-conventions.md` as needed.
