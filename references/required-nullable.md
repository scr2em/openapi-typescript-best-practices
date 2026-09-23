# Required vs Optional & Nullable vs Non-Nullable

Understanding the difference between **required/optional** and **nullable/non-nullable** is crucial for generating accurate TypeScript types. These are two independent concepts that can be combined in four ways.

Contents: the four combinations · each combination in detail (1–4) · comparison table · DO/DON'T · real-world profile update example · quick decision flowchart

## The Four Combinations

| Combination | OpenAPI | TypeScript | Meaning |
|------------|---------|------------|---------|
| **Required + Non-nullable** | `required: ["field"]`<br/>`nullable: false` (default) | `field: string` | Must be provided, cannot be null |
| **Required + Nullable** | `required: ["field"]`<br/>`nullable: true` | `field: string \| null` | Must be provided, can be null |
| **Optional + Non-nullable** | Not in `required`<br/>`nullable: false` (default) | `field?: string` | May be omitted, but if provided cannot be null |
| **Optional + Nullable** | Not in `required`<br/>`nullable: true` | `field?: string \| null` | May be omitted or null |

---

## 1. Required + Non-Nullable (Most Common)

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

## 2. Required + Nullable

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

## 3. Optional + Non-Nullable (Very Common)

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

## 4. Optional + Nullable

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

## Comparison Table with Examples

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

## Best Practices & Recommendations

### ✅ **DO:**

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

### ❌ **DON'T:**

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

## Real-World Example: User Profile Update

Here's a complete example showing when to use each combination:

**OpenAPI Schema:**
```json
{
  "UpdateUserProfileRequest": {
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
  "UserProfileResponse": {
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
export interface UpdateUserProfileRequest {
  /** Optional: update first name */
  firstName?: string;
  /** Optional: update last name */
  lastName?: string;
  /** Optional: update bio. Pass null to explicitly clear it. */
  bio?: string | null;
  /** Optional: update phone. Pass null to remove it. */
  phoneNumber?: string | null;
}

export interface UserProfileResponse {
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
const update1: UpdateUserProfileRequest = {
  firstName: "John"
};

// Update bio and explicitly clear phone number
const update2: UpdateUserProfileRequest = {
  bio: "Software engineer",
  phoneNumber: null  // ✅ Explicitly remove phone number
};

// Response always includes all required fields
const profile: UserProfileResponse = {
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

## Quick Decision Guide

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

