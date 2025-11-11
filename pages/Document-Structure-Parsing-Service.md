# Document Structure Parsing Service

**Version:** 1.0.2  
**Last Updated:** November 11, 2025

---

## 📋 Service Overview

The Document Structure Parsing Service provides intelligent table of contents (TOC) extraction and hierarchical structure mapping for textbook documents. It uses a TOC-first approach combined with queue-based page processing to accurately identify sections and their page ranges.

### Service Capabilities

- ✅ Automatic TOC detection and extraction
- ✅ AI-powered TOC parsing (GPT-4)
- ✅ Hierarchical section structure mapping
- ✅ Precise page range assignment
- ✅ Multi-level section numbering support (1.2.3, I.II.III)
- ✅ OCR edge case handling (split titles, missing markers)

### Limitations

⚠️ **Textbooks Only**: This service is designed specifically for textbook documents with a table of contents. Slides and syllabi are not supported.

---

## 🔄 Service Architecture

### Processing Flow

```
┌──────────────────┐
│  1. TOC Detection│
│  - Scan first 10%│
│  - Find keywords │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  2. LLM Parsing  │
│  - GPT-4 Analysis│
│  - Extract titles│
│  - Get page nums │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  3. Queue Setup  │
│  - Flatten TOC   │
│  - First-order   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  4. Page Scan    │
│  - Single pass   │
│  - Match titles  │
│  - Assign ranges │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  5. MongoDB Save │
│  - Update FILES  │
│  - Store sections│
└──────────────────┘
```

### Algorithm Details

**Key Innovation:** TOC-first extraction ensures accurate page ranges by:

1. Extracting the complete table of contents upfront
2. Processing pages sequentially in a single pass
3. Matching actual titles to TOC entries
4. Accumulating content until the next section is found

---

## 🧮 Structure Extraction Algorithm

### Step 1: TOC Detection

**Location Strategy:**

- Scans the **first 10%** of document pages
- Looks for TOC keywords: "Table of Contents", "Contents", "Syllabus"
- Supports multi-page TOCs

**Example TOC Detection:**

```markdown
# Table of Contents

1. Introduction ..................... 1
2. Basic Concepts .................. 15
   2.1 Definitions ................. 16
   2.2 Examples .................... 20
3. Advanced Topics ................. 35
```

### Step 2: LLM-Powered Parsing

**Input:** Raw TOC text  
**Model:** OpenAI GPT-4o-mini  
**Output:** Structured JSON

**Prompt Strategy:**

```
Extract the table of contents from the following text.
Return a JSON array with objects containing:
- title: The section title
- page: The page number
- level: The hierarchy level (1, 2, 3...)
```

**Example Output:**

```json
[
  { "title": "Introduction", "page": 1, "level": 1 },
  { "title": "Basic Concepts", "page": 15, "level": 1 },
  { "title": "Definitions", "page": 16, "level": 2 },
  { "title": "Examples", "page": 20, "level": 2 },
  { "title": "Advanced Topics", "page": 35, "level": 1 }
]
```

### Step 3: Queue-Based Processing

**Flattening Strategy:**

Original hierarchy:

```
1. Introduction
   1.1 Overview
   1.2 Goals
2. Main Content
   2.1 Section A
```

Flattened queue (first-order):

```
Queue: [Introduction, Overview, Goals, Main Content, Section A]
```

**Why Flatten?**

- Simplifies page scanning (no recursive traversal)
- Ensures every section is processed exactly once
- Makes content accumulation straightforward

### Step 4: Title Matching

**Two-Method Approach:**

#### Method 1: Markdown Headers

**Pattern:** Lines starting with `#` symbol

```python
if line.strip().startswith('#'):
    clean_title = line.strip().lstrip('#').strip()
```

**Example:**

```markdown
# Introduction

### 1.1 Overview
```

#### Method 2: Plain Text with Strict Validation

**Requirements:**

1. ✓ Must be on its own line (no extra words)
2. ✓ May have leading section numbers
3. ✓ Word count must match ±2 words of expected title

**Section Number Patterns Supported:**

| Pattern        | Examples               | Regex                                           |
| -------------- | ---------------------- | ----------------------------------------------- |
| Single level   | `1`, `2`, `3`          | `^\s*[\dIVXLCDM]+[\.\)]*\s+`                    |
| Multi-level    | `1.2`, `9.5`, `1.2.3`  | `^\s*(?:[\dIVXLCDM]+\.)*[\dIVXLCDM]+[\.\)]*\s+` |
| Roman numerals | `I`, `II`, `III`, `IV` | Same regex                                      |
| Mixed          | `I.II.III`, `1.A.2`    | Same regex                                      |

**Validation Logic:**

