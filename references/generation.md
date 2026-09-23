# Generating TypeScript with swagger-typescript-api

## Current Commands

```bash
# Generate TypeScript from Core API
pnpm api:core

# Generate TypeScript from Portal API
pnpm api:portal
```

These commands:
1. Fetch the latest OpenAPI spec from staging API
2. Generate TypeScript types using `swagger-typescript-api`
3. Output to `src/generated-api.ts` or `src/generated-api-portal.ts`

## Generation Options Explained

```bash
npx swagger-typescript-api generate \
  -p ./OpenAPI.json \              # Input OpenAPI spec
  -o ./src \                       # Output directory
  -n generated-api.ts \            # Output filename
  --axios \                        # Generate Axios HTTP client
  --generate-union-enums           # Generate union types for enums
```

Key flags:
- `--generate-union-enums`: Generates `type Status = "active" | "inactive"` instead of `enum Status { Active = "active" }`
- `--axios`: Creates Axios-based API client methods

