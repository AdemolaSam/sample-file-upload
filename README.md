# Frontend Upload Integration Guide

This guide describes how the frontend should integrate with the current file upload flow.

## Overview

The upload flow has 3 steps:

1. Request one or more presigned upload URLs from the backend.
2. Upload each file directly to object storage using the returned `uploadUrl`.
3. Call the backend callback endpoint so the uploaded file keys can be persisted to the database.

The backend does not receive the raw file bytes. It only:

- validates upload intent
- issues presigned URLs
- verifies uploaded objects exist
- persists file keys to the right models

## Endpoints

- `POST /v1/files/upload-url`
- `POST /v1/files/upload/callback`

Both endpoints require JWT authentication.

## Supported Folders

- `products`
- `store`
- `profile`
- `vendor_docs`
- `receipts`

## Supported File Types

The request DTO currently accepts:

- `image/jpeg`
- `image/jpg`
- `image/png`
- `image/webp`
- `application/pdf`
- `application/msword`
- `application/vnd.openxmlformats-officedocument.wordprocessingml.document`

## Step 1: Request Upload URLs

Send an array so multiple files can be prepared in one request.

### Product image sample

```http
POST /v1/files/upload-url
Authorization: Bearer <token>
Content-Type: application/json
```

```json
[
  {
    "fileName": "tomatoes-1.png",
    "fileType": "image/png",
    "fileSize": 284331,
    "folder": "products",
    "documentTitle": "product:8e9d6f7a-1111-2222-3333-444444444444"
  },
  {
    "fileName": "tomatoes-2.png",
    "fileType": "image/png",
    "fileSize": 301442,
    "folder": "products",
    "documentTitle": "variant:9b8c7d6e-1111-2222-3333-555555555555"
  }
]
```

### Store image sample

```json
[
  {
    "fileName": "logo.png",
    "fileType": "image/png",
    "fileSize": 142221,
    "folder": "store",
    "documentTitle": "logo"
  },
  {
    "fileName": "banner.png",
    "fileType": "image/png",
    "fileSize": 522198,
    "folder": "store",
    "documentTitle": "banner"
  }
]
```

### Profile image sample

```json
[
  {
    "fileName": "avatar.png",
    "fileType": "image/png",
    "fileSize": 120441,
    "folder": "profile",
    "documentTitle": "profile_image"
  }
]
```

### Receipt sample

```json
[
  {
    "fileName": "invoice.pdf",
    "fileType": "application/pdf",
    "fileSize": 211200,
    "folder": "receipts",
    "documentTitle": "order:1a2b3c4d-1111-2222-3333-666666666666",
    "category": "vendor_invoice"
  }
]
```

### Sample response

```json
{
  "results": [
    {
      "success": true,
      "uploadUrl": "https://...",
      "key": "products/<userId>/1741780000000-tomatoes-1.png",
      "expiresIn": 300
    },
    {
      "success": true,
      "uploadUrl": "https://...",
      "key": "products/<userId>/1741780000100-tomatoes-2.png",
      "expiresIn": 300
    }
  ]
}
```

## Step 2: Upload Files Directly to Storage

Use the returned `uploadUrl` and send the raw file bytes with `PUT`.

Do not send `multipart/form-data` to the storage URL.

```ts
async function uploadToStorage(uploadUrl: string, file: File) {
  const response = await fetch(uploadUrl, {
    method: "PUT",
    headers: {
      "Content-Type": file.type,
    },
    body: file,
  });

  if (!response.ok) {
    throw new Error(`Storage upload failed with status ${response.status}`);
  }
}
```

## Step 3: Finalize Uploads with Callback

After storage uploads succeed, notify the backend using the same metadata plus the returned `key`.

### Callback request sample for product and store images

```http
POST /v1/files/upload/callback
Authorization: Bearer <token>
Content-Type: application/json
```