```python
def is_valid_plain_text_title(line, expected_title):
    # Remove section numbers
    clean_line = re.sub(r'^\s*(?:[\dIVXLCDM]+\.)*[\dIVXLCDM]+[\.\)]*\s+', '', line)

    # Must be on own line (no extra words)
    if len(clean_line.split()) > len(expected_title.split()) + 2:
        return False

    # Fuzzy match (80% similarity)
    if similarity(clean_line, expected_title) >= 0.8:
        return True

    return False
```

**Examples:**

✅ Valid Matches:

```
1. Introduction                    → "Introduction"
9.5 Advanced Calculus             → "Advanced Calculus"
III. Historical Context           → "Historical Context"
Chapter 2: Basic Concepts         → "Basic Concepts"
```

❌ Invalid Matches:

```
Introduction to the basic concepts of...  → Too many extra words
See Introduction on page 5                → Extra words before/after
```

### Step 5: Page Range Assignment

**Single-Pass Algorithm:**

```python
queue = flatten_sections(toc)
current_section_idx = 0
current_section_start_page = None

for page_num in range(total_pages):
    page_text = get_page_text(page_num)

    # Check if next section starts here
    if current_section_idx < len(queue):
        next_section = queue[current_section_idx]

        if title_found_in_page(next_section.title, page_text):
            # Save previous section's range
            if current_section_start_page is not None:
                save_range(current_section_idx - 1,
                          current_section_start_page,
                          page_num - 1)

            # Start new section
            current_section_start_page = page_num
            current_section_idx += 1

# Handle last section (accumulate all remaining pages)
if current_section_start_page is not None:
    save_range(current_section_idx - 1,
              current_section_start_page,
              total_pages - 1)
```

**Key Features:**

- **Single Pass**: Each page is scanned exactly once
- **Last Section Handling**: Automatically accumulates all remaining pages
- **Content Accumulation**: Content between sections is assigned to the previous section

---

## 📤 Structure Extraction Endpoint

### API Endpoint

```
GET /api/v1/files/{file_id}/structure
```

### Request

**Path Parameters:**

| Parameter | Type | Required | Description                          |
| --------- | ---- | -------- | ------------------------------------ |
| `file_id` | UUID | Yes      | File identifier from upload response |

**Example:**

```bash
curl -X GET "http://localhost:8000/api/v1/files/{file_id}/structure"
```

### Response

**Status Code:** `200 OK`

**Response Body:**

```json
{
  "file_id": "550e8400-e29b-41d4-a716-446655440000",
  "sections": [
    {
      "title": "Introduction",
      "page_number": 1,
      "level": 1,
      "page_range": [1, 14],
      "children": [
        {
          "title": "Overview",
          "page_number": 2,
          "level": 2,
          "page_range": [2, 8],
          "children": []
        },
        {
          "title": "Goals",
          "page_number": 9,
          "level": 2,
          "page_range": [9, 14],
          "children": []
        }
      ]
    },
    {
      "title": "Main Content",
      "page_number": 15,
      "level": 1,
      "page_range": [15, 149],
      "children": []
    }
  ],
  "total_sections": 4,
  "structure_processed": true,
  "updated_at": "2025-11-11T10:05:00.000Z"
}
```

**Field Descriptions:**

| Field                 | Type       | Description                                         |
| --------------------- | ---------- | --------------------------------------------------- |
| `file_id`             | UUID       | Document identifier                                 |
| `sections`            | Array      | Top-level sections with nested children             |
| `title`               | String     | Section title                                       |
| `page_number`         | Integer    | TOC page number (where section starts)              |
| `level`               | Integer    | Hierarchy level (1 = chapter, 2 = subsection, etc.) |
| `page_range`          | Array[Int] | `[start_page, end_page]` inclusive                  |
| `children`            | Array      | Nested subsections                                  |
| `total_sections`      | Integer    | Total count (including nested)                      |
| `structure_processed` | Boolean    | Processing complete flag                            |

### Error Responses

| Status Code | Error                 | Description                                    |
| ----------- | --------------------- | ---------------------------------------------- |
| `404`       | Not Found             | File ID does not exist                         |
| `400`       | Bad Request           | Document is not a textbook or OCR not complete |
| `500`       | Internal Server Error | Processing error                               |

**Example Error:**

```json
{
  "detail": "Structure extraction only available for textbook documents"
}
```

---

## 🔍 When to Use Structure Extraction

### Supported Document Types

✅ **Textbooks:**

- Academic textbooks with TOC
- Technical manuals with chapter structure
- Educational materials with section hierarchy

❌ **Not Supported:**

- Slides (no continuous page flow)
- Syllabi (no hierarchical structure)
- Documents without table of contents

### Prerequisites

