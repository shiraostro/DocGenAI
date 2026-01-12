# DocGen MVP - Sprint 1 User Stories

## Sprint Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    SPRINT 1                                             │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Duration:       10 working days                                                       │
│   Team:           2 developers (full-time)                                              │
│   Capacity:       ~120 hours (2 × 10 × 6 effective hours)                              │
│   Story Points:   ~30 SP (assuming 1 SP ≈ 4 hours)                                     │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Goals:                                                                                │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   ✅ Infrastructure foundation (S3, Lambda Layer, Step Functions skeleton)             │
│   ✅ Step 1: FetchMetadata - working Lambda                                            │
│   ✅ Step 2: FetchAllFiles - working Map state with parallel file fetching             │
│   ✅ Step 3: ExtractAllOCR - working Map state with DocuPipe integration               │
│   ✅ Integration test: Steps 1-3 running end-to-end                                    │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Scope Flow:                                                                           │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│                                                                                         │
│   ┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐         │
│   │ On-Prem  │────▶│    Step 1    │────▶│    Step 2    │────▶│    Step 3    │         │
│   │   API    │     │ FetchMetadata│     │ FetchFiles   │     │  OCR Extract │         │
│   └──────────┘     └──────────────┘     └──────────────┘     └──────────────┘         │
│                                                                    │                   │
│                                                                    ▼                   │
│                                                              ┌──────────┐              │
│                                                              │    S3    │              │
│                                                              │extracted/│              │
│                                                              └──────────┘              │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Sprint Backlog Summary

| ID | User Story | Points | Owner | Priority |
|----|------------|--------|-------|----------|
| US-1.1 | Infrastructure Setup | 5 | Dev 1 | 🔴 High |
| US-1.2 | Lambda Shared Layer | 3 | Dev 1 | 🔴 High |
| US-1.3 | Step 1: FetchMetadata | 5 | Dev 2 | 🔴 High |
| US-1.4 | Step 2: FetchAllFiles | 5 | Dev 2 | 🔴 High |
| US-1.5 | Step 3: ExtractAllOCR | 8 | Dev 1 | 🔴 High |
| US-1.6 | Step Functions Orchestration | 3 | Dev 1 | 🟡 Medium |
| US-1.7 | Integration Testing | 3 | Both | 🟡 Medium |
| **Total** | | **32** | | |

---

## User Stories Detail

---

### 📋 US-1.1: Infrastructure Setup

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.1: Infrastructure Setup                                                            │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   As a: Developer                                                                       │
│   I want: Basic AWS infrastructure ready                                                │
│   So that: I can deploy and test Lambda functions                                       │
│                                                                                         │
│   Story Points: 5                                                                       │
│   Owner: Dev 1                                                                          │
│   Priority: 🔴 High (Blocker for other stories)                                        │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Acceptance Criteria:**
- [ ] S3 bucket created with correct structure
- [ ] DynamoDB Jobs table created
- [ ] IAM roles for Lambda functions created
- [ ] VPC endpoint for S3 configured (if using VPC)
- [ ] Terraform/SAM template committed to repo

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.1.1 | Create S3 bucket with lifecycle rules | 2h | Dev 1 |
| 1.1.2 | Create DynamoDB Jobs table (PK: job_id) | 2h | Dev 1 |
| 1.1.3 | Create IAM roles (Lambda execution roles) | 3h | Dev 1 |
| 1.1.4 | Create Secrets Manager secret for On-Prem API credentials | 1h | Dev 1 |
| 1.1.5 | Setup Terraform/SAM project structure | 2h | Dev 1 |
| 1.1.6 | Configure S3 VPC endpoint (Gateway) | 2h | Dev 1 |
| 1.1.7 | Document infrastructure in README | 1h | Dev 1 |

**S3 Bucket Structure:**
```
docgen-dev-bucket/
└── jobs/
    └── {job_id}/
        ├── metadata/
        ├── files/
        └── extracted/
```

