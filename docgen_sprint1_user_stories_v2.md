# DocGen MVP - Sprint 1 User Stories (v2)

## Changes from v1
- **NEW STEP 1**: GetBuildInstructions - retrieves template + API endpoints from DynamoDB
- **Step numbering updated**: FetchMetadata → Step 2, FetchAllFiles → Step 3, ExtractAllOCR → Step 4
- **DynamoDB Templates table**: Added to infrastructure setup
- **FetchMetadata**: Now receives api_endpoints from Step 1 (no hardcoded URLs)
- **Total Story Points**: 34 SP (was 32 SP)

---

## Sprint Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    SPRINT 1 (v2)                                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Duration:       10 working days                                                       │
│   Team:           2 developers (full-time)                                              │
│   Capacity:       ~120 hours (2 × 10 × 6 effective hours)                              │
│   Story Points:   ~34 SP (assuming 1 SP ≈ 4 hours)                                     │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Goals:                                                                                │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   ✅ Infrastructure foundation (S3, DynamoDB x2, Lambda Layer, Step Functions)         │
│   ✅ Step 1: GetBuildInstructions - fetch template + API endpoints from DynamoDB       │
│   ✅ Step 2: FetchMetadata - working Lambda with dynamic API endpoints                 │
│   ✅ Step 3: FetchAllFiles - working Map state with parallel file fetching             │
│   ✅ Step 4: ExtractAllOCR - working Map state with DocuPipe integration               │
│   ✅ Integration test: Steps 1-4 running end-to-end                                    │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Scope Flow (Updated per Architecture v4):                                             │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│                                                                                         │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐           │
│   │    Step 1    │──▶│    Step 2    │──▶│    Step 3    │──▶│    Step 4    │           │
│   │GetBuildInstr │   │ FetchMetadata│   │ FetchFiles   │   │  OCR Extract │           │
│   │  (DynamoDB)  │   │  (On-Prem)   │   │  (On-Prem)   │   │  (DocuPipe)  │           │
│   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘           │
│          │                                                         │                   │
│          ▼                                                         ▼                   │
│    ┌──────────┐                                              ┌──────────┐              │
│    │ DynamoDB │                                              │    S3    │              │
│    │Templates │                                              │extracted/│              │
│    └──────────┘                                              └──────────┘              │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Sprint Backlog Summary

| ID | User Story | Points | Owner | Priority |
|----|------------|--------|-------|----------|
| US-1.1 | Infrastructure Setup | 6 | Dev 1 | 🔴 High |
| US-1.2 | Lambda Shared Layer | 3 | Dev 1 | 🔴 High |
| US-1.3 | **Step 1: GetBuildInstructions** ⭐ NEW | 2 | Dev 1 | 🔴 High |
| US-1.4 | Step 2: FetchMetadata | 5 | Dev 2 | 🔴 High |
| US-1.5 | Step 3: FetchAllFiles | 5 | Dev 2 | 🔴 High |
| US-1.6 | Step 4: ExtractAllOCR | 8 | Dev 1 | 🔴 High |
| US-1.7 | Step Functions Orchestration | 3 | Dev 1 | 🟡 Medium |
| US-1.8 | Integration Testing | 2 | Both | 🟡 Medium |
| **Total** | | **34** | | |

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
│   Story Points: 6 (was 5, +1 for Templates table)                                      │
│   Owner: Dev 1                                                                          │
│   Priority: 🔴 High (Blocker for other stories)                                        │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Acceptance Criteria:**
- [ ] S3 bucket created with correct structure
- [ ] DynamoDB Jobs table created
- [ ] ⭐ **DynamoDB Templates table created with api_endpoints** (NEW)
- [ ] IAM roles for Lambda functions created
- [ ] VPC endpoint for S3 configured (if using VPC)
- [ ] Terraform/SAM template committed to repo

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.1.1 | Create S3 bucket with lifecycle rules | 2h | Dev 1 |
| 1.1.2 | Create DynamoDB Jobs table (PK: job_id) | 2h | Dev 1 |
| 1.1.3 | ⭐ **Create DynamoDB Templates table (PK: doc_type)** | 2h | Dev 1 |
| 1.1.4 | ⭐ **Populate Templates table with test data** | 1h | Dev 1 |
| 1.1.5 | Create IAM roles (Lambda execution roles) | 3h | Dev 1 |
| 1.1.6 | Create Secrets Manager secret for On-Prem API credentials | 1h | Dev 1 |
| 1.1.7 | Setup Terraform/SAM project structure | 2h | Dev 1 |
| 1.1.8 | Configure S3 VPC endpoint (Gateway) | 2h | Dev 1 |
| 1.1.9 | Document infrastructure in README | 1h | Dev 1 |

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