```json
[
  {
    "key": "products/<userId>/1741780000000-tomatoes-1.png",
    "folder": "products",
    "documentTitle": "product:8e9d6f7a-1111-2222-3333-444444444444"
  },
  {
    "key": "store/<userId>/1741780000200-logo.png",
    "folder": "store",
    "documentTitle": "logo"
  },
  {
    "key": "profile/<userId>/1741780000300-avatar.png",
    "folder": "profile",
    "documentTitle": "profile_image"
  }
]
```

### Callback request sample for receipts

```json
[
  {
    "key": "receipts/<userId>/1741780000400-invoice.pdf",
    "folder": "receipts",
    "documentTitle": "order:1a2b3c4d-1111-2222-3333-666666666666",
    "category": "vendor_invoice"
  }
]
```

### Sample callback response

```json
{
  "results": [
    {
      "success": true,
      "data": {
        "type": "product",
        "id": "8e9d6f7a-1111-2222-3333-444444444444",
        "key": "products/<userId>/1741780000000-tomatoes-1.png"
      }
    },
    {
      "success": true,
      "data": {
        "type": "logo",
        "key": "store/<userId>/1741780000200-logo.png"
      }
    },
    {
      "success": false,
      "key": "profile/<userId>/1741780000300-avatar.png",
      "error": "Upload processing failed"
    }
  ]
}
```

## `documentTitle` Rules

- `products`: use `product:{id}` or `variant:{id}`
- `store`: use `logo` or `banner`
- `profile`: use a simple label like `profile_image`
- `receipts`: use `order:{orderId}`
- `vendor_docs`: use a document label like `CAC` or `NIN`

## `category` Rules

`category` is optional in general, but is required for receipts.

Allowed receipt values:

- `vendor_invoice`
- `payment_confirmation`
- `delivery_note`
- `refund_receipt`

## Frontend Reference Implementation

This example shows the recommended request sequence for multiple files.

```ts
type UploadIntent = {
  file: File;
  folder: "products" | "store" | "profile" | "vendor_docs" | "receipts";
  documentTitle: string;
  category?: string;
};

async function uploadFiles(token: string, intents: UploadIntent[]) {
  const uploadUrlRes = await fetch("/v1/files/upload-url", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify(
      intents.map((item) => ({
        fileName: item.file.name,
        fileType: item.file.type,
        fileSize: item.file.size,
        folder: item.folder,
        documentTitle: item.documentTitle,
        category: item.category,
      })),
    ),
  });

  if (!uploadUrlRes.ok) {
    throw new Error("Failed to get upload URLs");
  }

  const { results } = await uploadUrlRes.json();

  const successfulUploads: Array<{
    key: string;
    folder: UploadIntent["folder"];
    documentTitle: string;
    category?: string;
  }> = [];

  await Promise.all(
    results.map(async (result: any, index: number) => {
      if (!result.success) {
        return;
      }

      const item = intents[index];

      await uploadToStorage(result.uploadUrl, item.file);

      successfulUploads.push({
        key: result.key,
        folder: item.folder,
        documentTitle: item.documentTitle,
        category: item.category,
      });
    }),
  );

  if (successfulUploads.length === 0) {
    return { results: [] };
  }

  const callbackRes = await fetch("/v1/files/upload/callback", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify(successfulUploads),
  });

  if (!callbackRes.ok) {
    throw new Error("Upload callback failed");
  }

  return callbackRes.json();
}
```

## Recommended Frontend Behavior

- Send presign requests in batches.
- Upload files in parallel with limited concurrency.
- Only send successfully uploaded files to the callback endpoint.
- Treat callback results as partial success, not all-or-nothing.
- Show per-file upload and finalize errors in the UI.

## Suggested Concurrency

For image uploads, a practical default is 3 to 5 parallel storage uploads at a time.

This keeps the UI responsive without creating unnecessary network pressure.

## Error Handling Notes

The callback can fail per file even if storage upload succeeded.

Common reasons:

- invalid `documentTitle`
- invalid `category`
- file key does not belong to the authenticated user
- target product, variant, vendor, user, or order was not found
- file exceeds max size
- max image count was reached

The frontend should surface file-level errors and allow retry where appropriate.
