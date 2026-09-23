# Common Pitfalls

## ❌ 1. Missing `operationId`

Without `operationId`, generated method names will be generic:

```json
{
  "paths": {
    "/users": {
      "get": {
        "summary": "Get all users"
        // ❌ Missing operationId
      }
    }
  }
}
```

✅ **ALWAYS add `operationId`:**
```json
{
  "paths": {
    "/users": {
      "get": {
        "operationId": "getUsers",
        "summary": "Get all users"
      }
    }
  }
}
```

## ❌ 2. Inconsistent Property Casing

```json
{
  "properties": {
    "firstName": { "type": "string" },
    "last_name": { "type": "string" },  // ❌ Inconsistent
    "PhoneNumber": { "type": "string" }  // ❌ Wrong case
  }
}
```

✅ **Be consistent - use camelCase:**
```json
{
  "properties": {
    "firstName": { "type": "string" },
    "lastName": { "type": "string" },
    "phoneNumber": { "type": "string" }
  }
}
```

## ❌ 3. Generic Schema Names

```json
{
  "Request": { ... },    // ❌ Too generic
  "Response": { ... },   // ❌ Too generic
  "Data": { ... }        // ❌ Too generic
}
```

✅ **Use specific, descriptive names:**
```json
{
  "CreateUserRequest": { ... },
  "UserResponse": { ... },
  "UserAnalyticsData": { ... }
}
```

## ❌ 4. Overusing `additionalProperties`

```json
{
  "User": {
    "type": "object",
    "properties": {
      "id": { "type": "integer" }
    },
    "additionalProperties": true  // ❌ Defeats type safety
  }
}
```

**Result:**
```typescript
export interface User {
  id?: number;
  [key: string]: any;  // ❌ Any property allowed
}
```

✅ **Define all properties explicitly:**
```json
{
  "User": {
    "type": "object",
    "properties": {
      "id": { "type": "integer" },
      "metadata": {
        "type": "object",
        "additionalProperties": { "type": "string" }  // ✅ Typed map
      }
    }
  }
}
```

## ❌ 5. Missing Response Schemas

```json
{
  "responses": {
    "200": {
      "description": "Success"
      // ❌ No schema defined
    }
  }
}
```

✅ **ALWAYS define response schemas:**
```json
{
  "responses": {
    "200": {
      "description": "User retrieved successfully",
      "content": {
        "application/json": {
          "schema": {
            "$ref": "#/components/schemas/User"
          }
        }
      }
    }
  }
}
```