Before requesting structure extraction, ensure:

1. ✓ Document uploaded successfully
2. ✓ OCR processing complete (`ocr_processed: true`)
3. ✓ Document classified as "textbook"
4. ✓ Document contains a table of contents

**Verification:**

```bash
# Check if ready for structure extraction
curl -X GET "http://localhost:8000/api/v1/files/{file_id}/status"

# Response should show:
# {
#   "ocr_processed": true,
#   "metadata_extracted": true,
#   "document_type": "textbook"
# }
```

---

## 📊 Performance & Accuracy

### Processing Time

| Document Size | Pages | TOC Pages | Processing Time |
| ------------- | ----- | --------- | --------------- |
| Small         | 50    | 1-2       | ~10s            |
| Medium        | 150   | 2-3       | ~25s            |
| Large         | 300   | 3-5       | ~50s            |

**Factors Affecting Speed:**

- Total page count
- TOC length (number of sections)
- Text density per page
- OCR quality

### Accuracy Metrics

**Title Matching Accuracy:**

- Markdown headers: **~98%** (high confidence)
- Plain text titles: **~85%** (depends on OCR quality)

**Page Range Accuracy:**

- **~95%** when TOC page numbers are correct
- **~80%** when OCR has formatting issues

**Common Failure Cases:**

1. TOC page numbers don't match actual pages (OCR error)
2. Section titles in content differ significantly from TOC
3. Split titles across multiple lines (partially mitigated)
4. Missing section number patterns

---

## 🐛 Troubleshooting

### Structure Extraction Failed

**Symptoms:**

- `structure_processed` remains `false`
- No sections returned

**Solutions:**

1. **Check Document Type:**

   ```bash
   curl -X GET "http://localhost:8000/api/v1/files/{file_id}"
   # Verify: metadata.document_type == "textbook"
   ```

2. **Verify TOC Exists:**

   - Manually check if document has a table of contents
   - Ensure TOC is in the first 10% of pages

3. **Check OCR Quality:**

   ```bash
   # Download OCR text from first few pages
   curl -X GET "http://localhost:8000/api/v1/files/{file_id}/pages/0/text"
   ```

   - Look for TOC keywords: "Table of Contents", "Contents"
   - Verify text is readable (not garbled)

4. **Review Server Logs:**
   - Check for LLM parsing errors
   - Look for title matching issues

### Incorrect Page Ranges

**Symptoms:**

- Sections have wrong start/end pages
- Page ranges overlap
- Missing sections

**Causes & Solutions:**

| Cause                          | Solution                                                 |
| ------------------------------ | -------------------------------------------------------- |
| TOC page numbers incorrect     | Manual verification required                             |
| Title not found in content     | Check OCR quality, verify title spelling                 |
| Split titles (across lines)    | Algorithm partially handles this; may need manual review |
| Multiple sections on same page | Expected behavior; ranges may overlap                    |

**Verification Steps:**

```python
# Check page range assignments
response = requests.get(f"http://localhost:8000/api/v1/files/{file_id}/structure")
structure = response.json()

for section in structure['sections']:
    print(f"{section['title']}: pages {section['page_range'][0]}-{section['page_range'][1]}")

    # Download actual page to verify
    page_text = requests.get(
        f"http://localhost:8000/api/v1/files/{file_id}/pages/{section['page_range'][0]}/text"
    ).text

    # Check if title appears in text
    if section['title'].lower() in page_text.lower():
        print("✓ Title found")
    else:
        print("✗ Title NOT found - possible mismatch")
```

### Missing Sections

**Symptoms:**

- Some sections from TOC are missing in output
- `total_sections` count is low

**Causes:**

1. Title matching failed (too strict validation)
2. Section number pattern not recognized
3. Title significantly different in content vs. TOC

**Solutions:**

1. **Check Title Variations:**

   - TOC: "Chapter 1: Introduction"
   - Content: "1. Introduction" or "Introduction"
   - Algorithm should handle this, but verify

2. **Review Section Numbers:**

   - Ensure pattern matches regex: `^\s*(?:[\dIVXLCDM]+\.)*[\dIVXLCDM]+[\.\)]*\s+`
   - Custom patterns may need algorithm updates

3. **Adjust Similarity Threshold:**
   - Default: 80% fuzzy match
   - May need fine-tuning for specific documents

---

## 🔧 Configuration

### Environment Variables

```env
# OpenAI Configuration (for TOC parsing)
OPENAI_API_KEY=sk-proj-your-key
LLM_MODEL=gpt-4o-mini
```

### Algorithm Parameters

**TOC Detection:**

```python
# Scan first 10% of pages for TOC
toc_scan_percentage = 0.1

# TOC keywords
toc_keywords = ["table of contents", "contents", "syllabus"]
```

