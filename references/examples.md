# Full Examples

## Example 1: Complete CRUD Resource

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
      "CreateProductRequest": {
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
    },
    "requestBodies": {
      "CreateProductRequest": {
        "description": "Product data for creation",
        "required": true,
        "content": {
          "application/json": {
            "schema": { "$ref": "#/components/schemas/CreateProductRequest" }
          }
        }
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
        "requestBody": { "$ref": "#/components/requestBodies/CreateProductRequest" },
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

export interface CreateProductRequest {
  name: string;
  description?: string;
  price: number;
  categoryId?: number;
  stock?: number;
}

// API Methods
api.getProducts(params?: { category?: ProductCategory; minPrice?: number }): Promise<Product[]>
api.createProduct(data: CreateProductRequest): Promise<Product>
```

## Example 2: Union Types with Discriminator

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
