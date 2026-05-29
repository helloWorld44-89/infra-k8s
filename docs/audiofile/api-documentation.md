---
title: AudioFile API Documentation
date: 2026-05-29
author: Hermes Docs Agent
tags: [api, documentation, audiobook, swagger]
---

# AudioFile API Documentation

## Overview

AudioFile provides a comprehensive REST API for programmatic access to all audiobook management features. The API follows RESTful conventions and uses OpenAPI 3.0 specification for standardization.

**Base URL**: `http://localhost:8080/api`

**Authentication**: 
- OIDC/SSO tokens (when authentication is enabled)
- API keys (for programmatic access)

**Content-Type**: `application/json`

## Authentication

### API Keys

```bash
# Create API key (admin only)
curl -X POST "http://localhost:8080/api/api-keys" \
  -H "Content-Type: application/json" \
  -d '{"name": "My Integration", "description": "For automation"}'
```

### Using API Keys

Include API key in headers:
```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
     -H "Content-Type: application/json" \
     "http://localhost:8080/api/books"
```

## API Endpoints

### Books API

#### List Books

```http
GET /api/books
```

Get a paginated list of all books in the library.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | integer | Page number (default: 1) |
| `limit` | integer | Items per page (default: 20, max: 100) |
| `search` | string | Search term |
| `author` | string | Filter by author |
| `series` | string | Filter by series |
| `format` | string | Filter by format (m4b, mp3, flac, opus) |
| `sort` | string | Sort field (title, author, date_added) |
| `order` | string | Sort order (asc, desc) |

**Response:**
```json
{
  "data": [
    {
      "id": 1,
      "title": "The Lord of the Rings",
      "author": "J.R.R. Tolkien",
      "narrator": "Rob Inglis",
      "series": "The Lord of the Rings",
      "series_number": 1,
      "format": "m4b",
      "duration": 3600,
      "file_size": 104857600,
      "quality": "high",
      "has_chapters": true,
      "chapter_count": 20,
      "date_added": "2026-05-29T10:00:00Z",
      "cover_url": "/api/covers/1",
      "progress": 0.75
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100,
    "total_pages": 5
  }
}
```

#### Get Book Details

```http
GET /api/books/{id}
```

Get detailed information about a specific book.

**Response:**
```json
{
  "id": 1,
  "title": "The Lord of the Rings",
  "author": "J.R.R. Tolkien",
  "narrator": "Rob Inglis",
  "series": "The Lord of the Rings",
  "series_number": 1,
  "isbn": "9780345339683",
  "publisher": " Recorded Books",
  "publish_year": 1990,
  "genre": "Fantasy",
  "description": "Epic fantasy adventure...",
  "format": "m4b",
  "duration": 3600,
  "file_size": 104857600,
  "quality": "high",
  "has_chapters": true,
  "chapter_count": 20,
  "chapters": [
    {
      "id": 1,
      "title": "Book One - The Fellowship of the Ring",
      "start_time": 0,
      "end_time": 1800,
      "duration": 1800
    }
  ],
  "date_added": "2026-05-29T10:00:00Z",
  "date_modified": "2026-05-29T10:00:00Z",
  "cover_url": "/api/covers/1",
  "file_path": "/library/The Lord of the Rings.m4b",
  "progress": 0.75,
  "last_position": 2700,
  "bookmarks": [
    {
      "id": 1,
      "title": "Important Quote",
      "time": 1800,
      "notes": "Frodo leaves the Shire"
    }
  ]
}
```

#### Create Book

```http
POST /api/books
```

Create a new book record (manual creation).

**Request:**
```json
{
  "title": "New Book",
  "author": "Author Name",
  "narrator": "Narrator Name",
  "series": "Series Name",
  "series_number": 1,
  "isbn": "1234567890",
  "publisher": "Publisher Name",
  "publish_year": 2023,
  "genre": "Fiction",
  "description": "Book description",
  "file_path": "/library/new_book.m4b",
  "format": "m4b",
  "duration": 3600,
  "file_size": 104857600,
  "quality": "high",
  "has_chapters": true,
  "chapter_count": 15
}
```

