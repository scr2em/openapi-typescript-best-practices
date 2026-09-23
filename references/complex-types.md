# Complex Types & Patterns

Contents: oneOf unions · allOf composition · arrays · paginated responses · array-or-paginated endpoints · nested objects

## 1. Union Types (oneOf)

Use `oneOf` for types that can be one of several schemas. Define each variant as its own named schema and reference it. Inline variants would generate anonymous object types that you can't import or narrow to by name.

```json
{
  "components": {
    "schemas": {
      "CreditCardPaymentMethod": {
        "type": "object",
        "required": ["type", "cardNumber"],
        "properties": {
          "type": { "type": "string", "enum": ["credit_card"] },
          "cardNumber": { "type": "string" },
          "expiryDate": { "type": "string" }
        }
      },
      "BankAccountPaymentMethod": {
        "type": "object",
        "required": ["type", "accountNumber"],
        "properties": {
          "type": { "type": "string", "enum": ["bank_account"] },
          "accountNumber": { "type": "string" },
          "routingNumber": { "type": "string" }
        }
      },
      "PaymentMethod": {
        "oneOf": [
          { "$ref": "#/components/schemas/CreditCardPaymentMethod" },
          { "$ref": "#/components/schemas/BankAccountPaymentMethod" }
        ],
        "discriminator": {
          "propertyName": "type"
        }
      }
    }
  }
}
```

**Result:**
```typescript
export interface CreditCardPaymentMethod {
  type: "credit_card";
  cardNumber: string;
  expiryDate?: string;
}

export interface BankAccountPaymentMethod {
  type: "bank_account";
  accountNumber: string;
  routingNumber?: string;
}

export type PaymentMethod = CreditCardPaymentMethod | BankAccountPaymentMethod;
```

❌ **AVOID - Inline variants:**
```json
{
  "PaymentMethod": {
    "oneOf": [
      { "type": "object", "properties": { "type": { "type": "string", "enum": ["credit_card"] }, "cardNumber": { "type": "string" } } },
      { "type": "object", "properties": { "type": { "type": "string", "enum": ["bank_account"] }, "accountNumber": { "type": "string" } } }
    ]
  }
}
```
This generates `type PaymentMethod = { type: "credit_card"; ... } | { type: "bank_account"; ... }`. The variants have no names, so there's no `CreditCardPaymentMethod` type to import for function parameters or type guards.

## 2. Inheritance (allOf)

Use `allOf` for schema composition:

```json
{
  "components": {
    "schemas": {
      "BaseEntity": {
        "type": "object",
        "required": ["id", "createdAt"],
        "properties": {
          "id": { "type": "integer" },
          "createdAt": { "type": "string", "format": "date-time" },
          "updatedAt": { "type": "string", "format": "date-time" }
        }
      },
      "User": {
        "allOf": [
          { "$ref": "#/components/schemas/BaseEntity" },
          {
            "type": "object",
            "required": ["email"],
            "properties": {
              "email": { "type": "string", "format": "email" },
              "firstName": { "type": "string" },
              "lastName": { "type": "string" }
            }
          }
        ]
      }
    }
  }
}
```

**Result:**
```typescript
export interface BaseEntity {
  id: number;
  createdAt: string;
  updatedAt?: string;
}

export type User = BaseEntity & {
  email: string;
  firstName?: string;
  lastName?: string;
};
```

`swagger-typescript-api` generates `allOf` as an **intersection type** (`&`), not `interface ... extends`. In practice they behave the same: a `User` can be used anywhere a `BaseEntity` is expected. The difference shows up if a property conflicts between parts. `extends` would be a compile error, but an intersection silently types that property as `never`. So avoid redefining a base property in the extending schema.

## 3. Arrays and Collections

### Simple Arrays

✅ **Array of primitives:**
```json
{
  "tags": {
    "type": "array",
    "items": { "type": "string" },
    "description": "List of tags associated with the user"
  }
}
```

**Result:**
```typescript
/** List of tags associated with the user */
tags?: string[];
```

✅ **Array of objects:**
```json
{
  "users": {
    "type": "array",
    "items": { "$ref": "#/components/schemas/User" }
  }
}
```

**Result:**
```typescript
users?: User[];
```

### Paginated Responses

For endpoints that return paginated data, create a generic pagination wrapper:

✅ **BEST PRACTICE - Reusable pagination wrapper:**
```json
{
  "components": {
    "schemas": {
      "PaginatedUserResponse": {
        "type": "object",
        "required": ["data", "total", "count", "itemsPerPage"],
        "properties": {
          "data": {
            "type": "array",
            "items": { "$ref": "#/components/schemas/User" },
            "description": "Array of users for this page"
          },
          "total": {
            "type": "integer",
            "description": "Total number of items across all pages"
          },
          "count": {
            "type": "integer",
            "description": "Number of items in this page"
          },
          "itemsPerPage": {
            "type": "integer",
            "description": "Maximum items per page"
          }
        }
      }
    }
  }
}
```

