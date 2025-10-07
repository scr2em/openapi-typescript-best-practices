# OpenAPI to TypeScript Best Practices Guide

## Table of Contents
1. [Overview](#overview)
2. [Setup & Workflow](#setup--workflow)
3. [Schema Design Best Practices](#schema-design-best-practices)
4. [Required vs Optional & Nullable vs Non-Nullable](#required-vs-optional--nullable-vs-non-nullable)
5. [Naming Conventions](#naming-conventions)
6. [Working with Enums](#working-with-enums)
7. [Complex Types & Patterns](#complex-types--patterns)
8. [Common Pitfalls](#common-pitfalls)
9. [Examples](#examples)
10. [Testing Generated Types](#testing-generated-types)

---

## Overview

This project uses [`swagger-typescript-api`](https://github.com/acacode/swagger-typescript-api) to automatically generate TypeScript types, interfaces, and API clients from OpenAPI 3.0 specifications.

### Why This Approach?

✅ **Single Source of Truth** - API contracts defined once in OpenAPI

✅ **Type Safety** - Automatic TypeScript types prevent runtime errors

✅ **Developer Experience** - Auto-complete and IntelliSense in IDEs

✅ **Documentation** - OpenAPI spec serves as living documentation

✅ **Consistency** - Frontend and backend share the same contract

---

## Setup & Workflow

### Current Commands

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

### Generation Options Explained

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

---

## Schema Design Best Practices

### 1. Always Use `components` for Definitions

**⚠️ CRITICAL RULE: Never define schemas, enums, request bodies, or responses inline in your paths.**

Define all reusable types in the `components` section rather than inline. 

#### Use `components/schemas` for Data Models and Enums

Define your data structures and enums once and reference them everywhere:

```json
{
  "components": {
    "schemas": {
      "User": {
        "type": "object",
        "required": ["id", "name", "status"],
        "properties": {
          "id": { "type": "integer" },
          "name": { "type": "string" },
          "status": { "$ref": "#/components/schemas/UserStatus" }
        }
      },
      "UserStatus": {
        "type": "string",
        "enum": ["active", "inactive", "pending", "suspended"],
        "description": "User account status"
      },
      "ErrorResponse": {
        "type": "object",
        "required": ["error", "message"],
        "properties": {
          "error": { "type": "string" },
          "message": { "type": "string" }
        }
      }
    }
  },
  "paths": {
    "/users/{id}": {
      "get": {
        "responses": {
          "200": {
            "content": {
              "application/json": {
                "schema": { "$ref": "#/components/schemas/User" }
              }
            }
          }
        }
      }
    }
  }
}
```

❌ **WRONG - Inline definitions:**
```json
{
  "paths": {
    "/users/{id}": {
      "get": {
        "responses": {
          "200": {
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "id": { "type": "integer" },
                    "name": { "type": "string" },
                    "status": {
                      "type": "string",
                      "enum": ["active", "inactive", "pending"]
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```
**Problems with inline definitions:**
- ❌ Cannot reuse the User schema or status enum elsewhere
- ❌ No named TypeScript interface/type generated
- ❌ Harder to maintain and keep consistent
- ❌ Poor documentation and discoverability

#### Use `components/requestBodies` for Request Bodies

✅ **CORRECT - Reusable request body:**
```json
{
  "components": {
    "requestBodies": {
      "UserRequest": {
        "description": "User data for creation or update",
        "required": true,
        "content": {
          "application/json": {
            "schema": { "$ref": "#/components/schemas/UserRequest" }
          }
        }
      }
    }
  },
  "paths": {
    "/users": {
      "post": {
        "requestBody": { "$ref": "#/components/requestBodies/UserRequest" }
      }
    },
    "/users/{id}": {
      "put": {
        "requestBody": { "$ref": "#/components/requestBodies/UserRequest" }
      }
    }
  }
}
```

❌ **WRONG - Inline request body (duplicated):**
```json
{
  "paths": {
    "/users": {
      "post": {
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": { "$ref": "#/components/schemas/UserRequest" }
            }
          }
        }
      }
    },
    "/users/{id}": {
      "put": {
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": { "$ref": "#/components/schemas/UserRequest" }
            }
          }
        }
      }
    }
  }
}
```
**Problems:** Same request body definition duplicated - changes require updating multiple places.

#### Use `components/responses` for Responses

Especially useful for standardizing error responses across all endpoints:

✅ **CORRECT - Reusable responses:**
```json
{
  "components": {
    "responses": {
      "NotFoundError": {
        "description": "Resource not found",
        "content": {
          "application/json": {
            "schema": { "$ref": "#/components/schemas/ErrorResponse" }
          }
        }
      },
      "ValidationError": {
        "description": "Validation error",
        "content": {
          "application/json": {
            "schema": { "$ref": "#/components/schemas/ErrorResponse" }
          }
        }
      },
      "UnauthorizedError": {
        "description": "Unauthorized access",
        "content": {
          "application/json": {
            "schema": { "$ref": "#/components/schemas/ErrorResponse" }
          }
        }
      }
    }
  },
  "paths": {
    "/users/{id}": {
      "get": {
        "responses": {
          "200": {
            "description": "User retrieved successfully",
            "content": {
              "application/json": {
                "schema": { "$ref": "#/components/schemas/UserResponse" }
              }
            }
          },
          "404": { "$ref": "#/components/responses/NotFoundError" },
          "401": { "$ref": "#/components/responses/UnauthorizedError" }
        }
      }
    },
    "/products/{id}": {
      "get": {
        "responses": {
          "200": {
            "description": "Product retrieved successfully",
            "content": {
              "application/json": {
                "schema": { "$ref": "#/components/schemas/ProductResponse" }
              }
            }
          },
          "404": { "$ref": "#/components/responses/NotFoundError" },
          "401": { "$ref": "#/components/responses/UnauthorizedError" }
        }
      }
    }
  }
}
```

❌ **WRONG - Inline error responses (duplicated across endpoints):**
```json
{
  "paths": {
    "/users/{id}": {
      "get": {
        "responses": {
          "404": {
            "description": "Resource not found",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "error": { "type": "string" }
                  }
                }
              }
            }
          }
        }
      }
    },
    "/products/{id}": {
      "get": {
        "responses": {
          "404": {
            "description": "Resource not found",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "error": { "type": "string" }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```


**When you can inline (rare exceptions):**
- ✅ Unique, endpoint-specific responses that will never be reused (e.g., a specific 200 success response)
- ✅ Simple query parameters with primitive types

---

### 2. Use `components/headers` for Common Headers, Inline for Endpoint-Specific Ones

Headers follow a different pattern than schemas - inline them when they're specific to an endpoint, but use `components/headers` when they're shared across multiple endpoints.

#### Use `components/headers` for Common Headers

Define reusable headers like authentication tokens, rate limiting, or pagination metadata:

✅ **CORRECT - Reusable headers:**
```json
{
  "components": {
    "headers": {
      "X-Rate-Limit": {
        "description": "Number of requests allowed per hour",
        "schema": {
          "type": "integer"
        }
      },
      "X-Rate-Limit-Remaining": {
        "description": "Number of requests remaining in the current period",
        "schema": {
          "type": "integer"
        }
      },
      "X-Total-Count": {
        "description": "Total number of items available",
        "schema": {
          "type": "integer"
        }
      }
    }
  },
  "paths": {
    "/users": {
      "get": {
        "responses": {
          "200": {
            "description": "List of users",
            "headers": {
              "X-Rate-Limit": {
                "$ref": "#/components/headers/X-Rate-Limit"
              },
              "X-Rate-Limit-Remaining": {
                "$ref": "#/components/headers/X-Rate-Limit-Remaining"
              },
              "X-Total-Count": {
                "$ref": "#/components/headers/X-Total-Count"
              }
            },
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": { "$ref": "#/components/schemas/User" }
                }
              }
            }
          }
        }
      }
    },
    "/products": {
      "get": {
        "responses": {
          "200": {
            "description": "List of products",
            "headers": {
              "X-Rate-Limit": {
                "$ref": "#/components/headers/X-Rate-Limit"
              },
              "X-Rate-Limit-Remaining": {
                "$ref": "#/components/headers/X-Rate-Limit-Remaining"
              },
              "X-Total-Count": {
                "$ref": "#/components/headers/X-Total-Count"
              }
            },
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": { "$ref": "#/components/schemas/Product" }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

#### Inline Headers for Endpoint-Specific Ones

When a header is unique to a specific endpoint, it's acceptable to define it inline:

✅ **CORRECT - Endpoint-specific header:**
```json
{
  "paths": {
    "/export/data": {
      "post": {
        "responses": {
          "202": {
            "description": "Export initiated",
            "headers": {
              "X-Export-Job-Id": {
                "description": "Unique identifier for this export job",
                "schema": {
                  "type": "string",
                  "format": "uuid"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

**Decision Guide - Headers:**
- **Use `components/headers`** for:
  - ✅ Rate limiting headers (used across all endpoints)
  - ✅ Pagination headers (used in list endpoints)
  - ✅ Common authentication/authorization headers
  - ✅ Standard API metadata headers
  
- **Inline headers** for:
  - ✅ Job/task-specific identifiers
  - ✅ Endpoint-unique tracking headers
  - ✅ Headers that will never be reused elsewhere

---

### 3. Add Descriptions for Better Documentation

✅ **ALWAYS include descriptions** - They become JSDoc comments:

```json
{
  "User": {
    "type": "object",
    "description": "Represents a registered user in the system",
    "required": ["id", "email"],
    "properties": {
      "id": {
        "type": "integer",
        "description": "Unique identifier for the user"
      },
      "email": {
        "type": "string",
        "format": "email",
        "description": "User's email address (must be unique)"
      },
      "createdAt": {
        "type": "string",
        "format": "date-time",
        "description": "ISO 8601 timestamp when user was created"
      }
    }
  }
}
```

**Result:**
```typescript
/**
 * Represents a registered user in the system
 */
export interface User {
  /** Unique identifier for the user */
  id: number;
  /** User's email address (must be unique) */
  email: string;
  /** ISO 8601 timestamp when user was created */
  createdAt?: string;
}
```

### 4. Use `format` for Better Type Hints

Leverage OpenAPI formats to provide semantic meaning:

```json
{
  "properties": {
    "email": {
      "type": "string",
      "format": "email"
    },
    "website": {
      "type": "string",
      "format": "uri"
    },
    "createdAt": {
      "type": "string",
      "format": "date-time"
    },
    "age": {
      "type": "integer",
      "format": "int32",
      "minimum": 0,
      "maximum": 150
    }
  }
}
```

**Common formats:**
- `email` - Email addresses
- `uri` / `url` - URLs
- `date` - Date only (YYYY-MM-DD)
- `date-time` - ISO 8601 timestamp
- `uuid` - UUID strings
- `binary` - Binary data
- `int32` / `int64` - Integer size hints

### 5. Define Separate Response/Request Schemas

Different operations often need different schemas. Use suffixes like `Response` and `Request`:

```json
{
  "components": {
    "schemas": {
      "UserResponse": {
        "type": "object",
        "required": ["id", "email", "createdAt"],
        "properties": {
          "id": { "type": "integer" },
          "email": { "type": "string", "format": "email" },
          "createdAt": { "type": "string", "format": "date-time" },
          "updatedAt": { "type": "string", "format": "date-time" }
        }
      },
      "UserRequest": {
        "type": "object",
        "required": ["email"],
        "properties": {
          "email": { "type": "string", "format": "email" },
          "firstName": { "type": "string" },
          "lastName": { "type": "string" }
        }
      }
    }
  }
}
```

**Result:**
```typescript
export interface UserResponse {
  id: number;
  email: string;
  createdAt: string;
  updatedAt?: string;
}

export interface UserRequest {
  email: string;
  firstName?: string;
  lastName?: string;
}
```

---

## Required vs Optional & Nullable vs Non-Nullable

Understanding the difference between **required/optional** and **nullable/non-nullable** is crucial for generating accurate TypeScript types. These are two independent concepts that can be combined in four ways.

### The Four Combinations

| Combination | OpenAPI | TypeScript | Meaning |
|------------|---------|------------|---------|
| **Required + Non-nullable** | `required: ["field"]`<br/>`nullable: false` (default) | `field: string` | Must be provided, cannot be null |
| **Required + Nullable** | `required: ["field"]`<br/>`nullable: true` | `field: string \| null` | Must be provided, can be null |
| **Optional + Non-nullable** | Not in `required`<br/>`nullable: false` (default) | `field?: string` | May be omitted, but if provided cannot be null |
| **Optional + Nullable** | Not in `required`<br/>`nullable: true` | `field?: string \| null` | May be omitted or null |

---

### 1. Required + Non-Nullable (Most Common)

The field **MUST** be present and **CANNOT** be null.

**OpenAPI Schema:**
```json
{
  "User": {
    "type": "object",
    "required": ["id", "email"],
    "properties": {
      "id": {
        "type": "integer"
      },
      "email": {
        "type": "string",
        "format": "email"
      }
    }
  }
}
```

**Generated TypeScript:**
```typescript
export interface User {
  id: number;        // ✅ Must be provided, cannot be null
  email: string;     // ✅ Must be provided, cannot be null
}
```

**Usage:**
```typescript
// ✅ Valid
const user: User = {
  id: 1,
  email: "user@example.com"
};

// ❌ TypeScript Error: Property 'email' is missing
const user: User = {
  id: 1
};

// ❌ TypeScript Error: Type 'null' is not assignable to type 'string'
const user: User = {
  id: 1,
  email: null
};
```

**When to use:** For fields that are always required and should never be null (IDs, emails, names, timestamps, etc.).

---

### 2. Required + Nullable

The field **MUST** be present but **CAN** be null.

**OpenAPI Schema:**
```json
{
  "Employee": {
    "type": "object",
    "required": ["id", "name", "managerId"],
    "properties": {
      "id": {
        "type": "integer"
      },
      "name": {
        "type": "string"
      },
      "managerId": {
        "type": "integer",
        "nullable": true,
        "description": "ID of the employee's manager, null if they have no manager (e.g., CEO)"
      }
    }
  }
}
```

**Generated TypeScript:**
```typescript
export interface Employee {
  id: number;
  name: string;
  /** ID of the employee's manager, null if they have no manager (e.g., CEO) */
  managerId: number | null;  // ✅ Must be provided, can be null
}
```

**Usage:**
```typescript
// ✅ Valid - manager exists
const employee: Employee = {
  id: 1,
  name: "John Doe",
  managerId: 5
};

// ✅ Valid - no manager (CEO)
const ceo: Employee = {
  id: 2,
  name: "Jane CEO",
  managerId: null
};

// ❌ TypeScript Error: Property 'managerId' is missing
const invalid: Employee = {
  id: 3,
  name: "Bob"
  // managerId must be present (even if null)
};
```

**When to use:**
- When you need to distinguish between "not set" vs "explicitly null"
- For nullable foreign keys (like managerId above)
- When the API always returns the field, but its value may be null
- For fields where null has specific business meaning (e.g., "explicitly cleared" vs "never set")

---

### 3. Optional + Non-Nullable (Very Common)

The field **MAY** be omitted, but if provided **CANNOT** be null.

**OpenAPI Schema:**
```json
{
  "User": {
    "type": "object",
    "required": ["id", "email"],
    "properties": {
      "id": {
        "type": "integer"
      },
      "email": {
        "type": "string",
        "format": "email"
      },
      "phoneNumber": {
        "type": "string",
        "description": "User's phone number (optional)"
      },
      "bio": {
        "type": "string",
        "description": "User biography (optional)"
      }
    }
  }
}
```

**Generated TypeScript:**
```typescript
export interface User {
  id: number;
  email: string;
  /** User's phone number (optional) */
  phoneNumber?: string;  // ✅ Can be omitted, but if provided must be string
  /** User biography (optional) */
  bio?: string;          // ✅ Can be omitted, but if provided must be string
}
```

**Usage:**
```typescript
// ✅ Valid - all fields provided
const user1: User = {
  id: 1,
  email: "user@example.com",
  phoneNumber: "+1234567890",
  bio: "Software engineer"
};

// ✅ Valid - optional fields omitted
const user2: User = {
  id: 2,
  email: "user2@example.com"
};

// ✅ Valid - some optional fields provided
const user3: User = {
  id: 3,
  email: "user3@example.com",
  bio: "Designer"
};

// ❌ TypeScript Error: Type 'null' is not assignable to type 'string | undefined'
const user4: User = {
  id: 4,
  email: "user4@example.com",
  phoneNumber: null  // ❌ Cannot be null, must be string or omitted
};

// ✅ To "unset" an optional field, use undefined or omit it
const user5: User = {
  id: 5,
  email: "user5@example.com",
  phoneNumber: undefined  // ✅ Same as omitting it
};
```

**When to use:**
- For most optional fields (profile info, preferences, metadata)
- When "not provided" and "null" mean the same thing
- This is the **RECOMMENDED default** for optional fields

---

### 4. Optional + Nullable

The field **MAY** be omitted, and if provided **CAN** be null.

**OpenAPI Schema:**
```json
{
  "Article": {
    "type": "object",
    "required": ["id", "title"],
    "properties": {
      "id": {
        "type": "integer"
      },
      "title": {
        "type": "string"
      },
      "publishedAt": {
        "type": "string",
        "format": "date-time",
        "nullable": true,
        "description": "When the article was published. Null if unpublished. Undefined if draft not ready for publishing."
      },
      "summary": {
        "type": "string",
        "nullable": true,
        "description": "Article summary. Can be explicitly null if author chose not to provide one."
      }
    }
  }
}
```

**Generated TypeScript:**
```typescript
export interface Article {
  id: number;
  title: string;
  /** When the article was published. Null if unpublished. Undefined if draft not ready for publishing. */
  publishedAt?: string | null;
  /** Article summary. Can be explicitly null if author chose not to provide one. */
  summary?: string | null;
}
```

**Usage:**
```typescript
// ✅ Valid - published article with summary
const article1: Article = {
  id: 1,
  title: "TypeScript Tips",
  publishedAt: "2025-10-06T12:00:00Z",
  summary: "Learn TypeScript best practices"
};

// ✅ Valid - unpublished article (publishedAt explicitly null)
const article2: Article = {
  id: 2,
  title: "Future Post",
  publishedAt: null,  // Explicitly unpublished
  summary: "Coming soon"
};

// ✅ Valid - draft not ready (publishedAt omitted)
const article3: Article = {
  id: 3,
  title: "Work in Progress",
  // publishedAt omitted - draft not ready for publishing decision
  summary: null  // Author explicitly chose not to provide summary
};

// ✅ Valid - minimal article
const article4: Article = {
  id: 4,
  title: "Minimal Post"
  // All optional fields omitted
};
```

**When to use:**
- When you need **three states**: not provided, explicitly null, or has value
- For fields with complex state logic (like the publishedAt example)
- When "undefined" and "null" have different business meanings

⚠️ **Warning:** This combination adds complexity. Only use when you genuinely need three states.

---

### Comparison Table with Examples

| Field State | Required Non-nullable | Required Nullable | Optional Non-nullable | Optional Nullable |
|-------------|----------------------|-------------------|-----------------------|-------------------|
| **OpenAPI** | `required: ["field"]` | `required: ["field"]`<br/>`nullable: true` | (not in required) | (not in required)<br/>`nullable: true` |
| **TypeScript** | `field: string` | `field: string \| null` | `field?: string` | `field?: string \| null` |
| **Must provide** | ✅ Yes | ✅ Yes | ❌ No | ❌ No |
| **Can be null** | ❌ No | ✅ Yes | ❌ No | ✅ Yes |
| **Can be undefined** | ❌ No | ❌ No | ✅ Yes | ✅ Yes |
| **Valid: value** | ✅ | ✅ | ✅ | ✅ |
| **Valid: null** | ❌ | ✅ | ❌ | ✅ |
| **Valid: undefined/omitted** | ❌ | ❌ | ✅ | ✅ |

---

### Best Practices & Recommendations

#### ✅ **DO:**

1. **Use Required + Non-nullable by default** for mandatory data:
   ```json
   {
     "required": ["id", "email", "createdAt"],
     "properties": {
       "id": { "type": "integer" },
       "email": { "type": "string" }
     }
   }
   ```

2. **Use Optional + Non-nullable for most optional fields**:
   ```json
   {
     "properties": {
       "phoneNumber": { "type": "string" },
       "bio": { "type": "string" }
     }
   }
   ```

3. **Use Required + Nullable when the field is always present but may be null**:
   ```json
   {
     "required": ["parentId"],
     "properties": {
       "parentId": {
         "type": "integer",
         "nullable": true,
         "description": "Null for root-level items"
       }
     }
   }
   ```

4. **Document why a field is nullable** in the description.

#### ❌ **DON'T:**

1. **Don't use Optional + Nullable unless you need three states**:
   ```json
   // ❌ Avoid unless necessary
   {
     "properties": {
       "phoneNumber": {
         "type": "string",
         "nullable": true  // ❌ Why nullable? Just make it optional
       }
     }
   }
   ```

2. **Don't make everything nullable to "be safe"** - it defeats type safety:
   ```json
   // ❌ Bad practice
   {
     "required": ["id", "name"],
     "properties": {
       "id": { "type": "integer", "nullable": true },      // ❌ IDs shouldn't be null
       "name": { "type": "string", "nullable": true }      // ❌ Names shouldn't be null
     }
   }
   ```

3. **Don't confuse "optional in request" with "nullable in response"**:
   - Create separate schemas for requests and responses
   - Use `Request` suffix for request schemas, `Response` for responses

---

### Real-World Example: User Profile Update

Here's a complete example showing when to use each combination:

**OpenAPI Schema:**
```json
{
  "UserProfileUpdate": {
    "type": "object",
    "description": "User profile update request",
    "properties": {
      "firstName": {
        "type": "string",
        "description": "Optional: update first name"
      },
      "lastName": {
        "type": "string",
        "description": "Optional: update last name"
      },
      "bio": {
        "type": "string",
        "nullable": true,
        "description": "Optional: update bio. Pass null to explicitly clear it."
      },
      "phoneNumber": {
        "type": "string",
        "nullable": true,
        "description": "Optional: update phone. Pass null to remove it."
      }
    }
  },
  "UserProfile": {
    "type": "object",
    "required": ["id", "email", "firstName", "lastName", "createdAt"],
    "properties": {
      "id": {
        "type": "integer"
      },
      "email": {
        "type": "string",
        "format": "email"
      },
      "firstName": {
        "type": "string"
      },
      "lastName": {
        "type": "string"
      },
      "bio": {
        "type": "string",
        "nullable": true,
        "description": "User bio, null if not set"
      },
      "phoneNumber": {
        "type": "string",
        "nullable": true,
        "description": "User phone, null if not provided"
      },
      "createdAt": {
        "type": "string",
        "format": "date-time"
      }
    }
  }
}
```

**Generated TypeScript:**
```typescript
/** User profile update request */
export interface UserProfileUpdate {
  /** Optional: update first name */
  firstName?: string;
  /** Optional: update last name */
  lastName?: string;
  /** Optional: update bio. Pass null to explicitly clear it. */
  bio?: string | null;
  /** Optional: update phone. Pass null to remove it. */
  phoneNumber?: string | null;
}

export interface UserProfile {
  id: number;
  email: string;
  firstName: string;
  lastName: string;
  /** User bio, null if not set */
  bio: string | null;
  /** User phone, null if not provided */
  phoneNumber: string | null;
  createdAt: string;
}
```

**Usage:**
```typescript
// Update only first name (leave everything else unchanged)
const update1: UserProfileUpdate = {
  firstName: "John"
};

// Update bio and explicitly clear phone number
const update2: UserProfileUpdate = {
  bio: "Software engineer",
  phoneNumber: null  // ✅ Explicitly remove phone number
};

// Response always includes all required fields
const profile: UserProfile = {
  id: 1,
  email: "john@example.com",
  firstName: "John",
  lastName: "Doe",
  bio: "Software engineer",
  phoneNumber: null,  // ✅ Phone was cleared
  createdAt: "2025-01-01T00:00:00Z"
};
```

---

### Quick Decision Guide

Use this flowchart to decide which combination to use:

```
Is the field always present in the response/required in the request?
│
├─ YES → Is null a valid value with specific meaning?
│        │
│        ├─ YES → Required + Nullable
│        │        (e.g., managerId, parentId, deletedAt)
│        │
│        └─ NO → Required + Non-nullable
│                 (e.g., id, email, createdAt)
│
└─ NO → Do you need to distinguish "omitted" from "explicitly null"?
         │
         ├─ YES → Optional + Nullable
         │        (e.g., clearing a field via API, three-state fields)
         │
         └─ NO → Optional + Non-nullable ⭐ (MOST COMMON)
                  (e.g., phoneNumber, bio, preferences)
```

---

## Naming Conventions

### Schema Names

✅ **GOOD naming patterns:**
- **Schemas:** `User`, `UserResponse`, `UserRequest`, `UserList`, `UserCreate`, `UserUpdate`, `UserSummary`, `ErrorResponse`, `UserStatus`
- **Request Bodies:** `UserRequest`, `CreateUserRequest`, `UpdateUserRequest`
- **Responses:** `UserResponse`, `NotFoundError`, `ValidationError`, `UnauthorizedError`
- **Headers:** `X-Rate-Limit`, `X-Total-Count`, `X-Request-Id`, `X-Pagination-Offset`

❌ **AVOID:**
- `user` - Use PascalCase, not camelCase
- `UserDTO` - Avoid suffixes like DTO, they're redundant
- `get_user` - Avoid snake_case
- `UserModel` - Avoid Model suffix

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

---

## Working with Enums

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

---

## Complex Types & Patterns

### 1. Union Types (oneOf)

Use `oneOf` for types that can be one of several schemas:

```json
{
  "components": {
    "schemas": {
      "PaymentMethod": {
        "oneOf": [
          {
            "type": "object",
            "required": ["type", "cardNumber"],
            "properties": {
              "type": { "type": "string", "enum": ["credit_card"] },
              "cardNumber": { "type": "string" },
              "expiryDate": { "type": "string" }
            }
          },
          {
            "type": "object",
            "required": ["type", "accountNumber"],
            "properties": {
              "type": { "type": "string", "enum": ["bank_account"] },
              "accountNumber": { "type": "string" },
              "routingNumber": { "type": "string" }
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
export type PaymentMethod =
  | {
      type: "credit_card";
      cardNumber: string;
      expiryDate?: string;
    }
  | {
      type: "bank_account";
      accountNumber: string;
      routingNumber?: string;
    };
```

### 2. Inheritance (allOf)

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

export interface User extends BaseEntity {
  email: string;
  firstName?: string;
  lastName?: string;
}
```

### 3. Arrays and Collections

#### Simple Arrays

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

#### Paginated Responses

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

#### Endpoints That Return Either Plain Array OR Paginated Response

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

### 4. Nested Objects

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

## Common Pitfalls

### ❌ 1. Missing `operationId`

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

### ❌ 2. Inconsistent Property Casing

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

### ❌ 3. Generic Schema Names

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

### ❌ 4. Overusing `additionalProperties`

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

### ❌ 5. Missing Response Schemas

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

---

## Examples

### Example 1: Complete CRUD Resource

```json
{
  "components": {
    "schemas": {
      "Product": {
        "type": "object",
        "description": "A product available for purchase",
        "required": ["id", "name", "price", "createdAt"],
        "properties": {
          "id": {
            "type": "integer",
            "description": "Unique product identifier"
          },
          "name": {
            "type": "string",
            "description": "Product name",
            "minLength": 1,
            "maxLength": 200
          },
          "description": {
            "type": "string",
            "description": "Product description"
          },
          "price": {
            "type": "number",
            "format": "float",
            "description": "Product price in USD",
            "minimum": 0
          },
          "category": {
            "$ref": "#/components/schemas/ProductCategory"
          },
          "stock": {
            "type": "integer",
            "description": "Available stock quantity",
            "minimum": 0
          },
          "images": {
            "type": "array",
            "items": { "type": "string", "format": "uri" },
            "description": "Product image URLs"
          },
          "createdAt": {
            "type": "string",
            "format": "date-time"
          },
          "updatedAt": {
            "type": "string",
            "format": "date-time"
          }
        }
      },
      "ProductCreate": {
        "type": "object",
        "required": ["name", "price"],
        "properties": {
          "name": { "type": "string", "minLength": 1, "maxLength": 200 },
          "description": { "type": "string" },
          "price": { "type": "number", "format": "float", "minimum": 0 },
          "categoryId": { "type": "integer" },
          "stock": { "type": "integer", "minimum": 0 }
        }
      },
      "ProductCategory": {
        "type": "string",
        "enum": ["electronics", "clothing", "food", "books", "other"],
        "description": "Product category"
      }
    }
  },
  "paths": {
    "/products": {
      "get": {
        "operationId": "getProducts",
        "summary": "List all products",
        "tags": ["Products"],
        "parameters": [
          {
            "name": "category",
            "in": "query",
            "schema": { "$ref": "#/components/schemas/ProductCategory" }
          },
          {
            "name": "minPrice",
            "in": "query",
            "schema": { "type": "number" }
          }
        ],
        "responses": {
          "200": {
            "description": "List of products",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": { "$ref": "#/components/schemas/Product" }
                }
              }
            }
          }
        }
      },
      "post": {
        "operationId": "createProduct",
        "summary": "Create a new product",
        "tags": ["Products"],
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": { "$ref": "#/components/schemas/ProductCreate" }
            }
          }
        },
        "responses": {
          "201": {
            "description": "Product created successfully",
            "content": {
              "application/json": {
                "schema": { "$ref": "#/components/schemas/Product" }
              }
            }
          }
        }
      }
    }
  }
}
```

**Generated TypeScript:**
```typescript
/** Product category */
export type ProductCategory = "electronics" | "clothing" | "food" | "books" | "other";

/** A product available for purchase */
export interface Product {
  /** Unique product identifier */
  id: number;
  /** Product name */
  name: string;
  /** Product description */
  description?: string;
  /** Product price in USD */
  price: number;
  category?: ProductCategory;
  /** Available stock quantity */
  stock?: number;
  /** Product image URLs */
  images?: string[];
  createdAt: string;
  updatedAt?: string;
}

export interface ProductCreate {
  name: string;
  description?: string;
  price: number;
  categoryId?: number;
  stock?: number;
}

// API Methods
api.getProducts(params?: { category?: ProductCategory; minPrice?: number }): Promise<Product[]>
api.createProduct(data: ProductCreate): Promise<Product>
```

### Example 2: Union Types with Discriminator

```json
{
  "components": {
    "schemas": {
      "NotificationEmail": {
        "type": "object",
        "required": ["type", "recipient", "subject"],
        "properties": {
          "type": { "type": "string", "enum": ["email"] },
          "recipient": { "type": "string", "format": "email" },
          "subject": { "type": "string" },
          "body": { "type": "string" }
        }
      },
      "NotificationSMS": {
        "type": "object",
        "required": ["type", "phoneNumber", "message"],
        "properties": {
          "type": { "type": "string", "enum": ["sms"] },
          "phoneNumber": { "type": "string" },
          "message": { "type": "string", "maxLength": 160 }
        }
      },
      "NotificationPush": {
        "type": "object",
        "required": ["type", "deviceToken", "title"],
        "properties": {
          "type": { "type": "string", "enum": ["push"] },
          "deviceToken": { "type": "string" },
          "title": { "type": "string" },
          "body": { "type": "string" }
        }
      },
      "Notification": {
        "oneOf": [
          { "$ref": "#/components/schemas/NotificationEmail" },
          { "$ref": "#/components/schemas/NotificationSMS" },
          { "$ref": "#/components/schemas/NotificationPush" }
        ],
        "discriminator": {
          "propertyName": "type"
        }
      }
    }
  }
}
```

**Generated TypeScript:**
```typescript
export interface NotificationEmail {
  type: "email";
  recipient: string;
  subject: string;
  body?: string;
}

export interface NotificationSMS {
  type: "sms";
  phoneNumber: string;
  message: string;
}

export interface NotificationPush {
  type: "push";
  deviceToken: string;
  title: string;
  body?: string;
}

export type Notification = NotificationEmail | NotificationSMS | NotificationPush;
```

**Usage in TypeScript:**
```typescript
function sendNotification(notification: Notification) {
  switch (notification.type) {
    case "email":
      // TypeScript knows this is NotificationEmail
      console.log(`Sending email to ${notification.recipient}`);
      break;
    case "sms":
      // TypeScript knows this is NotificationSMS
      console.log(`Sending SMS to ${notification.phoneNumber}`);
      break;
    case "push":
      // TypeScript knows this is NotificationPush
      console.log(`Sending push to device ${notification.deviceToken}`);
      break;
  }
}
```

## Quick Reference Checklist

When writing OpenAPI schemas, ensure:

- [ ] Schema defined in `components/schemas` (not inline)
- [ ] Enums defined in `components/schemas` (not inline)
- [ ] Meaningful, PascalCase schema name
- [ ] `required` array lists all mandatory fields
- [ ] camelCase property names
- [ ] Descriptions on schemas and properties
- [ ] Appropriate `format` for strings (email, uri, date-time, etc.)
- [ ] Separate schemas for response/request operations when needed (use `Response`/`Request` suffixes, not `-response`/`-request`)
- [ ] Reusable request bodies defined in `components/requestBodies`
- [ ] Reusable responses (especially errors) defined in `components/responses`
- [ ] Common headers (rate limiting, pagination) defined in `components/headers`
- [ ] Endpoint-specific headers can be inlined
- [ ] `operationId` on all path operations
- [ ] Response schemas defined for all status codes
- [ ] Examples provided for complex schemas
- [ ] Validation constraints (min, max, pattern, etc.)

---

## Additional Resources

- [OpenAPI 3.0 Specification](https://swagger.io/specification/)
- [swagger-typescript-api Documentation](https://github.com/acacode/swagger-typescript-api)
- [JSON Schema Validation](https://json-schema.org/understanding-json-schema/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)

---


**Last Updated:** October 2025
**Maintained by:** Frontend Team