**Response:**
```json
{
  "id": 2,
  "title": "New Book",
  "author": "Author Name",
  "narrator": "Narrator Name",
  "series": "Series Name",
  "series_number": 1,
  "isbn": "1234567890",
  "publisher": "Publisher Name",
  "publish_year": 2023,
  "genre": "Fiction",
  "description": "Book description",
  "format": "m4b",
  "duration": 3600,
  "file_size": 104857600,
  "quality": "high",
  "has_chapters": true,
  "chapter_count": 15,
  "date_added": "2026-05-29T11:00:00Z",
  "cover_url": "/api/covers/2",
  "progress": 0
}
```

#### Update Book

```http
PUT /api/books/{id}
```

Update book metadata.

**Request:**
```json
{
  "title": "Updated Title",
  "author": "Updated Author",
  "narrator": "Updated Narrator",
  "series": "Updated Series",
  "series_number": 2,
  "genre": "Updated Genre",
  "description": "Updated description"
}
```

**Response:**
```json
{
  "id": 2,
  "title": "Updated Title",
  "author": "Updated Author",
  "narrator": "Updated Narrator",
  "series": "Updated Series",
  "series_number": 2,
  "genre": "Updated Genre",
  "description": "Updated description",
  "date_modified": "2026-05-29T11:30:00Z"
}
```

#### Delete Book

```http
DELETE /api/books/{id}
```

Delete a book from the library.

**Response:**
```json
{
  "message": "Book deleted successfully",
  "id": 2
}
```

### Conversions API

#### List Conversions

```http
GET /api/conversions
```

Get list of conversion jobs.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by status (pending, running, completed, failed) |
| `format` | string | Filter by target format |
| `page` | integer | Page number |
| `limit` | integer | Items per page |

**Response:**
```json
{
  "data": [
    {
      "id": 1,
      "book_id": 1,
      "book_title": "The Lord of the Rings",
      "source_format": "aax",
      "target_format": "m4b",
      "status": "completed",
      "progress": 100,
      "quality": "high",
      "created_at": "2026-05-29T10:00:00Z",
      "started_at": "2026-05-29T10:05:00Z",
      "completed_at": "2026-05-29T10:45:00Z",
      "duration": 2400,
      "file_path": "/library/The Lord of the Rings.m4b",
      "file_size": 104857600,
      "error_message": null
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 10,
    "total_pages": 1
  }
}
```

#### Create Conversion

```http
POST /api/conversions
```

Start a new conversion job.

**Request:**
```json
{
  "book_id": 1,
  "target_format": "mp3",
  "quality": "medium",
  "remove_drm": true,
  "activation_bytes": "1234567890123456"
}
```

**Response:**
```json
{
  "id": 2,
  "book_id": 1,
  "book_title": "The Lord of the Rings",
  "source_format": "m4b",
  "target_format": "mp3",
  "status": "pending",
  "progress": 0,
  "quality": "medium",
  "created_at": "2026-05-29T12:00:00Z",
  "estimated_duration": 1800
}
```

#### Get Conversion Details

```http
GET /api/conversions/{id}
```

Get detailed information about a conversion job.

**Response:**
```json
{
  "id": 2,
  "book_id": 1,
  "book_title": "The Lord of the Rings",
  "source_format": "m4b",
  "target_format": "mp3",
  "status": "running",
  "progress": 45,
  "quality": "medium",
  "created_at": "2026-05-29T12:00:00Z",
  "started_at": "2026-05-29T12:05:00Z",
  "estimated_duration": 1800,
  "current_stage": "converting",
  "stage_progress": 75,
  "log_entries": [
    {
      "timestamp": "2026-05-29T12:05:00Z",
      "level": "info",
      "message": "Starting conversion"
    },
    {
      "timestamp": "2026-05-29T12:10:00Z",
      "level": "info",
      "message": "Processing chapters"
    }
  ]
}
```