**⭐ DynamoDB Templates Table Schema (NEW - per Architecture v4):**
```
PK: doc_type (String)
Attributes:
  - sections: List<String>        # ["מבוא", "רקע", "ממצאים", "המלצות"]
  - instructions: Map             # { "מבוא": "...", "רקע": "...", ... }
  - api_endpoints: Map {
      metadata_url: String        # "https://onprem.local/api/cases/{id}/metadata"
      files_url: String           # "https://onprem.local/api/cases/{id}/files"
      file_content_url: String    # "https://onprem.local/api/files/{file_id}/content"
    }
  - output_format: String         # "docx" | "pdf"
  - created_at: String
  - updated_at: String
```

**Example Templates Data:**
```json
{
  "doc_type": "summary_report",
  "sections": ["מבוא", "רקע", "ממצאים", "המלצות"],
  "instructions": {
    "מבוא": "סכם את מטרת המסמך ב-2-3 משפטים",
    "רקע": "תאר את הרקע לתיק בהתבסס על המסמכים",
    "ממצאים": "רשום את הממצאים העיקריים בנקודות",
    "המלצות": "הצג המלצות בהתבסס על הממצאים"
  },
  "api_endpoints": {
    "metadata_url": "https://onprem.company.local/api/cases/{reference_id}/metadata",
    "files_url": "https://onprem.company.local/api/cases/{reference_id}/files",
    "file_content_url": "https://onprem.company.local/api/files/{file_id}/content"
  },
  "output_format": "docx"
}
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
- [ ] ⭐ **DynamoDB utilities working (get_template)** (NEW)
- [ ] Common error classes defined
- [ ] Logging configured with correlation ID
- [ ] Config module reading environment variables
- [ ] Unit tests passing (>80% coverage)

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.2.1 | Create layer project structure | 1h | Dev 1 |
| 1.2.2 | Implement s3_utils.py (read_json, write_json, upload_file) | 2h | Dev 1 |
| 1.2.3 | ⭐ **Implement dynamodb_utils.py (get_template)** | 1h | Dev 1 |
| 1.2.4 | Implement errors.py (custom exceptions) | 1h | Dev 1 |
| 1.2.5 | Implement config.py (environment config) | 1h | Dev 1 |
| 1.2.6 | Implement logger.py (structured logging) | 1h | Dev 1 |
| 1.2.7 | Write unit tests | 2h | Dev 1 |
| 1.2.8 | Deploy layer to AWS | 1h | Dev 1 |

**Layer Structure:**
```
layers/shared/python/docgen_common/
├── __init__.py
├── s3_utils.py
├── dynamodb_utils.py    # ⭐ NEW
├── errors.py
├── config.py
├── logger.py
└── models.py
```

**Code Example - dynamodb_utils.py (NEW):**
```python
import boto3
from typing import Dict, Any, Optional
from .errors import TemplateNotFoundError

dynamodb = boto3.resource('dynamodb')

def get_template(doc_type: str, table_name: str = "docgen-templates") -> Dict[str, Any]:
    """
    Retrieve template configuration from DynamoDB
    Returns: sections, instructions, api_endpoints
    """
    table = dynamodb.Table(table_name)
    
    response = table.get_item(Key={"doc_type": doc_type})
    
    if "Item" not in response:
        raise TemplateNotFoundError(f"Template not found for doc_type: {doc_type}")
    
    return response["Item"]
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