**DynamoDB Jobs Table Schema:**
```
PK: job_id (String)
Attributes:
  - status: String (PROCESSING, COMPLETED, FAILED)
  - doc_type: String
  - reference_id: String
  - callback_url: String
  - created_at: String (ISO 8601)
  - updated_at: String (ISO 8601)
  - current_step: String
  - error: Map (optional)
```

---

### 📋 US-1.2: Lambda Shared Layer

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.2: Lambda Shared Layer                                                             │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   As a: Developer                                                                       │
│   I want: A shared Lambda layer with common utilities                                   │
│   So that: I can reuse code across all Lambda functions                                 │
│                                                                                         │
│   Story Points: 3                                                                       │
│   Owner: Dev 1                                                                          │
│   Priority: 🔴 High (Blocker for Lambda development)                                   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Acceptance Criteria:**
- [ ] Lambda Layer deployed and versioned
- [ ] S3 utilities working (read/write JSON, upload binary)
- [ ] Common error classes defined
- [ ] Logging configured with correlation ID
- [ ] Config module reading environment variables
- [ ] Unit tests passing (>80% coverage)

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.2.1 | Create layer project structure | 1h | Dev 1 |
| 1.2.2 | Implement s3_utils.py (read_json, write_json, upload_file) | 2h | Dev 1 |
| 1.2.3 | Implement errors.py (custom exceptions) | 1h | Dev 1 |
| 1.2.4 | Implement config.py (environment config) | 1h | Dev 1 |
| 1.2.5 | Implement logger.py (structured logging) | 1h | Dev 1 |
| 1.2.6 | Write unit tests | 2h | Dev 1 |
| 1.2.7 | Deploy layer to AWS | 1h | Dev 1 |

**Layer Structure:**
```
layers/shared/python/docgen_common/
├── __init__.py
├── s3_utils.py
├── errors.py
├── config.py
├── logger.py
└── models.py
```

**Code Example - s3_utils.py:**
```python
import json
import boto3
from typing import Any, Dict

s3 = boto3.client('s3')

def read_json_from_s3(bucket: str, key: str) -> Dict[str, Any]:
    """Read JSON file from S3"""
    response = s3.get_object(Bucket=bucket, Key=key)
    return json.loads(response['Body'].read().decode('utf-8'))

def write_json_to_s3(bucket: str, key: str, data: Dict[str, Any]) -> None:
    """Write JSON to S3"""
    s3.put_object(
        Bucket=bucket,
        Key=key,
        Body=json.dumps(data, ensure_ascii=False, indent=2),
        ContentType='application/json'
    )

def upload_binary_to_s3(bucket: str, key: str, data: bytes, content_type: str) -> None:
    """Upload binary data to S3"""
    s3.put_object(
        Bucket=bucket,
        Key=key,
        Body=data,
        ContentType=content_type
    )
```

---

### 📋 US-1.3: Step 1 - FetchMetadata Lambda

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.3: Step 1 - FetchMetadata Lambda                                                   │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   As a: System                                                                          │
│   I want: To fetch metadata and file list from On-Prem API                              │
│   So that: I know what files to process for document generation                         │
│                                                                                         │
│   Story Points: 5                                                                       │
│   Owner: Dev 2                                                                          │
│   Priority: 🔴 High                                                                     │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Flow:                                                                                 │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│                                                                                         │
│   ┌────────────┐       ┌────────────┐       ┌────────────┐       ┌────────────┐        │
│   │   Input    │──────▶│   Lambda   │──────▶│  On-Prem   │──────▶│     S3     │        │
│   │ job_id,    │       │  fetch-    │  VPN  │    API     │       │ metadata/  │        │
│   │ doc_type,  │       │  metadata  │       │            │       │            │        │
│   │ ref_id     │       └────────────┘       └────────────┘       └────────────┘        │
│   └────────────┘                                                                        │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Acceptance Criteria:**
- [ ] Lambda connects to On-Prem API via VPN
- [ ] Retrieves metadata for given reference_id
- [ ] Retrieves list of associated files
- [ ] Saves metadata to S3
- [ ] Returns file list for next step
- [ ] Handles API errors gracefully
- [ ] Timeout: 30 seconds
- [ ] Unit tests passing

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.3.1 | Create Lambda function structure | 1h | Dev 2 |
| 1.3.2 | Implement On-Prem API client | 3h | Dev 2 |
| 1.3.3 | Implement fetch_metadata handler | 3h | Dev 2 |
| 1.3.4 | Add error handling and retries | 2h | Dev 2 |
| 1.3.5 | Configure VPC settings for Lambda | 2h | Dev 2 |
| 1.3.6 | Write unit tests with mocked API | 2h | Dev 2 |
| 1.3.7 | Integration test with real API | 2h | Dev 2 |
| 1.3.8 | Deploy and test in AWS | 1h | Dev 2 |

