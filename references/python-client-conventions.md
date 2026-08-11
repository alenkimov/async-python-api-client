# Python API client conventions

Use these conventions as defaults, not as a demand for framework-sized architecture. Match the existing repository when updating a client unless doing so would contradict the OpenAPI contract or the user's request.

## Contents

- [Design boundaries](#design-boundaries)
- [Project and package shape](#project-and-package-shape)
- [Models and serialization](#models-and-serialization)
- [Client lifecycle and transport](#client-lifecycle-and-transport)
- [Proxy support](#proxy-support)
- [Operations and parameters](#operations-and-parameters)
- [Authentication](#authentication)
- [Responses and errors](#responses-and-errors)
- [Public return types and response envelopes](#public-return-types-and-response-envelopes)
- [Pagination](#pagination)
- [Special request and response bodies](#special-request-and-response-bodies)
- [Updating from a changed spec](#updating-from-a-changed-spec)
- [Testing and quality checks](#testing-and-quality-checks)
- [Final review checklist](#final-review-checklist)

## Design boundaries

- Target Python 3.13+ and use native modern typing: `X | None`, built-in generics, `Self`, `Literal`, `TypedDict`, `Protocol`, and `StrEnum` where they improve the public contract.
- Avoid `from __future__ import annotations` by default. Prefer clear declaration order and quoted forward references where needed so annotations work without postponed evaluation. Keep or add the future import only when it solves an unavoidable forward-reference or circular-typing problem, or when existing code relies on its annotation semantics; do not remove it mechanically during unrelated updates.
- Expose async network operations only. Do not add a sync facade, call blocking HTTP libraries, or hide blocking work inside `async` methods.
- Handwrite readable domain code. Never invoke OpenAPI Generator or a similar generator, commit generated SDK output, or create a `generated/` package.
- Keep the OpenAPI document as the behavioral authority. Preserve vendor extensions only when they carry needed semantics.
- Prefer direct code over factories, repositories, service locators, generic endpoint engines, or one-class-per-operation structures.
- Keep small APIs small. Split modules only when file size, domain boundaries, or import clarity justifies it.
- Never log credentials, authorization headers, proxy passwords, sensitive request bodies, or full sensitive URLs.

## Project and package shape

For a new modest client, start near this shape and remove files that do not earn their place:

```text
pyproject.toml
src/<package>/
├── __init__.py
├── _envelopes.py
├── client.py
├── enums.py
├── errors.py
├── models/
│   ├── __init__.py
│   ├── base.py
│   └── <domain>.py
└── py.typed
tests/
├── test_client.py
└── test_models.py
```

Put OpenAPI enums in the package-level `enums.py` so the client and model modules can share them without creating model-package import cycles.

Split `models/` primarily by API domain or resource, normally following stable OpenAPI tags or cohesive operation groups such as `nfts.py`, `events.py`, and `collections.py`. Keep a request or response model with the nested schemas chiefly owned by the same domain. Put only the base Pydantic model and mixins in `base.py`; use `common.py` only for genuinely cross-domain schemas with no clear owner. For a very small API, keep one domain module inside `models/` rather than inventing several tiny files.

Do not split models mechanically into one file per schema, separate request and response trees, alphabetical buckets, or files based only on line count. If a domain module becomes unwieldy, split it by a stable subdomain or operation family. Keep dependency direction clear to avoid circular imports, and re-export the intended public models from `models/__init__.py` and the package `__init__.py`.

Declare runtime dependencies such as `httpx`, `pydantic`, and `better-proxy` in project metadata. Respect the repository's package manager and version policy. For a new package, provide a Python 3.13+ constraint and conventional build metadata without adding unrelated tooling.

## Models and serialization

- Use Pydantic v2 `BaseModel`; use `model_validate`, `model_dump`, `ConfigDict`, and v2 validators rather than v1 compatibility methods.
- Generate request and response types from schemas, including nested objects, arrays, enums, formats, aliases, defaults, and requiredness.
- Distinguish a missing field from a nullable field. Do not translate every non-required property into an explicit `None` on outgoing requests.
- Serialize request models with `mode="json"`, `by_alias=True`, and usually `exclude_unset=True`. Use `exclude_none=True` only when the API treats explicit null and omission identically.
- Parse structured responses with `Model.model_validate(response.json())`. Use `TypeAdapter` for top-level arrays, unions, mappings, or scalar schemas.
- Define documented enums in the package-level `enums.py`, using `StrEnum` for string values when callers benefit from named members. Preserve unknown server values when forward compatibility is more important than a closed enum and the spec permits it.
- Map `date`, `date-time`, UUID, URI, decimal, and binary formats to useful Python types when the wire contract is clear.
- Use field aliases for non-Python identifiers. Keep Python attribute names idiomatic while emitting exact wire names.
- Model discriminated unions when the specification defines a discriminator. Avoid broad `Any`; confine it to genuinely free-form schemas.
- Choose response extra-field behavior deliberately. Ignoring unknown response fields is often forward-compatible; forbidding extras is useful only when strictness is part of the contract.
- Give public methods typed request objects or typed keyword parameters and typed return values. Do not make callers assemble undocumented dictionaries.

## Client lifecycle and transport

Create one `httpx.AsyncClient` per API client instance and reuse it across requests for pooling and shared configuration. Never create it inside each endpoint method.

A practical constructor may accept credentials, `base_url`, timeout, proxy configuration, and an optional caller-supplied `httpx.AsyncClient`. Keep only parameters the API or repository needs.

Track transport ownership:

- Close an internally created `AsyncClient` in `aclose()`.
- Do not close a caller-supplied `AsyncClient` unless the public contract explicitly transfers ownership.
- Implement `async def __aenter__(self) -> Self` and `async def __aexit__(...) -> None` in terms of the same lifecycle.
- Make repeated `aclose()` calls harmless where practical.

Configure stable concerns such as `base_url`, default headers, timeout, proxy, limits, redirects, and TLS behavior when constructing the shared client. Keep request-specific query parameters, headers, and bodies on each request.

Allow dependency injection of an `AsyncClient` or `AsyncBaseTransport` when it makes testing easier, but do not add a custom transport abstraction solely for tests.

## Proxy support

Use `better_proxy.Proxy` as the typed proxy representation. When accepting strings is helpful, normalize them once with `Proxy.from_str(...)`. Pass `proxy.as_url` to the `proxy=` parameter when constructing `httpx.AsyncClient`.

When both a proxy and a caller-supplied `httpx.AsyncClient` are provided, reject the combination at construction with a clear configuration error. Proxy transport configuration belongs to the supplied client; do not silently ignore the proxy, mutate the supplied client, or replace its transport.

Do not mutate proxy settings per request or rebuild the async client to rotate proxies. If proxy rotation is an explicit requirement, design a lifecycle-aware pool separately and test its concurrency behavior.

Keep proxy credentials out of representations, exceptions, and logs. Account for the selected HTTPX extras when SOCKS support is required by the specification or user.

## Operations and parameters

Derive each method from an OpenAPI operation:

- Prefer a stable, idiomatic method name based on `operationId`; use path and method only when `operationId` is missing or unusable.
- Match the HTTP method and path exactly. Quote and encode path parameters through HTTPX rather than interpolating untrusted raw values carelessly.
- Keep required parameters required. Preserve OpenAPI defaults and distinguish omitted values from explicit nulls.
- Put parameters in the documented location: path, query, header, cookie, or body.
- Preserve repeated query parameters and the declared OpenAPI `style` and `explode` behavior. Use a sequence of key-value pairs when duplicate keys matter.
- Use a typed request body model when a schema exists. Keep a body separate from transport-only options.
- Accept per-call timeout or headers only if the existing public API or a real use case requires them; avoid a generic `**kwargs` escape hatch that weakens typing.
- Handle every documented successful status code, not only `200`.

Factor a small private `_request` helper when it centralizes shared auth, status handling, and parsing. Keep operation-specific serialization and return types visible in endpoint methods. Avoid a data-driven endpoint registry that resembles generated code.

## Authentication

Read `components.securitySchemes` and effective `security` at both root and operation level. An operation-level declaration overrides the root, and an empty security requirement means anonymous access.

- For API keys, place the value in the exact documented header, query parameter, or cookie.
- For HTTP bearer schemes, send `Authorization: Bearer <token>`.
- For HTTP basic schemes, use HTTPX authentication support or an equivalent correctly encoded header.
- For OAuth2 or OpenID Connect, accept an access token when token acquisition is outside scope. Implement a flow only when requested and sufficiently specified.
- Preserve OpenAPI security OR/AND semantics. Do not force global authentication onto anonymous operations.
- Validate missing required credentials before sending a request and raise a clear configuration or authentication error without revealing the secret.

## Responses and errors

Validate successful responses against the documented schema, then expose the most ergonomic truthful public return type. Follow the response-envelope rules below. For `204`, `205`, or a documented empty body, return `None`. Handle text or bytes directly when the media type is not JSON.

Treat wildcard media types such as `*/*` as content negotiation, not as evidence of text or binary data. For a structured response schema, prefer JSON decoding and Pydantic validation when the runtime `Content-Type` is JSON (including a `+json` subtype) or the API contract indicates JSON; use text or bytes only when the runtime content type or contract indicates non-JSON data.

Define a sensible base exception such as `APIError` with safe diagnostic fields:

- HTTP status, method, and sanitized URL;
- parsed structured error data or a bounded body excerpt;
- documented error code and message when present;
- request or correlation ID from documented headers.

Add subclasses such as authentication, permission, not-found, conflict, validation, or rate-limit errors only when the API documents those categories or callers can act on them. Include retry metadata such as `Retry-After` on rate-limit errors when available.

Translate `httpx.TimeoutException`, `NetworkError`, and related transport failures into a typed client transport error only when the package promises a stable exception surface; retain the original exception as `__cause__`. Never collapse cancellation into an API error.

Parse documented non-2xx error schemas before falling back to safe JSON, text, or bytes summaries. Bound captured response bodies and redact credentials. Do not call `raise_for_status()` before extracting useful documented error details.

## Public return types and response envelopes

Distinguish domain models from transport-only response envelopes. Create Pydantic models for reusable domain entities and value objects. Do not create or expose a response model solely because an OpenAPI response wraps already typed values in a top-level JSON object.

Treat a response object as a transport-only envelope when it:

- is used only as the root response of an endpoint;
- merely groups an already typed payload with a cursor or simple metadata;
- has no independent domain meaning, validation rules, or reuse elsewhere.

Validate the wire envelope internally, then unwrap it into an ergonomic public return type:

- Return `T` or `list[T]` for an envelope containing one useful payload.
- Return `tuple[list[T], str | None]` for a paginated payload and cursor.
- Return `tuple[A, B]` for exactly two stable, independently meaningful values.

Keep tuples limited to two values and make their order clear through the method annotation, documentation, and unpacking examples. Retain a named response model when the response has more than two meaningful fields, complex optional metadata, independent semantics, reuse across endpoints, or is likely to evolve in a way that would make a tuple ambiguous.

Implement unwrapped transport envelopes as private `TypedDict` schemas validated by module-level `TypeAdapter` instances. Describe exact wire keys with `Required` and `NotRequired`, instantiate each adapter once, and use `validate_json()` to validate the complete response before unwrapping it. Convert missing keys, invalid shapes, and validation failures into the client's typed invalid-response error.

Always place these private envelope schemas and adapters in the package-level `_envelopes.py`, even for a small client, so clients share a consistent structure. Keep the module private and do not re-export its schemas or adapters. Use a private `BaseModel` instead only when the envelope itself requires cross-field validation, normalization, or other model-specific behavior.

When updating an existing client, apply envelope unwrapping only when explicitly requested because it changes public return types.

## Pagination

Infer pagination from parameters, response schemas, headers, and links rather than assuming a convention.

- Keep a typed single-page method when callers need metadata or manual control.
- Add an async iterator when it meaningfully simplifies traversal; fetch lazily and stop on the documented terminal condition.
- Support the actual mechanism: cursor, continuation token, offset/limit, page number, response link, or header link.
- Preserve user-supplied page size and filters across pages.
- Detect missing or repeated continuation values when they could cause an infinite loop.
- Do not eagerly collect all pages unless the public method explicitly promises a list.
- Do not add pagination helpers to endpoints the specification does not paginate.

## Special request and response bodies

- Use `files=` and `data=` correctly for multipart forms; type file inputs without reading large files into memory unnecessarily.
- Use `content=` for raw binary bodies and stream large downloads with HTTPX streaming APIs when required.
- Honor declared content types and Accept headers when an operation offers multiple representations.
- Represent optional uploads, arrays, and nested multipart fields according to the specification rather than JSON-encoding them by habit.
- Avoid automatic retries for non-idempotent operations. Add retry behavior only when requested, bounded, observable, and safe for the relevant methods or idempotency keys.

## Updating from a changed spec

Before editing, establish a change map covering paths, methods, operation IDs, parameters, schemas, required fields, enums, security, response statuses, error schemas, and pagination.

Then trace each changed component to models, methods, exports, docs, and tests. Update only those paths plus shared code that must change to support them.

Preserve the existing public API when it remains accurate:

- Retain established method and model names if the spec change does not invalidate them.
- Add optional parameters without reordering existing positional parameters; prefer keyword-only additions.
- Avoid renaming public imports merely to mirror cosmetic spec wording.
- Do not create aliases, duplicate methods, or compatibility modules. If the old contract is no longer truthful, make the direct change and report it.
- Avoid opportunistic rewrites, global reformatting, dependency upgrades, and model reorganization.

When only a new spec is available, compare it systematically with implemented behavior and state that inferred differences are based on the implementation rather than an old-spec diff.

## Testing and quality checks

Use the existing test stack. For a new package, prefer pytest with async support and either `httpx.MockTransport` or the repository's established HTTPX mocking library. Do not make real network calls in unit tests.

Cover at least:

- client reuse, context management, internal versus external transport ownership, and `aclose()`;
- exact method, URL, path/query encoding, headers, authentication, and serialized body;
- typed parsing for each distinct response shape and empty success responses;
- representative documented errors, malformed or non-JSON errors, and safe redaction;
- each pagination mechanism and its stopping condition;
- proxy normalization and client configuration without contacting a real proxy;
- changed behavior and regression cases when updating an existing client.

Run focused tests during iteration, then the full suite. Run Ruff formatting and lint checks when configured. When creating a new project, add a small Ruff configuration and tests rather than a large toolchain. Use the repository's package manager for all commands and dependency changes.

## Final review checklist

- Confirm that every requested operation matches the current OpenAPI document.
- Confirm that all public network methods are async and fully typed.
- Confirm that `from __future__ import annotations` is absent unless its use is justified.
- Confirm that one shared `httpx.AsyncClient` is reused and closed according to ownership.
- Confirm that Pydantic v2 APIs and Python 3.13+ typing are used.
- Confirm that `better-proxy.Proxy`, authentication, errors, and pagination match the actual contract.
- Confirm that no generator output, `generated/` layer, sync facade, or unnecessary abstraction was introduced.
- Confirm that tests exercise wire behavior rather than only model construction.
- Confirm that the final diff is minimal for an update and that unavoidable breaking changes are reported.
