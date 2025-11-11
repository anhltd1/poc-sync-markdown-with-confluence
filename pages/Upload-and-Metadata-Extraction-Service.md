# Upload and Metadata Extraction Service

**Version:** 1.0.2  
**Last Updated:** November 11, 2025

---

## 📋 Service Overview

The Upload and Metadata Extraction Service is the core processing pipeline that handles document ingestion, OCR text extraction, document classification, and intelligent metadata generation. This service processes documents asynchronously through multiple background tasks.

### Service Capabilities

- ✅ Multi-format document upload (PDF, DOCX, PPT, PPTX)
- ✅ Automatic PDF conversion for non-PDF formats
- ✅ Mistral AI-powered OCR processing
- ✅ Image and formula extraction
- ✅ AI-driven document classification
- ✅ Automatic metadata generation (difficulty, topics, summary)

---

## 🔄 Service Architecture

### Processing Flow

```
┌──────────────────┐
│  1. File Upload  │
│  & Validation    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  2. Document     │
│  Conversion      │
│  (to PDF)        │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  3. MongoDB      │
│  Initial Save    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Background Tasks │
│   (Parallel)     │
└────────┬─────────┘
         │
         ├─────────────┬─────────────┐
         ▼             ▼             ▼
   ┌─────────┐  ┌──────────┐  ┌──────────┐
   │   OCR   │  │Document  │  │ Metadata │
   │Processing│  │Classifier│  │Extractor │
   │(30-60s) │  │ (5-10s)  │  │ (5-10s)  │
   └────┬────┘  └────┬─────┘  └────┬─────┘
        │            │             │
        └────────────┴─────────────┘
                     │
                     ▼
            ┌────────────────┐
            │ MongoDB Update │
            │ - OCR_PAGES    │
            │ - IMAGES       │
            │ - Metadata     │
            └────────────────┘
```

### Background Tasks

| Task # | Name                        | Description                             | Duration | Dependencies           |
| ------ | --------------------------- | --------------------------------------- | -------- | ---------------------- |
| 1      | **OCR Processing**          | Extract text, images, formulas from PDF | 30-60s   | Mistral AI API         |
| 2      | **Document Classification** | Identify document type                  | 5-10s    | OpenAI GPT-4, PDF file |
| 3      | **Metadata Extraction**     | Generate difficulty, topics, summary    | 5-10s    | OpenAI GPT-4, PDF file |

> **Note:** Tasks run in parallel for optimal performance

---

## 📤 Upload Endpoint

### API Endpoint

```
POST /api/v1/upload/
```

### Request

**Content-Type:** `multipart/form-data`

**Parameters:**

| Parameter | Type | Required | Description                          |
| --------- | ---- | -------- | ------------------------------------ |
| `file`    | File | Yes      | Document file (PDF, DOCX, PPT, PPTX) |

**Example:**

```bash
curl -X POST "http://localhost:8000/api/v1/upload/" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@textbook.pdf"
```

### Response

**Status Code:** `200 OK`