### 📋 US-1.3: Step 1 - GetBuildInstructions Lambda ⭐ NEW

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.3: Step 1 - GetBuildInstructions Lambda ⭐ NEW                                     │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   As a: System                                                                          │
│   I want: To fetch template configuration including API endpoints                       │
│   So that: Subsequent steps know which APIs to call for this doc_type                   │
│                                                                                         │
│   Story Points: 2                                                                       │
│   Owner: Dev 1                                                                          │
│   Priority: 🔴 High (First step in pipeline)                                           │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Flow:                                                                                 │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│                                                                                         │
│   ┌────────────┐       ┌────────────────────┐       ┌────────────┐                     │
│   │   Input    │──────▶│ docgen-doc-builder │──────▶│  DynamoDB  │                     │
│   │ doc_type   │       │  .get_template()   │       │  Templates │                     │
│   └────────────┘       └────────────────────┘       └────────────┘                     │
│                                 │                                                       │
│                                 ▼                                                       │
│                        ┌────────────────────┐                                          │
│                        │ Output: sections,  │                                          │
│                        │ instructions,      │                                          │
│                        │ api_endpoints      │                                          │
│                        └────────────────────┘                                          │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Acceptance Criteria:**
- [ ] Lambda reads template from DynamoDB Templates table
- [ ] Returns sections, instructions, and api_endpoints
- [ ] Handles missing template error gracefully
- [ ] Timeout: 10 seconds
- [ ] Unit tests passing

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.3.1 | Create Lambda function structure | 1h | Dev 1 |
| 1.3.2 | Implement get_template handler | 2h | Dev 1 |
| 1.3.3 | Add error handling for missing templates | 1h | Dev 1 |
| 1.3.4 | Write unit tests | 2h | Dev 1 |
| 1.3.5 | Deploy and test | 1h | Dev 1 |

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
  "template": {
    "sections": ["מבוא", "רקע", "ממצאים", "המלצות"],
    "instructions": {
      "מבוא": "סכם את מטרת המסמך ב-2-3 משפטים",
      "רקע": "תאר את הרקע לתיק בהתבסס על המסמכים",
      "ממצאים": "רשום את הממצאים העיקריים בנקודות",
      "המלצות": "הצע המלצות בהתבסס על הממצאים"
    },
    "api_endpoints": {
      "metadata_url": "https://onprem.company.local/api/cases/{reference_id}/metadata",
      "files_url": "https://onprem.company.local/api/cases/{reference_id}/files",
      "file_content_url": "https://onprem.company.local/api/files/{file_id}/content"
    },
    "output_format": "docx"
  }
}
```

**Handler Code:**
```python
# functions/doc_builder/handler.py

from docgen_common.dynamodb_utils import get_template
from docgen_common.config import get_config
from docgen_common.logger import get_logger
from docgen_common.errors import TemplateNotFoundError

config = get_config()
logger = get_logger()

def get_build_instructions(event, context):
    """
    Step 1: Get template configuration from DynamoDB
    Returns: sections, instructions, api_endpoints for the given doc_type
    """
    job_id = event['job_id']
    doc_type = event['doc_type']
    
    logger.info(f"Getting template for job {job_id}, doc_type {doc_type}")
    
    try:
        template = get_template(doc_type)
        
        logger.info(f"Template found with {len(template.get('sections', []))} sections")
        
        return {
            **event,
            "template": {
                "sections": template.get("sections", []),
                "instructions": template.get("instructions", {}),
                "api_endpoints": template.get("api_endpoints", {}),
                "output_format": template.get("output_format", "docx")
            }
        }
        
    except TemplateNotFoundError as e:
        logger.error(f"Template not found for doc_type {doc_type}: {e}")
        raise
