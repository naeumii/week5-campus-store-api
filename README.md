Campus Store API Design Document
1. Endpoints & Pagination
Endpoint Definitions & Parameter Labeling
Retrieve One Product:
```http
  GET /api/v1/products/{id}
  ```
Path Parameter: `{id}` (Integer) — Identifies the specific product resource within the database hierarchy.
List Filtered & Sorted Products:
```http
  GET /api/v1/categories/3/products?sort=price&direction=asc&limit=10
  ```
Path Parameters: `/categories/{id}/products` — Identifies category ID `3`.
Query Parameters: Used for filtering, sorting, and batching the resource collection:
`sort`: Explicitly allowed field for sorting (`price`, `name`).
`direction`: Sort order (`asc` or `desc`).
`limit`: Maximum limits to prevent performance degradation.
Create an Order:
```http
  POST /api/v1/orders
  ```
Pagination Choice: Cursor Pagination
First Request (Initial Batch):
```http
  GET /v1/products?limit=5
  ```
Response containing continuation token:
    ```json
    {
      "data": [
        { "id": 101, "name": "Keyboard", "price": 2499.00 },
        { "id": 102, "name": "Program Jacket", "price": 1250.00 }
      ],
      "pagination": {
        "next_cursor": "eyJpZCI6MTAyLCJzb3J0X2J5IjoidGltZXN0YW1wIn0="
      }
    }
    ```
Next-Batch Request:
```http
  GET /v1/products?limit=5&cursor=eyJpZCI6MTAyLCJzb3J0X2J5IjoidGltZXN0YW1wIn0=
  ```
Explanation: Cursor pagination uses a deterministic token pointer, ensuring consistent results without drift anomalies even if items are added or removed live, unlike heavy offset scans.
---
2. Reliable Order Creation
HTTP Request with Idempotency Header
```http
POST /v1/orders HTTP/1.1
Host: api.campusstore.edu
Idempotency-Key: checkout-abc123
Content-Type: application/json

{
  "product_id": 101,
  "quantity": 1
}
```
Transaction Steps & Concurrency Control
Idempotency Key Check: Recognizes client retries and returns the previously stored response instead of duplicating orders or double-billing.
Database Transactions & Pessimistic Locking: We use Laravel's `lockForUpdate()` within an atomic transaction block:
```php
   DB::transaction(function () use ($id, $qty) {
       $product = Product::where('id', $id)->lockForUpdate()->firstOrFail();
       if ($product->stock < $qty) {
           throw new Exception("Out of stock");
       }
       $product->decrement('stock', $qty);
   });
   ```
Explanation: This locks the specific database row until the transaction finishes, stopping competing buyers from causing negative inventory race conditions.
---
3. Response DTO (Data Transfer Object)
Public Product JSON Schema
```json
{
  "id": 101,
  "name": "Keyboard",
  "price": 2499.00,
  "created_at": "2026-08-15T08:30:00Z"
}
```
Justification for Field Omission
Supplier Cost (`cost`): Omitted from the public response to protect confidential wholesale pricing and markup margins.
Internal Admin Notes (`notes`): Omitted because they contain internal warehouse storage locations and audit remarks meant strictly for staff.
---
4. API Versioning & Compatibility
URI-Based Versioning Requests
Version 1 (Legacy Client Contract):
```http
  GET /api/v1/products/101
  ```
Response (v1 uses `name`):
    ```json
    { "id": 101, "name": "Computer technology uniform" }
    ```
Version 2 (Updated Client Contract):
```http
  GET /api/v2/products/101
  ```
Response (v2 uses `title`):
    ```json
    { "id": 101, "title": "Engineering Drafting Kit" }
    ```
Explanation: URI-based versioning isolates changes cleanly in the path and allows version 1 clients to continue operating uninterrupted until a formal deprecation schedule is enforced.