#### Cancel Conversion

```http
PUT /api/conversions/{id}/cancel
```

Cancel a running conversion job.

**Response:**
```json
{
  "id": 2,
  "status": "cancelled",
  "cancelled_at": "2026-05-29T12:15:00Z",
  "message": "Conversion cancelled by user"
}
```

### Import API

#### Start Import

```http
POST /api/books/import
```

Trigger manual import of audiobook files.

**Request:**
```json
{
  "files": [
    "/path/to/book1.aax",
    "/path/to/book2.m4b"
  ],
  "auto_convert": true,
  "target_format": "m4b",
  "quality": "high"
}
```

**Response:**
```json
{
  "import_id": "12345678-1234-1234-1234-123456789012",
  "message": "Import started",
  "estimated_duration": 300
}
```

#### Get Import Status

```http
GET /api/import/status/{import_id}
```

Check import progress.

**Response:**
```json
{
  "import_id": "12345678-1234-1234-1234-123456789012",
  "status": "processing",
  "progress": 60,
  "processed_files": 2,
  "total_files": 3,
  "results": [
    {
      "file": "/path/to/book1.aax",
      "status": "completed",
      "book_id": 1,
      "duration": 120
    },
    {
      "file": "/path/to/book2.m4b",
      "status": "completed",
      "book_id": 2,
      "duration": 90
    },
    {
      "file": "/path/to/book3.aax",
      "status": "processing",
      "progress": 75
    }
  ]
}
```

### User API

#### Get Current User

```http
GET /api/users/me
```

Get current user profile.

**Response:**
```json
{
  "id": 1,
  "username": "john_doe",
  "name": "John Doe",
  "email": "john@example.com",
  "role": "admin",
  "preferences": {
    "theme": "dark",
    "language": "en",
    "default_speed": 1.0,
    "volume": 0.8
  },
  "created_at": "2026-05-01T00:00:00Z",
  "last_login": "2026-05-29T10:00:00Z"
}
```

#### Update User Preferences

```http
PUT /api/users/me
```

Update user preferences.

**Request:**
```json
{
  "preferences": {
    "theme": "light",
    "language": "en",
    "default_speed": 1.25,
    "volume": 0.9
  }
}
```

**Response:**
```json
{
  "id": 1,
  "username": "john_doe",
  "name": "John Doe",
  "email": "john@example.com",
  "role": "admin",
  "preferences": {
    "theme": "light",
    "language": "en",
    "default_speed": 1.25,
    "volume": 0.9
  },
  "updated_at": "2026-05-29T12:00:00Z"
}
```

### Playback API

#### Get Playback Position

```http
GET /api/playback/{book_id}
```

Get current playback position for a book.

**Response:**
```json
{
  "book_id": 1,
  "position": 2700,
  "duration": 3600,
  "progress": 0.75,
  "last_played": "2026-05-29T11:00:00Z"
}
```

#### Update Playback Position

```http
POST /api/playback/{book_id}
```

Update current playback position.

**Request:**
```json
{
  "position": 3000,
  "progress": 0.83
}
```

**Response:**
```json
{
  "book_id": 1,
  "position": 3000,
  "progress": 0.83,
  "updated_at": "2026-05-29T12:00:00Z"
}
```

#### Add Bookmark

```http
POST /api/playbookmarks/{book_id}
```

Add a bookmark to a book.

**Request:**
```json
{
  "title": "Important Quote",
  "time": 1800,
  "notes": "Frodo leaves the Shire"
}
```

**Response:**
```json
{
  "id": 1,
  "book_id": 1,
  "title": "Important Quote",
  "time": 1800,
  "notes": "Frodo leaves the Shire",
  "created_at": "2026-05-29T12:00:00Z"
}
```

