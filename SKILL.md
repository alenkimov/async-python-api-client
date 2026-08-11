---
name: async-python-api-client
description: Create and maintain clean, handwritten-style Python 3.13+ API clients from OpenAPI YAML or JSON specifications. Use when Codex must build a strictly async client from scratch, compare an existing client with a changed OpenAPI spec, add or revise endpoints and models with minimal churn, or improve typed authentication, pagination, errors, proxy support, and tests without using an OpenAPI code generator.
---

# Async Python API Client

Treat the OpenAPI document as the source of truth while writing ordinary maintainable Python. Do not run an OpenAPI code generator, introduce a `generated/` layer, or reproduce boilerplate that the API does not need.

Read [references/python-client-conventions.md](references/python-client-conventions.md) before designing or editing the client. Apply its conventions in proportion to the API's size and the repository's established style.

## Establish the scope

1. Locate and parse every relevant OpenAPI YAML or JSON file, including referenced components.
2. Inspect repository instructions, package metadata, source layout, public exports, tests, and current client behavior.
3. Inventory operations, schemas, security schemes, pagination contracts, error responses, uploads, and unusual content types.
4. Resolve conflicts in favor of the current specification, but report material ambiguities instead of guessing silently.

## Choose the workflow

### Create a client

1. Choose the smallest clear package layout for the number of operations and models.
2. Implement Pydantic v2 request and response models, then typed async operations over one reused `httpx.AsyncClient`.
3. Derive authentication, pagination, response handling, and error behavior from the specification.
4. Add `better-proxy.Proxy` support, explicit lifecycle ownership, `aclose()`, and async context-manager methods.
5. Add focused tests and Ruff configuration when the project uses them or when creating a new package.

### Update a client

1. Diff the old and new specifications when both are available; otherwise map the current spec against the implementation and tests.
2. Trace only affected operations, schemas, exports, authentication, pagination, errors, and tests.
3. Make the smallest coherent edits. Preserve names, signatures, return types, and import paths when they remain truthful and practical.
4. Do not add compatibility aliases or speculative abstraction layers. Call out unavoidable public breaking changes caused by the specification.
5. Leave unrelated formatting and behavior untouched.

## Verify the result

1. Check every implemented operation against its HTTP method, path, parameters, body, success statuses, response schema, security, and pagination metadata.
2. Run the narrowest relevant tests first, then the full available test suite and Ruff checks.
3. Review the diff for accidental generated-style repetition, sync I/O, per-request client construction, untyped payloads, leaked secrets, or unrelated churn.
4. Summarize implemented spec coverage, validation performed, and any unsupported or ambiguous OpenAPI features.
