# Schema Design

Contents: components for schemas/enums, requestBodies, responses · headers (shared vs inline) · descriptions · formats · separate Request/Response schemas

## 1. Always Use `components` for Definitions

**⚠️ CRITICAL RULE: Never define schemas, enums, request bodies, or responses inline in your paths.**

Define all reusable types in the `components` section rather than inline. 

### Use `components/schemas` for Data Models and Enums

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

### Use `components/requestBodies` for Request Bodies

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

### Use `components/responses` for Responses

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

## 2. Use `components/headers` for Common Headers, Inline for Endpoint-Specific Ones

Headers follow a different pattern than schemas - inline them when they're specific to an endpoint, but use `components/headers` when they're shared across multiple endpoints.

### Use `components/headers` for Common Headers

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

### Inline Headers for Endpoint-Specific Ones

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

## 3. Add Descriptions for Better Documentation

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

## 4. Use `format` for Better Type Hints

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

## 5. Define Separate Response/Request Schemas

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