### System API

#### Health Check

```http
GET /health
```

Check system health.

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2026-05-29T12:00:00Z",
  "version": "1.0.0",
  "database": "connected",
  "redis": "connected",
  "queue_length": 0,
  "uptime": 86400
}
```

#### Get System Info

```http
GET /api/system/info
```

Get system information.

**Response:**
```json
{
  "version": "1.0.0",
  "build_date": "2026-05-29",
  "python_version": "3.11.0",
  "total_books": 100,
  "total_size": 10737418240,
  "conversion_queue": 0,
  "active_users": 1,
  "uptime": 86400,
  "memory_usage": 512,
  "cpu_usage": 25.5
}
```

### Webhooks

#### Import Complete Webhook

When an import job completes, AudioFile sends a webhook to the configured URL.

**Request:**
```json
{
  "event": "import_complete",
  "import_id": "12345678-1234-1234-1234-123456789012",
  "status": "completed",
  "processed_files": 3,
  "total_files": 3,
  "duration": 300,
  "timestamp": "2026-05-29T12:00:00Z",
  "results": [
    {
      "file": "/path/to/book1.aax",
      "status": "completed",
      "book_id": 1,
      "duration": 120
    }
  ]
}
```

#### Conversion Complete Webhook

When a conversion job completes, AudioFile sends a webhook.

**Request:**
```json
{
  "event": "conversion_complete",
  "conversion_id": 2,
  "book_id": 1,
  "status": "completed",
  "source_format": "aax",
  "target_format": "m4b",
  "duration": 2400,
  "file_path": "/library/The Lord of the Rings.m4b",
  "file_size": 104857600,
  "timestamp": "2026-05-29T12:00:00Z"
}
```

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Book not found",
    "details": "Book with ID 999 not found"
  },
  "timestamp": "2026-05-29T12:00:00Z"
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `VALIDATION_ERROR` | 400 | Request validation failed |
| `UNAUTHORIZED` | 401 | Authentication failed |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `CONFLICT` | 409 | Resource conflict |
| `INTERNAL_ERROR` | 500 | Internal server error |
| `SERVICE_UNAVAILABLE` | 503 | Service unavailable |

## Rate Limiting

API endpoints are rate limited to prevent abuse:

- **General endpoints**: 100 requests per minute
- **Conversion endpoints**: 10 requests per minute
- **Import endpoints**: 5 requests per minute

Rate limit headers are included in responses:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1678901234
```

## SDK Libraries

### Python SDK

```python
import audiofile

# Initialize client
client = audiofile.Client(
    base_url="http://localhost:8080",
    api_key="your-api-key"
)

# Get books
books = client.books.list(page=1, limit=20)

# Create conversion
conversion = client.conversions.create(
    book_id=1,
    target_format="mp3",
    quality="medium"
)

# Get import status
status = client.imports.get_status("12345678-1234-1234-1234-123456789012")
```

### JavaScript SDK

```javascript
import { AudioFileClient } from 'audiofile-sdk';

// Initialize client
const client = new AudioFileClient({
  baseUrl: 'http://localhost:8080',
  apiKey: 'your-api-key'
});

// Get books
const books = await client.books.list({ page: 1, limit: 20 });

// Start conversion
const conversion = await client.conversions.create({
  bookId: 1,
  targetFormat: 'mp3',
  quality: 'medium'
});
```

## Webhook Configuration

### Setting Up Webhooks

1. **Webhook URL**: Configure in settings
2. **Events**: Select which events to receive
3. **Secret**: Optional HMAC signature for verification
4. **Retries**: Configure retry attempts for failed deliveries

### Webhook Verification

If using HMAC signature:

```bash
# Signature calculation
signature = HMAC-SHA256(secret, request_body)
```

Verify signature with `X-Webhook-Signature` header.

---

*This API documentation is part of the AudioFile project. For the latest updates and additional examples, visit the project repository.*