**Input Event:**
```json
{
  "job_id": "job-abc-123",
  "doc_type": "summary_report",
  "reference_id": "case-12345",
  "callback_url": "https://onprem.local/api/docgen/callback"
}
```

**Output:**
```json
{
  "job_id": "job-abc-123",
  "doc_type": "summary_report",
  "reference_id": "case-12345",
  "callback_url": "https://onprem.local/api/docgen/callback",
  "metadata_s3_path": "s3://docgen-dev-bucket/jobs/job-abc-123/metadata/metadata.json",
  "files": [
    { "name": "doc1.pdf", "size": 1024000 },
    { "name": "doc2.pdf", "size": 2048000 },
    { "name": "doc3.pdf", "size": 512000 }
  ],
  "files_count": 3
}
```

**Handler Code:**
```python
# functions/data_fetcher/handler.py

import os
import requests
from docgen_common.s3_utils import write_json_to_s3
from docgen_common.config import get_config
from docgen_common.logger import get_logger
from docgen_common.errors import OnPremAPIError

config = get_config()
logger = get_logger()

def fetch_metadata(event, context):
    """
    Step 1: Fetch metadata and file list from On-Prem API
    """
    job_id = event['job_id']
    reference_id = event['reference_id']
    doc_type = event['doc_type']
    
    logger.info(f"Fetching metadata for job {job_id}, reference {reference_id}")
    
    try:
        # Call On-Prem API
        api_url = f"{config.ONPREM_API_URL}/api/documents/{reference_id}"
        headers = {
            "Authorization": f"Bearer {config.ONPREM_API_TOKEN}",
            "Content-Type": "application/json"
        }
        
        response = requests.get(api_url, headers=headers, timeout=30)
        response.raise_for_status()
        
        data = response.json()
        
        # Extract metadata and file list
        metadata = {
            "reference_id": reference_id,
            "doc_type": doc_type,
            "title": data.get("title"),
            "created_at": data.get("created_at"),
            "additional_data": data.get("data", {})
        }
        
        files = [
            {"name": f["name"], "size": f.get("size", 0)}
            for f in data.get("files", [])
        ]
        
        # Save metadata to S3
        metadata_key = f"jobs/{job_id}/metadata/metadata.json"
        write_json_to_s3(config.BUCKET_NAME, metadata_key, metadata)
        
        logger.info(f"Found {len(files)} files for job {job_id}")
        
        return {
            **event,
            "metadata_s3_path": f"s3://{config.BUCKET_NAME}/{metadata_key}",
            "files": files,
            "files_count": len(files)
        }
        
    except requests.exceptions.RequestException as e:
        logger.error(f"Failed to fetch metadata: {e}")
        raise OnPremAPIError(f"Failed to fetch metadata: {e}")
```

---

