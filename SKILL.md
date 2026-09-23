---
name: openapi-typescript-best-practices
description: Best practices for writing, reviewing, and fixing OpenAPI 3.0 specs so they generate clean, accurate TypeScript types and API clients with swagger-typescript-api. Use this skill whenever the user is creating or editing an OpenAPI/Swagger spec (JSON or YAML), designing API schemas, request bodies, responses, headers, or enums, reviewing an API contract, deciding required vs optional or nullable fields, modeling unions/inheritance/pagination, or debugging why generated TypeScript types look wrong (e.g. anonymous inline types, `any`, missing `| null`, generic method names) — even if they don't mention "best practices" or TypeScript explicitly.
---

# OpenAPI → TypeScript Best Practices

The OpenAPI spec is the single source of truth for the frontend/backend contract. Types and an Axios client are generated from it with [`swagger-typescript-api`](https://github.com/acacode/swagger-typescript-api), so every choice in the spec shows up directly in the TypeScript developers use. The rules below exist because each one produces better generated code: named types instead of anonymous inline shapes, accurate optionality and nullability, JSDoc from descriptions, and readable method names.

## How to use this skill

- **Writing or editing a spec**: follow the core rules below, then read the reference file for the area you're touching.
- **Reviewing a spec**: walk the checklist at the bottom and report each violation with the location, why it matters for the generated TypeScript, and the corrected snippet.
- **Debugging generated types**: map the symptom to its cause (table below), then fix the spec — never hand-edit generated files like `src/generated-api.ts`, they're overwritten on the next generation.

Match the spec's existing format (JSON vs YAML) and conventions. If the project has already made a consistent choice that differs from these rules (e.g. snake_case properties everywhere), keep it consistent rather than mixing styles — consistency matters more than the specific convention.

## Core rules

1. **Put reusable definitions in `components`, not inline in `paths`.** Schemas and enums go in `components/schemas`, shared request bodies in `components/requestBodies`, shared responses (especially errors like `NotFoundError`, `ValidationError`, `UnauthorizedError`) in `components/responses`. Inline definitions produce anonymous types that can't be imported, reused, or discovered. Acceptable inline cases: a one-off success response wrapping a `$ref` or array of `$ref`, simple primitive query parameters, and small non-reusable nested objects.
2. **Headers: `components/headers` when shared, inline when endpoint-specific.** Rate-limit, pagination, and auth headers are shared; a job ID returned by one endpoint can be inline.
3. **Every operation has an `operationId`** (camelCase verb + noun: `getUsers`, `createProduct`). It becomes the generated client method name.
4. **Every response has a schema** (except genuinely empty ones like 204). A response without `content` generates `void`/`any`.
5. **Add `description` to schemas and properties** — they become JSDoc comments. For nullable fields, describe what `null` means.
6. **Use `format`** (`email`, `uri`, `date`, `date-time`, `uuid`, `binary`, `int32`, `int64`, `float`) and validation constraints (`minimum`, `maxLength`, `pattern`, …).
7. **Separate request and response schemas** when they differ: `UserRequest` / `UserResponse` (or `CreateUserRequest`, `UpdateUserRequest`). Responses include server-set fields like `id` and `createdAt`; requests don't.
8. **Required and nullable are independent.** Default to required + non-nullable for mandatory data and optional + non-nullable for optional data. Only add `nullable: true` when `null` carries meaning. Never make fields nullable "to be safe" — it forces null checks everywhere and defeats type safety.
9. **Naming**: PascalCase specific schema names (`CreateUserRequest`, not `Request`, `Data`, or `UserCreate`; no `DTO`/`Model` suffixes); camelCase properties; string enums (not numeric), with lowercase values for states/statuses and UPPER_SNAKE_CASE for constant-like values.
10. **Avoid `additionalProperties: true`** — it generates `[key: string]: any`. For maps, use a typed `additionalProperties: { "type": "string" }`.
11. **Discriminated unions**: `oneOf` of `$ref`s (each variant is a named schema, not inline) plus `discriminator.propertyName`, where each variant has a single-value enum on the discriminator property. This generates `type PaymentMethod = CreditCardPaymentMethod | BankAccountPaymentMethod`, which narrows in a `switch`.
12. **Composition with `allOf`** generates an intersection type (`type User = BaseEntity & {...}`), not `interface ... extends`. Don't redefine a base property in the extending part: a conflicting property silently becomes `never` instead of raising an error.

### Required/nullable quick map

| OpenAPI | TypeScript | Use for |
|---|---|---|
| in `required`, not nullable | `field: string` | ids, emails, timestamps — most fields |
| in `required`, `nullable: true` | `field: string \| null` | always present but may be null (`managerId`, `parentId`, `deletedAt`) |
| not in `required`, not nullable | `field?: string` | ⭐ default for optional fields |
| not in `required`, `nullable: true` | `field?: string \| null` | only when omitted ≠ null (e.g. PATCH where `null` clears a value) |

`nullable` is OpenAPI 3.0 syntax. If the spec is 3.1, use `"type": ["string", "null"]` instead.

## Symptom → cause

| Generated TypeScript symptom | Likely spec cause |
|---|---|
| Anonymous/inline object types, duplicated shapes | Schema, enum, or response defined inline in `paths` |
| Method names like `usersList` / `usersDetail2` | Missing `operationId` |
| `any`, `void`, or `[key: string]: any` | Missing response schema, or `additionalProperties: true` |
| Field is `?:` but should always be there | Field missing from the `required` array |
| Missing `\| null` or unwanted `\| null` | `nullable` wrong for the field's semantics |
| TS `enum` instead of a string-literal union | Generation run without `--generate-union-enums` |
| No JSDoc on types | Missing `description` |

## Reference files

Read the one that matches the task; each has full correct/incorrect examples with the generated TypeScript.

- `references/schema-design.md` — components (schemas, enums, requestBodies, responses), headers, descriptions, formats, Request/Response separation
- `references/required-nullable.md` — the four combinations in depth, DO/DON'T, PATCH-style update example, decision flowchart
- `references/naming-and-enums.md` — schema/property naming, enum definitions and value casing
- `references/complex-types.md` — `oneOf`, `allOf`, arrays, paginated responses (`PaginatedXResponse` with `data`/`total`/`count`/`itemsPerPage`), endpoints returning array-or-paginated, nested objects
- `references/pitfalls.md` — common mistakes with before/after fixes
- `references/examples.md` — complete CRUD resource and discriminated-union examples
- `references/generation.md` — `swagger-typescript-api` command and flags (`--axios`, `--generate-union-enums`)

## Generation

Check `package.json` for the project's generation scripts (e.g. `pnpm api:core`, `pnpm api:portal`) before running the CLI directly. After spec changes, regenerate and check that the output has the expected named types, optionality, and method names.

## Review checklist

- [ ] Schemas and enums in `components/schemas`, not inline
- [ ] Specific PascalCase schema names; camelCase properties
- [ ] `required` array lists all mandatory fields
- [ ] `nullable` only where null has meaning, with the meaning described
- [ ] Descriptions on schemas and properties
- [ ] Appropriate `format` for strings and numbers; validation constraints where useful
- [ ] Separate `…Request` / `…Response` schemas when shapes differ
- [ ] Shared request bodies in `components/requestBodies`
- [ ] Shared responses (especially errors) in `components/responses`
- [ ] Shared headers in `components/headers`; endpoint-specific headers may be inline
- [ ] `operationId` on every operation
- [ ] Response schema defined for every status code that returns a body
- [ ] String enums; no `additionalProperties: true`
- [ ] Unions use `oneOf` + `discriminator`; pagination uses the `PaginatedXResponse` shape