```

---

### 📋 US-1.4: Step 2 - FetchMetadata Lambda (Updated)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.4: Step 2 - FetchMetadata Lambda                                                   │
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
│   ⭐ Key Change: Now receives api_endpoints from Step 1 (no hardcoded URLs)            │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│                                                                                         │
│   ┌────────────┐       ┌────────────┐       ┌────────────┐       ┌────────────┐        │
│   │   Input    │──────▶│   Lambda   │──────▶│  On-Prem   │──────▶│     S3     │        │
│   │ job_id,    │       │  fetch-    │  VPN  │    API     │       │ metadata/  │        │
│   │ doc_type,  │       │  metadata  │       │            │       │            │        │
│   │ ref_id,    │       └────────────┘       └────────────┘       └────────────┘        │
│   │⭐api_endpts│                                                                        │
│   └────────────┘                                                                        │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Acceptance Criteria:**
- [ ] Lambda connects to On-Prem API via VPN
- [ ] ⭐ **Uses api_endpoints from Step 1 (not hardcoded)** (CHANGED)
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
| 1.4.1 | Create Lambda function structure | 1h | Dev 2 |
| 1.4.2 | Implement On-Prem API client | 3h | Dev 2 |
| 1.4.3 | ⭐ **Implement dynamic URL handling from api_endpoints** | 1h | Dev 2 |
| 1.4.4 | Implement fetch_metadata handler | 3h | Dev 2 |
| 1.4.5 | Add error handling and retries | 2h | Dev 2 |
| 1.4.6 | Configure VPC settings for Lambda | 2h | Dev 2 |
| 1.4.7 | Write unit tests with mocked API | 2h | Dev 2 |
| 1.4.8 | Integration test with real API | 2h | Dev 2 |
| 1.4.9 | Deploy and test in AWS | 1h | Dev 2 |

**Input Event (Updated - now includes template from Step 1):**
```json
{
  "job_id": "job-abc-123",
  "doc_type": "summary_report",
  "reference_id": "case-12345",
  "callback_url": "https://onprem.local/api/docgen/callback",
  "template": {
    "sections": ["מבוא", "רקע", "ממצאים", "המלצות"],
    "instructions": { ... },
    "api_endpoints": {
      "metadata_url": "https://onprem.company.local/api/cases/{reference_id}/metadata",
      "files_url": "https://onprem.company.local/api/cases/{reference_id}/files",
      "file_content_url": "https://onprem.company.local/api/files/{file_id}/content"
    }
  }
}
```

**Output:**
```json
{
  "job_id": "job-abc-123",
  "doc_type": "summary_report",
  "reference_id": "case-12345",
  "callback_url": "https://onprem.local/api/docgen/callback",
  "template": { ... },
  "metadata_s3_path": "s3://docgen-dev-bucket/jobs/job-abc-123/metadata/metadata.json",
  "files": [
    { "file_id": "file-001", "name": "doc1.pdf", "size": 1024000 },
    { "file_id": "file-002", "name": "doc2.pdf", "size": 2048000 },
    { "file_id": "file-003", "name": "doc3.pdf", "size": 512000 }
  ],
  "files_count": 3
}
```

**Handler Code (Updated):**
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
    Step 2: Fetch metadata and file list from On-Prem API
    ⭐ Uses api_endpoints from Step 1 (dynamic URLs)
    """
    job_id = event['job_id']
    reference_id = event['reference_id']
    doc_type = event['doc_type']
    
    # ⭐ Get API endpoints from template (received from Step 1)
    api_endpoints = event.get('template', {}).get('api_endpoints', {})
    
    if not api_endpoints:
        raise OnPremAPIError("No api_endpoints provided from Step 1")
    
    logger.info(f"Fetching metadata for job {job_id}, reference {reference_id}")
    
    try:
        # ⭐ Build URL dynamically from template
        metadata_url = api_endpoints['metadata_url'].replace('{reference_id}', reference_id)
        files_url = api_endpoints['files_url'].replace('{reference_id}', reference_id)
        
        headers = {
            "Authorization": f"Bearer {config.ONPREM_API_TOKEN}",
            "Content-Type": "application/json"
        }
        
        # Fetch metadata
        metadata_response = requests.get(metadata_url, headers=headers, timeout=30)
        metadata_response.raise_for_status()
        metadata = metadata_response.json()
        
        # Fetch file list
        files_response = requests.get(files_url, headers=headers, timeout=30)
        files_response.raise_for_status()
        files_data = files_response.json()
        
        files = [
            {
                "file_id": f["id"],
                "name": f["name"],
                "size": f.get("size", 0)
            }
            for f in files_data.get("files", [])
        ]
        
        # Save metadata to S3
        metadata_key = f"jobs/{job_id}/metadata/metadata.json"
        write_json_to_s3(config.BUCKET_NAME, metadata_key, {
            "reference_id": reference_id,
            "doc_type": doc_type,
            "data": metadata
        })
        
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

### 📋 US-1.5: Step 3 - FetchAllFiles Lambda (Map State)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.5: Step 3 - FetchAllFiles Lambda (Map State)                                       │
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
│   ⭐ Key Change: Uses file_content_url from api_endpoints                              │
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
- [ ] ⭐ **Uses file_content_url from api_endpoints** (CHANGED)
- [ ] Decodes BASE64 and uploads to S3
- [ ] Works correctly in Map state (parallel execution)
- [ ] Handles large files (up to 10MB)
- [ ] Returns S3 path for each file
- [ ] Timeout: 60 seconds per file
- [ ] Unit tests passing

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.5.1 | Implement fetch_file handler | 3h | Dev 2 |
| 1.5.2 | ⭐ **Implement dynamic URL from file_content_url** | 1h | Dev 2 |
| 1.5.3 | Implement BASE64 decoding and S3 upload | 2h | Dev 2 |
| 1.5.4 | Add retry logic for failed downloads | 2h | Dev 2 |
| 1.5.5 | Handle large files (streaming if needed) | 2h | Dev 2 |
| 1.5.6 | Write unit tests | 2h | Dev 2 |
| 1.5.7 | Test with Step Functions Map state | 2h | Dev 2 |
| 1.5.8 | Performance test with 50 files | 1h | Dev 2 |

**Input Event (Single File - from Map state):**
```json
{
  "job_id": "job-abc-123",
  "reference_id": "case-12345",
  "file": {
    "file_id": "file-001",
    "name": "doc1.pdf",
    "size": 1024000
  },
  "file_content_url": "https://onprem.company.local/api/files/{file_id}/content"
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

**Handler Code (Updated):**
```python
# functions/data_fetcher/handler.py (continued)

import base64

def fetch_file(event, context):
    """
    Step 3: Fetch single file from On-Prem API (BASE64) and upload to S3
    Called in parallel via Step Functions Map state
    ⭐ Uses file_content_url from api_endpoints (dynamic)
    """
    job_id = event['job_id']
    file_info = event['file']
    file_id = file_info['file_id']
    file_name = file_info['name']
    
    # ⭐ Build URL dynamically from template
    file_content_url_template = event.get('file_content_url', '')
    api_url = file_content_url_template.replace('{file_id}', file_id)
    
    logger.info(f"Fetching file {file_name} (id: {file_id}) for job {job_id}")
    
    try:
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

### 📋 US-1.6: Step 4 - ExtractAllOCR Lambda (Map State)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.6: Step 4 - ExtractAllOCR Lambda (Map State)                                       │
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
| 1.6.1 | Create Lambda function structure | 1h | Dev 1 |
| 1.6.2 | Implement DocuPipe API client | 4h | Dev 1 |
| 1.6.3 | Implement submit_document function | 2h | Dev 1 |
| 1.6.4 | Implement poll_status function | 2h | Dev 1 |
| 1.6.5 | Implement extract_ocr handler with polling loop | 4h | Dev 1 |
| 1.6.6 | Add circuit breaker pattern | 3h | Dev 1 |
| 1.6.7 | Save extracted text to S3 | 2h | Dev 1 |
| 1.6.8 | Write unit tests with mocked DocuPipe | 3h | Dev 1 |
| 1.6.9 | Integration test with real DocuPipe | 2h | Dev 1 |
| 1.6.10 | Performance test with multiple files | 2h | Dev 1 |

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
from datetime import datetime
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
    Step 4: Extract text from PDF using DocuPipe OCR
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


def parse_s3_path(s3_path: str) -> tuple:
    """Parse s3://bucket/key into (bucket, key)"""
    path = s3_path.replace("s3://", "")
    parts = path.split("/", 1)
    return parts[0], parts[1]
```

---

### 📋 US-1.7: Step Functions Orchestration (Updated)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.7: Step Functions Orchestration                                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   As a: Developer                                                                       │
│   I want: Step Functions state machine connecting Steps 1-4                             │
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
- [ ] ⭐ **Steps 1-4 connected with correct input/output** (CHANGED from 1-3)
- [ ] Map states configured with maxConcurrency
- [ ] Error handling with Catch blocks
- [ ] Deployed to AWS
- [ ] Can be triggered via AWS Console

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.7.1 | ⭐ **Write ASL definition for Steps 1-4** | 3h | Dev 1 |
| 1.7.2 | Configure Map states with correct ItemsPath | 2h | Dev 1 |
| 1.7.3 | Add error handling (Catch, Retry) | 2h | Dev 1 |
| 1.7.4 | Create IAM role for Step Functions | 1h | Dev 1 |
| 1.7.5 | Deploy and test via Console | 2h | Dev 1 |

**ASL Definition (Steps 1-4 - Updated):**
```json
{
  "Comment": "DocGen MVP - Steps 1-4 (Sprint 1)",
  "StartAt": "GetBuildInstructions",
  "States": {
    "GetBuildInstructions": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:docgen-doc-builder",
      "Parameters": {
        "action": "get_template",
        "job_id.$": "$.job_id",
        "doc_type.$": "$.doc_type",
        "reference_id.$": "$.reference_id",
        "callback_url.$": "$.callback_url"
      },
      "ResultPath": "$",
      "Next": "FetchMetadata",
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

    "FetchMetadata": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:docgen-data-fetcher",
      "Parameters": {
        "action": "fetch_metadata",
        "job_id.$": "$.job_id",
        "doc_type.$": "$.doc_type",
        "reference_id.$": "$.reference_id",
        "callback_url.$": "$.callback_url",
        "template.$": "$.template"
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
        "file.$": "$$.Map.Item.Value",
        "file_content_url.$": "$.template.api_endpoints.file_content_url"
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
              "file.$": "$.file",
              "file_content_url.$": "$.file_content_url"
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

### 📋 US-1.8: Integration Testing

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ US-1.8: Integration Testing                                                             │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   As a: Developer                                                                       │
│   I want: End-to-end integration tests for Steps 1-4                                    │
│   So that: I can verify the pipeline works correctly                                    │
│                                                                                         │
│   Story Points: 2 (was 3, reduced due to step count)                                   │
│   Owner: Both                                                                           │
│   Priority: 🟡 Medium                                                                   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Acceptance Criteria:**
- [ ] ⭐ **Test template retrieval from DynamoDB** (NEW)
- [ ] Test with 1 file - passes
- [ ] Test with 10 files - passes
- [ ] Test with 50 files - passes
- [ ] Test with corrupted PDF - handles error gracefully
- [ ] Test with missing file - handles error gracefully
- [ ] Test with missing template - handles error gracefully (NEW)
- [ ] All tests documented

**Tasks:**

| Task | Description | Hours | Owner |
|------|-------------|-------|-------|
| 1.8.1 | Create test data (sample PDFs) | 1h | Dev 2 |
| 1.8.2 | ⭐ **Populate test templates in DynamoDB** | 0.5h | Dev 1 |
| 1.8.3 | Write integration test script | 2h | Dev 1 |
| 1.8.4 | Test happy path (1, 10, 50 files) | 1.5h | Both |
| 1.8.5 | Test error scenarios | 1.5h | Both |
| 1.8.6 | Document test results | 0.5h | Dev 2 |
| 1.8.7 | Fix bugs found in testing | 2h | Both |

---

## Sprint Timeline (Updated)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                  SPRINT TIMELINE (v2)                                   │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Day    │ Dev 1                          │ Dev 2                                      │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   1      │ US-1.1: Infrastructure (S3,    │ US-1.4: FetchMetadata - API client         │
│          │ DynamoDB Jobs & Templates)     │ research & implementation                  │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   2      │ US-1.1: Infrastructure         │ US-1.4: FetchMetadata - handler            │
│          │ (IAM, Secrets, complete)       │ with dynamic api_endpoints                 │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   3      │ US-1.2: Lambda Layer           │ US-1.4: FetchMetadata - tests              │
│          │ (s3_utils, dynamodb_utils)     │ & deployment                               │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   4      │ US-1.2: Lambda Layer +         │ US-1.5: FetchAllFiles - implementation     │
│          │ US-1.3: GetBuildInstructions   │                                            │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   5      │ US-1.6: ExtractAllOCR -        │ US-1.5: FetchAllFiles - BASE64 handling    │
│          │ DocuPipe client                │ & S3 upload                                │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   6      │ US-1.6: ExtractAllOCR -        │ US-1.5: FetchAllFiles - tests              │
│          │ submit & poll logic            │ & deployment                               │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   7      │ US-1.6: ExtractAllOCR -        │ US-1.7: Step Functions ASL                 │
│          │ circuit breaker                │ (assist Dev 1)                             │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   8      │ US-1.6: ExtractAllOCR -        │ US-1.7: Step Functions deployment          │
│          │ tests & deployment             │ & testing                                  │
│   ───────┼────────────────────────────────┼────────────────────────────────────────────│
│   9      │ US-1.8: Integration testing    │ US-1.8: Integration testing                │
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
- [ ] ⭐ **Templates table populated with test data** (NEW)
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
| ⭐ **Templates not defined** | Medium | Create default templates early in sprint |
| ⭐ **api_endpoints schema changes** | Low | Version templates in DynamoDB |

---

## Dependencies

| Dependency | Owner | Status |
|------------|-------|--------|
| On-Prem API endpoint ready | On-Prem Team | ❓ To verify |
| On-Prem API returns file_id | On-Prem Team | ❓ To verify |
| DocuPipe API credentials | Platform Team | ❓ To verify |
| VPN configured | Network Team | ❓ To verify |
| AWS account access | DevOps | ✅ Ready |
| ⭐ **Templates schema finalized** | Product Team | ❓ To define |

---

## Summary of Changes from v1

| Aspect | v1 | v2 |
|--------|----|----|
| Total Steps | 3 | 4 |
| Total Story Points | 32 | 34 |
| DynamoDB Tables | 1 (Jobs) | 2 (Jobs + Templates) |
| Step 1 | FetchMetadata | GetBuildInstructions |
| API URLs | Hardcoded in config | Dynamic from Templates |
| Template handling | N/A | From DynamoDB |
| file_id support | No | Yes |