### 📋 US-1.4: Step 2 - FetchAllFiles Lambda (Map State)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.4: Step 2 - FetchAllFiles Lambda (Map State)                                       │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   As a: System                                                                          │
│   I want: To download all source files from On-Prem API                                 │
│   So that: I can process them with OCR                                                  │
│                                                                                         │
│   Story Points: 5                                                                       │
│   Owner: Dev 2                                                                          │
│   Priority: 🔴 High                                                                     │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Flow (Map State - Parallel):                                                          │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│                                                                                         │
│   ┌──────────┐       ┌────────────────────────────────────────────────────────┐        │
│   │  files   │       │  MAP STATE (maxConcurrency: 5)                         │        │
│   │  array   │──────▶│                                                        │        │
│   └──────────┘       │  ┌─────────┐    ┌────────────┐    ┌────────────┐      │        │
│                      │  │ file_1  │───▶│   Lambda   │───▶│     S3     │      │        │
│                      │  ├─────────┤    │ fetch_file │    │   files/   │      │        │
│                      │  │ file_2  │───▶│            │───▶│            │      │        │
│                      │  ├─────────┤    └────────────┘    └────────────┘      │        │
│                      │  │  ...    │                                           │        │
│                      │  └─────────┘                                           │        │
│                      └────────────────────────────────────────────────────────┘        │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Acceptance Criteria:**
- [ ] Lambda fetches single file as BASE64 from On-Prem API
- [ ] Decodes BASE64 and uploads to S3
- [ ] Works correctly in Map state (parallel execution)
- [ ] Handles large files (up to 10MB)
- [ ] Returns S3 path for each file
- [ ] Timeout: 60 seconds per file
- [ ] Unit tests passing

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.4.1 | Implement fetch_file handler | 3h | Dev 2 |
| 1.4.2 | Implement BASE64 decoding and S3 upload | 2h | Dev 2 |
| 1.4.3 | Add retry logic for failed downloads | 2h | Dev 2 |
| 1.4.4 | Handle large files (streaming if needed) | 2h | Dev 2 |
| 1.4.5 | Write unit tests | 2h | Dev 2 |
| 1.4.6 | Test with Step Functions Map state | 2h | Dev 2 |
| 1.4.7 | Performance test with 50 files | 1h | Dev 2 |

**Input Event (Single File - from Map state):**
```json
{
  "job_id": "job-abc-123",
  "reference_id": "case-12345",
  "file": {
    "name": "doc1.pdf",
    "size": 1024000
  }
}
```

**Output (Single File):**
```json
{
  "job_id": "job-abc-123",
  "file_name": "doc1.pdf",
  "s3_path": "s3://docgen-dev-bucket/jobs/job-abc-123/files/doc1.pdf",
  "size_bytes": 1024000,
  "status": "SUCCESS"
}
```

**Handler Code:**
```python
# functions/data_fetcher/handler.py (continued)

import base64

def fetch_file(event, context):
    """
    Step 2: Fetch single file from On-Prem API (BASE64) and upload to S3
    Called in parallel via Step Functions Map state
    """
    job_id = event['job_id']
    reference_id = event['reference_id']
    file_info = event['file']
    file_name = file_info['name']
    
    logger.info(f"Fetching file {file_name} for job {job_id}")
    
    try:
        # Call On-Prem API to get file as BASE64
        api_url = f"{config.ONPREM_API_URL}/api/documents/{reference_id}/files/{file_name}"
        headers = {
            "Authorization": f"Bearer {config.ONPREM_API_TOKEN}",
            "Accept": "application/json"
        }
        
        response = requests.get(api_url, headers=headers, timeout=60)
        response.raise_for_status()
        
        data = response.json()
        base64_content = data['content']
        
        # Decode BASE64
        file_bytes = base64.b64decode(base64_content)
        
        # Upload to S3
        s3_key = f"jobs/{job_id}/files/{file_name}"
        content_type = "application/pdf" if file_name.endswith('.pdf') else "application/octet-stream"
        
        upload_binary_to_s3(config.BUCKET_NAME, s3_key, file_bytes, content_type)
        
        logger.info(f"Uploaded {file_name} to S3 ({len(file_bytes)} bytes)")
        
        return {
            "job_id": job_id,
            "file_name": file_name,
            "s3_path": f"s3://{config.BUCKET_NAME}/{s3_key}",
            "size_bytes": len(file_bytes),
            "status": "SUCCESS"
        }
        
    except Exception as e:
        logger.error(f"Failed to fetch file {file_name}: {e}")
        return {
            "job_id": job_id,
            "file_name": file_name,
            "status": "FAILED",
            "error": str(e)
        }
```

---

