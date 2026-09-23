# OpenAPI to TypeScript Best Practices

A [Claude skill](https://docs.claude.com/en/docs/claude-code/skills) and guide for writing OpenAPI 3.0 specs that generate clean, accurate TypeScript types and API clients with [`swagger-typescript-api`](https://github.com/acacode/swagger-typescript-api).

Claude loads the skill automatically when you create, edit, review, or debug an OpenAPI spec, or when generated TypeScript types look wrong.

## Why This Approach?

✅ **Single Source of Truth** - API contracts defined once in OpenAPI

✅ **Type Safety** - Automatic TypeScript types prevent runtime errors

✅ **Developer Experience** - Auto-complete and IntelliSense in IDEs

✅ **Documentation** - OpenAPI spec serves as living documentation

✅ **Consistency** - Frontend and backend share the same contract

## Installing the skill

**Personal** (available in all your projects):

```bash
git clone https://github.com/scr2em/openapi-typescript-best-practices.git ~/.claude/skills/openapi-typescript-best-practices
```

**Project** (shared with your team via the repo, run from the project root):

```bash
git submodule add https://github.com/scr2em/openapi-typescript-best-practices.git .claude/skills/openapi-typescript-best-practices
```

To update later:

```bash
# Personal
git -C ~/.claude/skills/openapi-typescript-best-practices pull

# Project
git submodule update --remote .claude/skills/openapi-typescript-best-practices
```

Restart Claude Code after installing. You can confirm the skill is loaded by typing `/openapi-typescript-best-practices` or asking Claude to review an OpenAPI spec.

## Contents

| File | Covers |
|---|---|
| [`SKILL.md`](SKILL.md) | Core rules, required/nullable quick map, symptom → cause table, review checklist |
| [`references/schema-design.md`](references/schema-design.md) | `components` for schemas, enums, request bodies, responses; headers; descriptions; formats; Request/Response schemas |
| [`references/required-nullable.md`](references/required-nullable.md) | Required vs optional & nullable vs non-nullable, with a decision flowchart |
| [`references/naming-and-enums.md`](references/naming-and-enums.md) | Schema/property naming, enum conventions |
| [`references/complex-types.md`](references/complex-types.md) | `oneOf`, `allOf`, arrays, pagination, nested objects |
| [`references/pitfalls.md`](references/pitfalls.md) | Common mistakes and fixes |
| [`references/examples.md`](references/examples.md) | Complete CRUD and discriminated-union examples |
| [`references/generation.md`](references/generation.md) | `swagger-typescript-api` commands and flags |

## Additional Resources

- [OpenAPI 3.0 Specification](https://swagger.io/specification/)
- [swagger-typescript-api Documentation](https://github.com/acacode/swagger-typescript-api)
- [JSON Schema Validation](https://json-schema.org/understanding-json-schema/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)

---

**Maintained by:** Frontend Team