**Response Body:**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "content": "",
  "metadata": {
    "file_name": "textbook.pdf",
    "file_type": "pdf",
    "upload_time": "2025-11-11T10:00:00.000Z",
    "uploaded_by": "admin",
    "file_size": 2048576,
    "num_pages": 150,
    "document_type": "",
    "difficulty": "",
    "topics": [],
    "summary": "",
    "ocr_processed": false,
    "structure_processed": false,
    "ocr_page_ids": []
  }
}
```

**Key Fields:**

- `id`: Unique file identifier (use this for all subsequent API calls)
- `metadata.num_pages`: Total pages detected
- `metadata.ocr_processed`: Will be `true` when OCR completes
- `metadata.structure_processed`: Will be `true` when structure extraction completes

### Error Responses

| Status Code | Error                 | Description                         |
| ----------- | --------------------- | ----------------------------------- |
| `400`       | Bad Request           | Invalid file type or file too large |
| `500`       | Internal Server Error | Processing error                    |

**Example Error:**

```json
{
  "detail": "File type .txt not allowed. Allowed types: pdf, docx, ppt, pptx"
}
```

---

## 🔍 OCR Processing Service

### Overview

Uses Mistral AI's advanced OCR engine to extract text, images, and mathematical formulas from documents.

### Features

- **Text Extraction**: High-accuracy text recognition
- **Formula Recognition**: LaTeX math formula extraction
- **Image Extraction**: Automatic image detection and saving
- **Markdown Output**: Structured markdown format with headings

### Output Storage

**OCR_PAGES Collection:**

```json
{
  "_id": "ObjectId(...)",
  "file_id": "550e8400-e29b-41d4-a716-446655440000",
  "page_number": 0,
  "md_path": "ocr_results/.../text/page_0.txt",
  "images": ["page_0_image_1.jpg", "page_0_image_2.jpg"],
  "text_length": 1024,
  "created_at": "2025-11-11T10:01:30.000Z",
  "ocr_engine": "mistral-ocr-latest"
}
```

**IMAGES Collection:**

```json
{
  "_id": "ObjectId(...)",
  "image_id": "page_0_image_1",
  "file_id": "550e8400-e29b-41d4-a716-446655440000",
  "page_index": 0,
  "image_path": "ocr_results/.../images/image-1.jpg",
  "created_at": "2025-11-11T10:01:30.000Z"
}
```

### API Endpoints

**Get All OCR Pages:**

```
GET /api/v1/files/{file_id}/ocr-pages
```

**Get Specific Page:**

```
GET /api/v1/files/{file_id}/pages/{page_number}
```

**Download Page Text:**

```
GET /api/v1/files/{file_id}/pages/{page_number}/text
```

**Get All Images:**

```
GET /api/v1/files/{file_id}/images
```

**Download Image:**

```
GET /api/v1/images/{image_id}
```

---

## 🏷️ Document Classification Service

### Overview

Uses OpenAI GPT-4 to automatically classify documents into three categories: slides, textbooks, or syllabi.

### Classification Logic

**Input:**

- PDF file content (first few pages)
- File metadata (name, page count)

**Output:**

- Document type: `slides` | `textbook` | `syllabus`

**Classification Criteria:**

| Document Type | Indicators                                                                         |
| ------------- | ---------------------------------------------------------------------------------- |
| **Slides**    | Bullet points, minimal text per page, visual elements, presentation layout         |
| **Textbook**  | Chapters, sections, table of contents, continuous narrative, detailed explanations |
| **Syllabus**  | Course information, schedule, grading policies, objectives, requirements           |

### Storage

Updated in `FILES.metadata.document_type`:

```json
{
  "metadata": {
    "document_type": "textbook"
  }
}
```

---

## 📊 Metadata Extraction Service

### Overview

Generates intelligent metadata using OpenAI GPT-4 by analyzing document content and structure.

### Extracted Metadata

| Field          | Type          | Description              | Example                                      |
| -------------- | ------------- | ------------------------ | -------------------------------------------- |
| **difficulty** | String (1-5)  | Content difficulty level | `"3"` (Intermediate)                         |
| **topics**     | Array[String] | Main topics covered      | `["Calculus", "Derivatives", "Integration"]` |
| **summary**    | String        | Brief content summary    | `"Introduction to calculus concepts..."`     |

### Difficulty Levels

| Level | Label        | Description                           |
| ----- | ------------ | ------------------------------------- |
| 1     | Beginner     | Basic concepts, introductory level    |
| 2     | Elementary   | Foundation with some depth            |
| 3     | Intermediate | Standard college/university level     |
| 4     | Advanced     | Graduate level, specialized knowledge |
| 5     | Expert       | Research level, highly specialized    |

### Metadata Generation Process

1. **Content Analysis**: Analyze full document text
2. **Topic Extraction**: Identify main topics and keywords
3. **Difficulty Assessment**: Evaluate complexity and prerequisites
4. **Summary Generation**: Create concise content overview

### Storage

Updated in `FILES.metadata`:

```json
{
  "metadata": {
    "document_type": "textbook",
    "difficulty": "3",
    "topics": ["Mathematics", "Calculus", "Derivatives"],
    "summary": "Comprehensive introduction to differential calculus covering limits, derivatives, and applications."
  }
}
```

---

## 📡 Monitoring Processing Status

### Status Endpoint

```
GET /api/v1/files/{file_id}/status
```

### Response

```json
{
  "file_id": "550e8400-e29b-41d4-a716-446655440000",
  "ocr_processed": true,
  "structure_processed": false,
  "metadata_extracted": true,
  "ocr_pages_count": 150,
  "structure_sections_count": 0,
  "updated_at": "2025-11-11T10:02:30.000Z"
}
```

### Status Fields

| Field                      | Type    | Description                                    |
| -------------------------- | ------- | ---------------------------------------------- |
| `ocr_processed`            | Boolean | OCR extraction complete                        |
| `metadata_extracted`       | Boolean | Classification and metadata complete           |
| `structure_processed`      | Boolean | Structure extraction complete (textbooks only) |
| `ocr_pages_count`          | Integer | Number of pages processed                      |
| `structure_sections_count` | Integer | Total sections found (if applicable)           |

### Polling Strategy

**Recommended polling interval:** 10 seconds

```python
import time
import requests