### 📋 US-1.5: Step 3 - ExtractAllOCR Lambda (Map State)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.5: Step 3 - ExtractAllOCR Lambda (Map State)                                       │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   As a: System                                                                          │
│   I want: To extract text from all PDF files using DocuPipe OCR                         │
│   So that: I can process the text content with AI                                       │
│                                                                                         │
│   Story Points: 8                                                                       │
│   Owner: Dev 1                                                                          │
│   Priority: 🔴 High                                                                     │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Flow (Map State - Parallel with Polling):                                             │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│                                                                                         │
│   ┌──────────┐       ┌────────────────────────────────────────────────────────┐        │
│   │  files   │       │  MAP STATE (maxConcurrency: 5)                         │        │
│   │  array   │──────▶│                                                        │        │
│   └──────────┘       │  ┌─────────┐    ┌────────────┐    ┌────────────┐      │        │
│                      │  │ file_1  │───▶│   Lambda   │───▶│  DocuPipe  │      │        │
│                      │  ├─────────┤    │ extract_ocr│    │    API     │      │        │
│                      │  │ file_2  │───▶│            │    └─────┬──────┘      │        │
│                      │  ├─────────┤    │ (submit +  │          │             │        │
│                      │  │  ...    │───▶│   poll)    │◀─────────┘             │        │
│                      │  └─────────┘    └─────┬──────┘                        │        │
│                      │                       │                                │        │
│                      │                       ▼                                │        │
│                      │                 ┌────────────┐                        │        │
│                      │                 │     S3     │                        │        │
│                      │                 │ extracted/ │                        │        │
│                      │                 └────────────┘                        │        │
│                      └────────────────────────────────────────────────────────┘        │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Acceptance Criteria:**
- [ ] Lambda submits PDF to DocuPipe API
- [ ] Polls for completion (with timeout)
- [ ] Retrieves extracted text
- [ ] Saves extracted text as JSON to S3
- [ ] Handles DocuPipe errors gracefully
- [ ] Circuit breaker after 5 consecutive failures
- [ ] Timeout: 5 minutes per file
- [ ] Unit tests passing

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.5.1 | Create Lambda function structure | 1h | Dev 1 |
| 1.5.2 | Implement DocuPipe API client | 4h | Dev 1 |
| 1.5.3 | Implement submit_document function | 2h | Dev 1 |
| 1.5.4 | Implement poll_status function | 2h | Dev 1 |
| 1.5.5 | Implement extract_ocr handler with polling loop | 4h | Dev 1 |
| 1.5.6 | Add circuit breaker pattern | 3h | Dev 1 |
| 1.5.7 | Save extracted text to S3 | 2h | Dev 1 |
| 1.5.8 | Write unit tests with mocked DocuPipe | 3h | Dev 1 |
| 1.5.9 | Integration test with real DocuPipe | 2h | Dev 1 |
| 1.5.10 | Performance test with multiple files | 2h | Dev 1 |

**Input Event (Single File - from Map state):**
```json
{
  "job_id": "job-abc-123",
  "file_name": "doc1.pdf",
  "s3_path": "s3://docgen-dev-bucket/jobs/job-abc-123/files/doc1.pdf"
}
```

**Output (Single File):**
```json
{
  "job_id": "job-abc-123",
  "file_name": "doc1.pdf",
  "extracted_s3_path": "s3://docgen-dev-bucket/jobs/job-abc-123/extracted/doc1.json",
  "pages_count": 15,
  "word_count": 3500,
  "status": "SUCCESS"
}
```

**Extracted JSON Structure (saved to S3):**
```json
{
  "file_name": "doc1.pdf",
  "pages": [
    {
      "page_number": 1,
      "text": "Page 1 content...",
      "tables": []
    },
    {
      "page_number": 2,
      "text": "Page 2 content...",
      "tables": [
        {
          "rows": [["Header1", "Header2"], ["Cell1", "Cell2"]]
        }
      ]
    }
  ],
  "full_text": "Complete document text...",
  "metadata": {
    "pages_count": 15,
    "word_count": 3500,
    "extracted_at": "2025-01-12T14:30:00Z"
  }
}
```

