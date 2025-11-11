# AI Document Processing System - Overview

**Version:** 1.0.2  
**Last Updated:** November 11, 2025

---

## 📋 System Overview

The AI Document Processing System is a comprehensive backend service that transforms educational documents into structured, searchable, and AI-ready formats. The system supports multiple document types (PDF, DOCX, PPT, PPTX) and provides rich content extraction capabilities.

### Key Capabilities

- **Multi-Format Support**: PDF, DOCX, PPT, PPTX
- **Advanced OCR**: Mistral AI-powered text and formula extraction
- **Smart Classification**: AI-driven document type detection
- **Metadata Extraction**: Automatic difficulty, topics, and summary generation
- **Structure Extraction**: Hierarchical TOC parsing for textbooks
- **Image Processing**: Extraction and storage of document images

---

## 🏗️ System Architecture

### Architecture Diagram

```
┌─────────────┐
│ File Upload │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│ Document        │
│ Converter       │
│ (PDF Storage)   │
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│ Background      │
│ Processing      │
│ (Async Tasks)   │
└──────┬──────────┘
       │
       ├──────────────────┬──────────────────┬──────────────────┐
       ▼                  ▼                  ▼                  ▼
┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│   OCR    │      │Document  │      │Metadata  │      │Structure │
│Processing│      │Classifier│      │Extractor │      │ Parser   │
└────┬─────┘      └────┬─────┘      └────┬─────┘      └────┬─────┘
     │                 │                 │                 │
     └─────────────────┴─────────────────┴─────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ MongoDB Storage  │
                    │  - FILES         │
                    │  - OCR_PAGES     │
                    │  - IMAGES        │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  API Responses   │
                    └──────────────────┘
```

### Technology Stack

| Component               | Technology                       |
| ----------------------- | -------------------------------- |
| **Backend Framework**   | FastAPI (Python)                 |
| **Database**            | MongoDB Atlas                    |
| **OCR Engine**          | Mistral AI API                   |
| **LLM**                 | OpenAI GPT-4                     |
| **Document Conversion** | PyPDF2, python-docx, python-pptx |
| **Async Processing**    | FastAPI BackgroundTasks          |

### Processing Pipeline

1. **Upload & Convert** → Documents converted to PDF format
2. **OCR Processing** → Extract text, images, and formulas (30-60s)
3. **Classification** → Identify document type (5-10s)
4. **Metadata Extraction** → Generate difficulty, topics, summary (5-10s)
5. **Structure Extraction** → Parse TOC and build hierarchical tree for textbooks (30-60s)
6. **Storage** → Save structured data to MongoDB

**Total Processing Time:** ~1-2 minutes for typical documents

---

## 🔌 API Endpoints

### Base URL

```
http://localhost:8000/api/v1
```

### Endpoints Summary

| Method | Endpoint                                    | Description             | Input                               | Output                               |
| ------ | ------------------------------------------- | ----------------------- | ----------------------------------- | ------------------------------------ |
| `POST` | `/upload/`                                  | Upload document         | `file: multipart/form-data`         | File metadata with `file_id`         |
| `GET`  | `/files/{file_id}`                          | Get complete file info  | `file_id: string`                   | Complete file document with metadata |
| `GET`  | `/files/{file_id}/status`                   | Check processing status | `file_id: string`                   | Processing status flags              |
| `GET`  | `/files/{file_id}/structure`                | Get document structure  | `file_id: string`                   | Hierarchical structure tree          |
| `GET`  | `/files/{file_id}/ocr-pages`                | Get all OCR pages       | `file_id: string`                   | List of OCR-processed pages          |
| `GET`  | `/files/{file_id}/pages/{page_number}`      | Get specific page       | `file_id: string, page_number: int` | Single page content                  |
| `GET`  | `/files/{file_id}/pages/{page_number}/text` | Download page markdown  | `file_id: string, page_number: int` | Markdown file download               |
| `GET`  | `/files/{file_id}/images`                   | Get all images          | `file_id: string`                   | List of extracted images             |
| `GET`  | `/images/{image_id}`                        | Download image          | `image_id: string`                  | Image file download                  |

### API Documentation

- **Swagger UI**: `http://localhost:8000/docs`
- **ReDoc**: `http://localhost:8000/redoc`
- **OpenAPI JSON**: `http://localhost:8000/openapi.json`