def wait_for_processing(file_id, max_wait=300):
    """Poll until processing completes or timeout"""
    start_time = time.time()

    while time.time() - start_time < max_wait:
        response = requests.get(f"http://localhost:8000/api/v1/files/{file_id}/status")
        status = response.json()

        if status['ocr_processed'] and status['metadata_extracted']:
            return True

        time.sleep(10)

    return False
```

---

## 🔧 Configuration

### Environment Variables

```env
# OpenAI Configuration
OPENAI_API_KEY=sk-proj-your-key
LLM_MODEL=gpt-4o-mini

# Mistral AI Configuration
MISTRAL_API_KEY=your-mistral-key
```

### File Size Limits

- **Maximum file size**: 50MB per document
- **Supported formats**: PDF, DOCX, PPT, PPTX
- **Page limit**: No hard limit (performance may vary with very large documents)

### Performance Tuning

**Concurrent Processing:**

- Background tasks run in parallel
- No queuing mechanism (suitable for moderate load)

**API Rate Limits:**

- Mistral AI: Check provider limits
- OpenAI: Check provider limits

---

## 🐛 Troubleshooting

### Common Issues

#### OCR Processing Failed

**Symptoms:**

- `ocr_processed` remains `false` after 5+ minutes
- No OCR_PAGES records created

**Solutions:**

1. Check Mistral AI API key validity
2. Verify PDF file is not corrupted
3. Check API rate limits
4. Review server logs for errors

#### Metadata Not Extracted

**Symptoms:**

- `metadata_extracted` is `false`
- `document_type`, `difficulty`, `topics` remain empty

**Solutions:**

1. Check OpenAI API key validity
2. Ensure OCR processing completed first
3. Verify PDF file has readable content
4. Check API rate limits

#### File Upload Failed

**Symptoms:**

- 400 Bad Request error
- File not accepted

**Solutions:**

1. Check file format (must be PDF, DOCX, PPT, or PPTX)
2. Verify file size < 50MB
3. Ensure file is not corrupted
4. Check Content-Type header

---

## 📈 Performance Metrics

### Average Processing Times

| Document Type         | Pages | OCR Time | Classification Time | Metadata Time | Total Time |
| --------------------- | ----- | -------- | ------------------- | ------------- | ---------- |
| Small (1-50 pages)    | 25    | 15s      | 5s                  | 5s            | ~25s       |
| Medium (51-150 pages) | 100   | 45s      | 7s                  | 8s            | ~60s       |
| Large (151-300 pages) | 200   | 90s      | 10s                 | 10s           | ~110s      |

### Resource Usage

- **CPU**: Moderate (mostly I/O bound)
- **Memory**: ~100MB per document during processing
- **Disk**: ~50MB per document (OCR output + images)
- **Network**: Dependent on API calls (Mistral + OpenAI)

---

## 📚 Code Examples

### Python Integration

```python
import requests
import time

class DocumentProcessor:
    def __init__(self, base_url="http://localhost:8000"):
        self.base_url = base_url
        self.api_base = f"{base_url}/api/v1"

    def upload_document(self, file_path):
        """Upload a document and wait for processing"""
        # Upload
        with open(file_path, 'rb') as f:
            response = requests.post(
                f"{self.api_base}/upload/",
                files={'file': f}
            )

        if response.status_code != 200:
            raise Exception(f"Upload failed: {response.text}")

        file_data = response.json()
        file_id = file_data['id']
        print(f"✓ Uploaded: {file_id}")

        # Wait for processing
        print("⏳ Processing...")
        while True:
            status = requests.get(f"{self.api_base}/files/{file_id}/status").json()

            print(f"  OCR: {'✓' if status['ocr_processed'] else '⏳'} | "
                  f"Metadata: {'✓' if status['metadata_extracted'] else '⏳'}")

            if status['ocr_processed'] and status['metadata_extracted']:
                break

            time.sleep(10)

        # Get final result
        result = requests.get(f"{self.api_base}/files/{file_id}").json()
        print(f"✓ Complete!")
        print(f"  Type: {result['metadata']['document_type']}")
        print(f"  Difficulty: {result['metadata']['difficulty']}")
        print(f"  Topics: {', '.join(result['metadata']['topics'])}")

        return result

# Usage
processor = DocumentProcessor()
result = processor.upload_document("textbook.pdf")
```

---

## 🔗 Related Pages

- **[System Overview](Overview)** - Complete system architecture and API reference
- **[Document Structure Parsing Service](Document-Structure-Parsing-Service)** - TOC-based structure extraction for textbooks