**Handler Code:**
```python
# functions/ocr_processor/handler.py

import time
import requests
from docgen_common.s3_utils import read_binary_from_s3, write_json_to_s3
from docgen_common.config import get_config
from docgen_common.logger import get_logger
from docgen_common.errors import OCRExtractionError

config = get_config()
logger = get_logger()

# Circuit breaker state (in real implementation, use Redis/DynamoDB)
circuit_breaker = {
    "failures": 0,
    "last_failure": None,
    "state": "CLOSED"  # CLOSED, OPEN, HALF_OPEN
}

def extract_ocr(event, context):
    """
    Step 3: Extract text from PDF using DocuPipe OCR
    Called in parallel via Step Functions Map state
    """
    job_id = event['job_id']
    file_name = event['file_name']
    s3_path = event['s3_path']
    
    logger.info(f"Starting OCR extraction for {file_name} in job {job_id}")
    
    # Check circuit breaker
    if circuit_breaker["state"] == "OPEN":
        raise OCRExtractionError("Circuit breaker is OPEN - DocuPipe unavailable")
    
    try:
        # 1. Read PDF from S3
        bucket, key = parse_s3_path(s3_path)
        pdf_bytes = read_binary_from_s3(bucket, key)
        
        # 2. Submit to DocuPipe
        docupipe_job_id = submit_to_docupipe(pdf_bytes, file_name)
        
        # 3. Poll for completion
        result = poll_docupipe_status(docupipe_job_id, timeout_seconds=300)
        
        # 4. Parse extracted content
        extracted_data = {
            "file_name": file_name,
            "pages": result.get("pages", []),
            "full_text": result.get("full_text", ""),
            "metadata": {
                "pages_count": len(result.get("pages", [])),
                "word_count": len(result.get("full_text", "").split()),
                "extracted_at": datetime.utcnow().isoformat() + "Z"
            }
        }
        
        # 5. Save to S3
        extracted_key = f"jobs/{job_id}/extracted/{file_name.replace('.pdf', '.json')}"
        write_json_to_s3(config.BUCKET_NAME, extracted_key, extracted_data)
        
        # Reset circuit breaker on success
        circuit_breaker["failures"] = 0
        circuit_breaker["state"] = "CLOSED"
        
        logger.info(f"OCR completed for {file_name}: {extracted_data['metadata']['pages_count']} pages")
        
        return {
            "job_id": job_id,
            "file_name": file_name,
            "extracted_s3_path": f"s3://{config.BUCKET_NAME}/{extracted_key}",
            "pages_count": extracted_data["metadata"]["pages_count"],
            "word_count": extracted_data["metadata"]["word_count"],
            "status": "SUCCESS"
        }
        
    except Exception as e:
        # Update circuit breaker
        circuit_breaker["failures"] += 1
        if circuit_breaker["failures"] >= 5:
            circuit_breaker["state"] = "OPEN"
            logger.error("Circuit breaker OPENED after 5 failures")
        
        logger.error(f"OCR extraction failed for {file_name}: {e}")
        return {
            "job_id": job_id,
            "file_name": file_name,
            "status": "FAILED",
            "error": str(e)
        }


def submit_to_docupipe(pdf_bytes: bytes, file_name: str) -> str:
    """Submit PDF to DocuPipe API and return job ID"""
    
    url = f"{config.DOCUPIPE_API_URL}/v1/documents"
    headers = {
        "Authorization": f"Bearer {config.DOCUPIPE_API_KEY}",
    }
    files = {
        "file": (file_name, pdf_bytes, "application/pdf")
    }
    
    response = requests.post(url, headers=headers, files=files, timeout=60)
    response.raise_for_status()
    
    return response.json()["job_id"]


def poll_docupipe_status(docupipe_job_id: str, timeout_seconds: int = 300) -> dict:
    """Poll DocuPipe until job completes or timeout"""
    
    url = f"{config.DOCUPIPE_API_URL}/v1/documents/{docupipe_job_id}"
    headers = {
        "Authorization": f"Bearer {config.DOCUPIPE_API_KEY}",
    }
    
    start_time = time.time()
    poll_interval = 5  # seconds
    
    while time.time() - start_time < timeout_seconds:
        response = requests.get(url, headers=headers, timeout=30)
        response.raise_for_status()
        
        data = response.json()
        status = data.get("status")
        
        if status == "COMPLETED":
            return data.get("result", {})
        elif status == "FAILED":
            raise OCRExtractionError(f"DocuPipe job failed: {data.get('error')}")
        
        # Still processing, wait and retry
        time.sleep(poll_interval)
    
    raise OCRExtractionError(f"DocuPipe job timed out after {timeout_seconds} seconds")
```