---

## ⚙️ System Requirements

### Prerequisites

- **Python**: 3.10.11 or higher
- **MongoDB Atlas**: Account and cluster setup
- **API Keys**:
  - OpenAI GPT-4 API key
  - Mistral AI API key

### Environment Configuration

Required environment variables:

```env
# OpenAI Configuration
OPENAI_API_KEY=sk-proj-your-key
LLM_MODEL=gpt-4o-mini

# Mistral AI Configuration
MISTRAL_API_KEY=your-mistral-key

# MongoDB Configuration (in mongodb_config.json)
{
  "mongodb": {
    "host": "cluster.mongodb.net",
    "username": "your-username",
    "password": "your-password"
  }
}
```

### Resource Requirements

- **Disk Space**: 50MB+ per document
- **Memory**: 2GB+ RAM recommended
- **Network**: Stable internet for API calls

---

## 📚 Core Services

The system provides two main processing services:

### 1. Upload and Metadata Extraction Service

Handles file upload, OCR processing, document classification, and metadata generation.

**[→ View Detailed Documentation](Upload-and-Metadata-Extraction-Service)**

**Key Features:**

- Multi-format document upload
- Mistral AI OCR processing
- Document type classification (slides/textbook/syllabus)
- Automatic metadata generation (difficulty, topics, summary)
- Image extraction and storage

### 2. Document Structure Parsing Service

Extracts hierarchical structure from textbooks with Table of Contents.

**[→ View Detailed Documentation](Document-Structure-Parsing-Service)**

**Key Features:**

- TOC detection and parsing
- Hierarchical chapter/section organization
- Title matching with fuzzy algorithms
- Page range assignment
- Content extraction

> ⚠️ **Note**: Structure extraction only works for textbooks with Table of Contents

---

## 🚀 Quick Start

### 1. Upload Document

```bash
curl -X POST "http://localhost:8000/api/v1/upload/" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@document.pdf"
```

**Response:**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "metadata": {
    "file_name": "document.pdf",
    "num_pages": 150,
    "ocr_processed": false,
    "structure_processed": false
  }
}
```

### 2. Check Processing Status

```bash
curl "http://localhost:8000/api/v1/files/{file_id}/status"
```

**Response:**

```json
{
  "file_id": "550e8400-e29b-41d4-a716-446655440000",
  "ocr_processed": true,
  "structure_processed": true,
  "metadata_extracted": true,
  "ocr_pages_count": 150,
  "structure_sections_count": 25
}
```

### 3. Get Results

```bash
# Get complete file info
curl "http://localhost:8000/api/v1/files/{file_id}"

# Get document structure (textbooks only)
curl "http://localhost:8000/api/v1/files/{file_id}/structure"
```

---

## 📊 MongoDB Collections

### FILES Collection

Stores complete document metadata and structure

**Key Fields:**

- `_id`: Unique file identifier (UUID string)
- `content`: Combined OCR markdown from all pages
- `metadata.document_type`: slides | textbook | syllabus
- `metadata.structure`: Hierarchical structure array
- `metadata.ocr_processed`: Boolean flag
- `metadata.structure_processed`: Boolean flag

### OCR_PAGES Collection

Stores individual page OCR results

**Key Fields:**

- `file_id`: Reference to FILES.\_id
- `page_number`: 0-based page index
- `md_path`: Path to markdown file
- `images`: Array of extracted image paths

### IMAGES Collection

Stores extracted images metadata

**Key Fields:**

- `image_id`: Unique image identifier
- `file_id`: Reference to FILES.\_id
- `page_index`: Page where image appears
- `image_path`: Path to saved image file

---

## 📞 Support

**Interactive Documentation**: `http://localhost:8000/docs`

**GitHub Repository**: [VT-AI-Tutor/ai-doc-processing](https://github.com/VT-AI-Tutor/ai-doc-processing)

---

## 📚 Related Pages

- **[Upload and Metadata Extraction Service](Upload-and-Metadata-Extraction-Service)** - Detailed documentation for file upload, OCR, and metadata extraction
- **[Document Structure Parsing Service](Document-Structure-Parsing-Service)** - Detailed documentation for TOC-based structure extraction
