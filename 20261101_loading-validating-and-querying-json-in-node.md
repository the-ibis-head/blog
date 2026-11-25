---
title: "Loading, Validating, and Querying JSON in Node"
date: 20261101
tags: JSON
status: local
---

### Changelog

- `0.0.1` Initial publication
- `0.0.2` Added project setup

## Introduction

This article will will be an end-to-end walkthrough where you pull JSON from a file or an HTTP endpoint, validate it, slice it up with Array methods, and print clean output.

We'll use native ES Modules, Node's built-in fetch, and no external validation libraries — just plain JavaScript Array methods doing real work.

## Project Setup

```bash
mkdir order-json-toolkit
cd order-json-toolkit
pnpm init
```

### Add data file

We'll work with a list of e-commerce orders. Six orders, two categories, three statuses — enough variety to make every Array method demo meaningful.

_data.json_

```json
[
  {
    "id": "ORD-1001",
    "product": "Wireless Mouse",
    "category": "Electronics",
    "price": 24.99,
    "quantity": 2,
    "status": "shipped",
    "customerEmail": "ana@example.com",
    "orderDate": "2026-01-03"
  },
  {
    "id": "ORD-1002",
    "product": "Standing Desk",
    "category": "Furniture",
    "price": 349.0,
    "quantity": 1,
    "status": "pending",
    "customerEmail": "ben@example.com",
    "orderDate": "2026-01-05"
  },
  {
    "id": "ORD-1003",
    "product": "USB-C Hub",
    "category": "Electronics",
    "price": 39.5,
    "quantity": 3,
    "status": "shipped",
    "customerEmail": "ana@example.com",
    "orderDate": "2026-01-06"
  },
  {
    "id": "ORD-1004",
    "product": "Office Chair",
    "category": "Furniture",
    "price": 199.99,
    "quantity": 1,
    "status": "cancelled",
    "customerEmail": "cleo@example.com",
    "orderDate": "2026-01-07"
  },
  {
    "id": "ORD-1005",
    "product": "Mechanical Keyboard",
    "category": "Electronics",
    "price": 89.0,
    "quantity": 1,
    "status": "shipped",
    "customerEmail": "dev@example.com",
    "orderDate": "2026-01-09"
  },
  {
    "id": "ORD-1006",
    "product": "Desk Lamp",
    "category": "Furniture",
    "price": 29.95,
    "quantity": 4,
    "status": "pending",
    "customerEmail": "ben@example.com",
    "orderDate": "2026-01-10"
  }
]
```

### Add entry file (index.js)

**Notes**

- We are resolving the path relative to `__dirname` (derived from `import.meta.url`). `ESM` (ECMAScript Modules) has no `__dirname` built in which means this works regardless of the current working directory the script is run from.

- `JSON.parse` throws a `SyntaxError` on malformed JSON so we could wrap the call site in `try/catch` but we'll centralize that later.

- If using fetch data from any JSON API we use `loadFromHttp` instead of `loadFromFile`.
  - `response.json()` parses the body for you so no need to call `JSON.parse`.
  - The only extra responsibility you have over the file-loading path is checking `response.ok`, since fetch does not reject on HTTP error status codes like `404` or `500`, only on network failure.

_index.js_

```javascript
import path from "node:path";
import { readFile } from "node:fs/promises";
import { fileURLToPath } from "node:url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

export async function loadFromFile(relativePath = "orders.json") {
  const fullPath = path.join(__dirname, relativePath);
  const raw = await readFile(fullPath, "utf-8");
  return JSON.parse(raw);
}

export async function loadFromHttp(url) {
  const response = await fetch(url);
  if (!response.ok) {
    throw new Error(
      `Request failed: ${response.status} ${response.statusText}`,
    );
  }
  return response.json();
}
```

Optionally we could add a small wrapper that tries HTTP first and falls back to the local file.

```javascript
export async function loadOrders({ url, fallbackPath } = {}) {
  if (url) {
    try {
      return await loadFromHttp(url);
    } catch (err) {
      console.warn(
        `HTTP load failed (${err.message}), falling back to local file.`,
      );
    }
  }
  return loadFromFile(fallbackPath);
}
```

### Validating the data

With untrusted input (a file or an API response) you have to validate before you trust the shape. We'll do this with plain Array methods rather than pulling in a schema library, so you can see exactly what's being checked.

- Using `every()` inside `isValidOrder` confirms every required key is present on a single order and it short-circuits on the first missing field, so it's cheap.
- Using `filter()` splits the full array into valid and invalid buckets in one pass each.

```javascript
const REQUIRED_FIELDS = [
  "id",
  "product",
  "category",
  "price",
  "quantity",
  "status",
  "customerEmail",
  "orderDate",
];
const VALID_STATUSES = ["pending", "shipped", "cancelled"];

function isValidOrder(order) {
  if (typeof order !== "object" || order === null) return false;
  const hasAllFields = REQUIRED_FIELDS.every((field) => field in order);
  if (!hasAllFields) return false;
  const typeChecksPass =
    typeof order.id === "string" &&
    typeof order.product === "string" &&
    typeof order.price === "number" &&
    order.price >= 0 &&
    typeof order.quantity === "number" &&
    Number.isInteger(order.quantity) &&
    order.quantity > 0 &&
    VALID_STATUSES.includes(order.status) &&
    typeof order.customerEmail === "string" &&
    order.customerEmail.includes("@");
  return typeChecksPass;
}

export function validateOrders(orders) {
  if (!Array.isArray(orders)) {
    throw new TypeError("Expected the loaded JSON to be an array of orders.");
  }
  const valid = orders.filter(isValidOrder);
  const invalid = orders.filter((order) => !isValidOrder(order));
  return {
    valid,
    invalid,
    allValid: invalid.length === 0,
    summary: `${valid.length}/${orders.length} orders passed validation`,
  };
}
```