---

### 📋 US-1.6: Step Functions Orchestration

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.6: Step Functions Orchestration                                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   As a: Developer                                                                       │
│   I want: Step Functions state machine connecting Steps 1-3                             │
│   So that: The pipeline runs automatically end-to-end                                   │
│                                                                                         │
│   Story Points: 3                                                                       │
│   Owner: Dev 1                                                                          │
│   Priority: 🟡 Medium                                                                   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Acceptance Criteria:**
- [ ] State machine definition (ASL) created
- [ ] Steps 1-3 connected with correct input/output
- [ ] Map states configured with maxConcurrency
- [ ] Error handling with Catch blocks
- [ ] Deployed to AWS
- [ ] Can be triggered via AWS Console

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.6.1 | Write ASL definition for Steps 1-3 | 3h | Dev 1 |
| 1.6.2 | Configure Map states with correct ItemsPath | 2h | Dev 1 |
| 1.6.3 | Add error handling (Catch, Retry) | 2h | Dev 1 |
| 1.6.4 | Create IAM role for Step Functions | 1h | Dev 1 |
| 1.6.5 | Deploy and test via Console | 2h | Dev 1 |

**ASL Definition (Steps 1-3):**
```json
{
  "Comment": "DocGen MVP - Steps 1-3",
  "StartAt": "FetchMetadata",
  "States": {
    "FetchMetadata": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:docgen-data-fetcher",
      "Parameters": {
        "action": "fetch_metadata",
        "job_id.$": "$.job_id",
        "doc_type.$": "$.doc_type",
        "reference_id.$": "$.reference_id",
        "callback_url.$": "$.callback_url"
      },
      "ResultPath": "$",
      "Next": "FetchAllFiles",
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "HandleError"
        }
      ]
    },
    
    "FetchAllFiles": {
      "Type": "Map",
      "ItemsPath": "$.files",
      "MaxConcurrency": 5,
      "Parameters": {
        "job_id.$": "$.job_id",
        "reference_id.$": "$.reference_id",
        "file.$": "$$.Map.Item.Value"
      },
      "Iterator": {
        "StartAt": "FetchSingleFile",
        "States": {
          "FetchSingleFile": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:docgen-data-fetcher",
            "Parameters": {
              "action": "fetch_file",
              "job_id.$": "$.job_id",
              "reference_id.$": "$.reference_id",
              "file.$": "$.file"
            },
            "End": true,
            "Retry": [
              {
                "ErrorEquals": ["States.TaskFailed"],
                "IntervalSeconds": 2,
                "MaxAttempts": 2
              }
            ]
          }
        }
      },
      "ResultPath": "$.fetched_files",
      "Next": "ExtractAllOCR",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "HandleError"
        }
      ]
    },
    
    "ExtractAllOCR": {
      "Type": "Map",
      "ItemsPath": "$.fetched_files",
      "MaxConcurrency": 5,
      "Parameters": {
        "job_id.$": "$.job_id",
        "file_name.$": "$$.Map.Item.Value.file_name",
        "s3_path.$": "$$.Map.Item.Value.s3_path"
      },
      "Iterator": {
        "StartAt": "ExtractSingleOCR",
        "States": {
          "ExtractSingleOCR": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:docgen-ocr-processor",
            "Parameters": {
              "job_id.$": "$.job_id",
              "file_name.$": "$.file_name",
              "s3_path.$": "$.s3_path"
            },
            "End": true,
            "Retry": [
              {
                "ErrorEquals": ["States.TaskFailed"],
                "IntervalSeconds": 5,
                "MaxAttempts": 2
              }
            ]
          }
        }
      },
      "ResultPath": "$.extracted_files",
      "Next": "Sprint1Complete"
    },
    
    "Sprint1Complete": {
      "Type": "Succeed"
    },
    
    "HandleError": {
      "Type": "Fail",
      "Error": "ProcessingFailed",
      "Cause.$": "$.error"
    }
  }
}
```