**Result:**
```typescript
export interface PaginatedUserResponse {
  /** Array of users for this page */
  data: User[];
  /** Total number of items across all pages */
  total: number;
  /** Number of items in this page */
  count: number;
  /** Maximum items per page */
  itemsPerPage: number;
}
```

### Endpoints That Return Either Plain Array OR Paginated Response

When an endpoint can return either a plain array or a paginated response (e.g., based on query parameters), use `oneOf` in the response schema:

✅ **BEST PRACTICE - Flexible response with oneOf:**
```json
{
  "components": {
    "schemas": {
      "User": {
        "type": "object",
        "required": ["id", "name"],
        "properties": {
          "id": { "type": "integer" },
          "name": { "type": "string" }
        }
      },
      "PaginatedUserResponse": {
        "type": "object",
        "required": ["data", "total", "count", "itemsPerPage"],
        "properties": {
          "data": {
            "type": "array",
            "items": { "$ref": "#/components/schemas/User" }
          },
          "total": { "type": "integer" },
          "count": { "type": "integer" },
          "itemsPerPage": { "type": "integer" }
        }
      }
    },
    "responses": {
      "UserListResponse": {
        "description": "List of users (plain array or paginated)",
        "content": {
          "application/json": {
            "schema": {
              "oneOf": [
                {
                  "type": "array",
                  "items": { "$ref": "#/components/schemas/User" }
                },
                { "$ref": "#/components/schemas/PaginatedUserResponse" }
              ]
            }
          }
        }
      }
    }
  },
  "paths": {
    "/users": {
      "get": {
        "operationId": "getUsers",
        "parameters": [
          {
            "name": "paginate",
            "in": "query",
            "description": "Whether to return paginated results",
            "schema": { "type": "boolean", "default": false }
          }
        ],
        "responses": {
          "200": {
            "$ref": "#/components/responses/UserListResponse"
          }
        }
      }
    }
  }
}
```

**Result:**
```typescript
// Paginated response schema
export interface PaginatedUserResponse {
  data: User[];
  total: number;
  count: number;
  itemsPerPage: number;
}

// The response type will be generated as a union
export type UserListResponse = User[] | PaginatedUserResponse;

// Usage
const response = await api.getUsers({ paginate: true });

// Type guard to check which type was returned
if (Array.isArray(response)) {
  // Plain array
  console.log(`Got ${response.length} users`);
} else {
  // Paginated response
  console.log(`Got ${response.count} of ${response.total} users`);
}
```

**Benefits of this approach:**
- ✅ Single endpoint handles both use cases
- ✅ Type-safe - TypeScript knows both possible response shapes
- ✅ Reusable pagination structure across different entities
- ✅ Clear documentation of response variations

**💡 Pro Tip - Generic Pagination Schemas:**

For multiple entities, create entity-specific paginated responses:

```json
{
  "components": {
    "schemas": {
      "PaginatedUserResponse": {
        "type": "object",
        "required": ["data", "total", "count", "itemsPerPage"],
        "properties": {
          "data": {
            "type": "array",
            "items": { "$ref": "#/components/schemas/User" }
          },
          "total": { "type": "integer" },
          "count": { "type": "integer" },
          "itemsPerPage": { "type": "integer" }
        }
      },
      "PaginatedProductResponse": {
        "type": "object",
        "required": ["data", "total", "count", "itemsPerPage"],
        "properties": {
          "data": {
            "type": "array",
            "items": { "$ref": "#/components/schemas/Product" }
          },
          "total": { "type": "integer" },
          "count": { "type": "integer" },
          "itemsPerPage": { "type": "integer" }
        }
      }
    }
  }
}
```

This gives you clean, type-safe interfaces:
```typescript
export interface PaginatedUserResponse {
  data: User[];
  total: number;
  count: number;
  itemsPerPage: number;
}

export interface PaginatedProductResponse {
  data: Product[];
  total: number;
  count: number;
  itemsPerPage: number;
}
```

---

## 4. Nested Objects

✅ **Inline nested objects (for simple, non-reusable structures):**
```json
{
  "User": {
    "type": "object",
    "properties": {
      "id": { "type": "integer" },
      "address": {
        "type": "object",
        "properties": {
          "street": { "type": "string" },
          "city": { "type": "string" },
          "zipCode": { "type": "string" }
        }
      }
    }
  }
}
```

✅ **BETTER - Reference separate schema (for reusable structures):**
```json
{
  "components": {
    "schemas": {
      "Address": {
        "type": "object",
        "required": ["city"],
        "properties": {
          "street": { "type": "string" },
          "city": { "type": "string" },
          "zipCode": { "type": "string" }
        }
      },
      "User": {
        "type": "object",
        "properties": {
          "id": { "type": "integer" },
          "address": { "$ref": "#/components/schemas/Address" }
        }
      }
    }
  }
}
```