**Title Matching:**

```python
# Fuzzy match threshold
similarity_threshold = 0.8

# Word count tolerance for plain text validation
word_count_tolerance = 2

# Section number regex
section_number_pattern = r'^\s*(?:[\dIVXLCDM]+\.)*[\dIVXLCDM]+[\.\)]*\s+'
```

---

## 📈 Advanced Features

### Handling Complex TOCs

**Multi-Level Numbering:**

Supports:

- `1.2.3` (decimal)
- `I.II.III` (Roman numerals)
- `1.A.2` (mixed)
- `9.5` (non-contiguous)

**Example:**

```
1. Part One
   1.1 Chapter 1
       1.1.1 Section A
       1.1.2 Section B
   1.2 Chapter 2
2. Part Two
```

**Flattened Queue:**

```
[Part One, Chapter 1, Section A, Section B, Chapter 2, Part Two]
```

### OCR Edge Case Handling

**Split Titles:**

OCR Output:

```
9.5 Advanced
Calculus
```

Expected:

```
9.5 Advanced Calculus
```

**Mitigation:**

- Check next line if current line is too short
- Combine lines if similarity improves

**Titles Without Markers:**

```
Introduction
```

(no section number, no markdown #)

**Handling:**

- Use plain text validation
- Require strict "on its own line" rule

---

## 📚 Code Examples

### Python Integration

```python
import requests
import time

class StructureExtractor:
    def __init__(self, base_url="http://localhost:8000"):
        self.base_url = base_url
        self.api_base = f"{base_url}/api/v1"

    def extract_structure(self, file_id):
        """Extract document structure with polling"""

        # Check if ready
        status = requests.get(f"{self.api_base}/files/{file_id}/status").json()

        if not status['ocr_processed']:
            raise Exception("OCR processing not complete. Please wait.")

        # Trigger structure extraction (happens automatically on first request)
        print("🔍 Extracting structure...")

        # Poll until complete
        while True:
            status = requests.get(f"{self.api_base}/files/{file_id}/status").json()

            if status['structure_processed']:
                break

            print(f"  Sections found: {status['structure_sections_count']}")
            time.sleep(5)

        # Get final structure
        response = requests.get(f"{self.api_base}/files/{file_id}/structure")
        structure = response.json()

        print(f"✓ Complete! Found {structure['total_sections']} sections")
        return structure

    def print_structure(self, sections, indent=0):
        """Pretty-print hierarchical structure"""
        for section in sections:
            prefix = "  " * indent
            page_range = f"{section['page_range'][0]}-{section['page_range'][1]}"
            print(f"{prefix}• {section['title']} (pages {page_range})")

            if section['children']:
                self.print_structure(section['children'], indent + 1)

# Usage
extractor = StructureExtractor()

# Extract structure
structure = extractor.extract_structure(file_id)

# Print hierarchy
print("\n📖 Document Structure:")
extractor.print_structure(structure['sections'])
```

**Output Example:**

```
🔍 Extracting structure...
  Sections found: 8
  Sections found: 15
✓ Complete! Found 23 sections

📖 Document Structure:
• Introduction (pages 1-14)
  • Overview (pages 2-8)
  • Goals (pages 9-14)
• Basic Concepts (pages 15-50)
  • Definitions (pages 16-25)
  • Examples (pages 26-50)
• Advanced Topics (pages 51-149)
```

### Navigation Example

```python
def navigate_to_section(file_id, section_title):
    """Find section and retrieve its content"""

    # Get structure
    response = requests.get(f"http://localhost:8000/api/v1/files/{file_id}/structure")
    structure = response.json()

    # Search for section (recursive)
    def find_section(sections, title):
        for section in sections:
            if section['title'].lower() == title.lower():
                return section
            if section['children']:
                result = find_section(section['children'], title)
                if result:
                    return result
        return None

    section = find_section(structure['sections'], section_title)

    if not section:
        print(f"Section '{section_title}' not found")
        return None

    # Download pages in range
    start, end = section['page_range']
    print(f"📖 {section['title']} (pages {start}-{end})")

    content = []
    for page_num in range(start, end + 1):
        page_text = requests.get(
            f"http://localhost:8000/api/v1/files/{file_id}/pages/{page_num}/text"
        ).text
        content.append(page_text)

    return '\n\n'.join(content)

# Usage
content = navigate_to_section(file_id, "Advanced Topics")
print(content[:500])  # First 500 chars
```

---

## 🔗 Related Pages

- **[System Overview](Overview)** - Complete system architecture and API reference
- **[Upload and Metadata Extraction Service](Upload-and-Metadata-Extraction-Service)** - Document upload and OCR processing