---

### 📋 US-1.7: Integration Testing

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.7: Integration Testing                                                             │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   As a: Developer                                                                       │
│   I want: End-to-end integration tests for Steps 1-3                                    │
│   So that: I can verify the pipeline works correctly                                    │
│                                                                                         │
│   Story Points: 3                                                                       │
│   Owner: Both                                                                           │
│   Priority: 🟡 Medium                                                                   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Acceptance Criteria:**
- [ ] Test with 1 file - passes
- [ ] Test with 10 files - passes
- [ ] Test with 50 files - passes
- [ ] Test with corrupted PDF - handles error gracefully
- [ ] Test with missing file - handles error gracefully
- [ ] All tests documented

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.7.1 | Create test data (sample PDFs) | 1h | Dev 2 |
| 1.7.2 | Write integration test script | 3h | Dev 1 |
| 1.7.3 | Test happy path (1, 10, 50 files) | 2h | Both |
| 1.7.4 | Test error scenarios | 2h | Both |
| 1.7.5 | Document test results | 1h | Dev 2 |
| 1.7.6 | Fix bugs found in testing | 3h | Both |

---

## Sprint Timeline

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                  SPRINT TIMELINE                                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Day    │ Dev 1                          │ Dev 2                                      │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   1      │ US-1.1: Infrastructure (S3,    │ US-1.3: FetchMetadata - API client         │
│          │ DynamoDB, IAM)                 │ research & implementation                  │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   2      │ US-1.1: Infrastructure         │ US-1.3: FetchMetadata - handler            │
│          │ (complete)                     │ implementation                             │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   3      │ US-1.2: Lambda Layer           │ US-1.3: FetchMetadata - tests              │
│          │ (s3_utils, errors)             │ & deployment                               │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   4      │ US-1.2: Lambda Layer           │ US-1.4: FetchAllFiles - implementation     │
│          │ (complete & deploy)            │                                            │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   5      │ US-1.5: ExtractAllOCR -        │ US-1.4: FetchAllFiles - BASE64 handling    │
│          │ DocuPipe client                │ & S3 upload                                │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   6      │ US-1.5: ExtractAllOCR -        │ US-1.4: FetchAllFiles - tests              │
│          │ submit & poll logic            │ & deployment                               │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   7      │ US-1.5: ExtractAllOCR -        │ US-1.6: Step Functions ASL                 │
│          │ circuit breaker                │ (assist Dev 1)                             │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   8      │ US-1.5: ExtractAllOCR -        │ US-1.6: Step Functions deployment          │
│          │ tests & deployment             │ & testing                                  │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   9      │ US-1.7: Integration testing    │ US-1.7: Integration testing                │
│          │ (50 files test)                │ (error scenarios)                          │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   10     │ Bug fixes & documentation      │ Bug fixes & documentation                  │
│          │ Sprint review                  │ Sprint review                              │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Definition of Done

- [ ] Code reviewed and approved
- [ ] Unit tests passing (>80% coverage)
- [ ] Integration tests passing
- [ ] Deployed to dev environment
- [ ] Documentation updated
- [ ] No critical bugs open

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| On-Prem API not ready | High | Use mock API for development |
| DocuPipe rate limits | Medium | Implement circuit breaker, adjust maxConcurrency |
| VPN connectivity issues | High | Test VPN early in sprint |
| Large files slow to process | Medium | Add progress logging, increase timeouts |

---

## Dependencies

| Dependency | Owner | Status |
|------------|-------|--------|
| On-Prem API endpoint ready | On-Prem Team | ❓ To verify |
| DocuPipe API credentials | Platform Team | ❓ To verify |
| VPN configured | Network Team | ❓ To verify |
| AWS account access | DevOps | ✅ Ready |
