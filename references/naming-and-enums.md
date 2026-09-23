# Naming Conventions & Enums

## Naming

### Schema Names

✅ **GOOD naming patterns:**
- **Entities & enums:** `User`, `UserSummary`, `UserStatus`
- **Request schemas:** `UserRequest`, `CreateUserRequest`, `UpdateUserRequest`
- **Response schemas:** `UserResponse`, `PaginatedUserResponse`, `ErrorResponse`
- **`components/requestBodies`:** `UserRequest`, `CreateUserRequest`, `UpdateUserRequest`
- **`components/responses`:** `UserResponse`, `NotFoundError`, `ValidationError`, `UnauthorizedError`
- **`components/headers`:** `X-Rate-Limit`, `X-Total-Count`, `X-Request-Id`, `X-Pagination-Offset`

❌ **AVOID:**
- `user` - Use PascalCase, not camelCase
- `UserDTO` - Avoid suffixes like DTO, they're redundant
- `get_user` - Avoid snake_case
- `UserModel` - Avoid Model suffix
- `UserCreate`, `UserUpdate`, `UserProfileUpdate` - Use the `Request` suffix (`CreateUserRequest`, `UpdateUserRequest`) so request schemas are recognizable at a glance
- `user-request`, `user_response` - Use PascalCase `Request`/`Response` suffixes, not kebab or snake case

### Property Names

✅ **Use camelCase for properties:**
```json
{
  "properties": {
    "firstName": { "type": "string" },
    "phoneNumber": { "type": "string" },
    "createdAt": { "type": "string" }
  }
}
```

❌ **AVOID snake_case** (unless your backend consistently uses it):
```json
{
  "properties": {
    "first_name": { "type": "string" },
    "phone_number": { "type": "string" }
  }
}
```

## Enums

### 1. Define Enums in `components/schemas`

✅ **BEST PRACTICE** - Named enum schema:
```json
{
  "components": {
    "schemas": {
      "UserStatus": {
        "type": "string",
        "enum": ["active", "inactive", "pending", "suspended"],
        "description": "Current status of the user account"
      }
    }
  }
}
```

**Result with `--generate-union-enums`:**
```typescript
/** Current status of the user account */
export type UserStatus = "active" | "inactive" | "pending" | "suspended";
```

### 2. Use String Enums for Readability

✅ **GOOD** - String values:
```json
{
  "OrderStatus": {
    "type": "string",
    "enum": ["pending", "processing", "shipped", "delivered", "cancelled"]
  }
}
```

❌ **AVOID** - Numeric enums (less readable):
```json
{
  "OrderStatus": {
    "type": "integer",
    "enum": [0, 1, 2, 3, 4]
  }
}
```

### 3. Enum Naming Conventions

✅ **Use UPPER_SNAKE_CASE for constant-like values:**
```json
{
  "ExportType": {
    "type": "string",
    "enum": [
      "EXPORT_DC_PROFILES",
      "EXPORT_MMP_MEETINGS",
      "EXPORT_DC_USER_ACTIVITIES"
    ]
  }
}
```

✅ **Use lowercase for state/status values:**
```json
{
  "PaymentStatus": {
    "type": "string",
    "enum": ["pending", "succeeded", "failed", "refunded"]
  }
}
```

