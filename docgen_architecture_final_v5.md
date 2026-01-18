# DocGen MVP - Step Functions Architecture (Final v5)

## Changes from v4
- **job_id**: Generated from `$$.Execution.Name` (Step Functions built-in)
- **callback_url**: Environment variable (not per-request)
- **State Flow**: Detailed I/O for each step with `ResultPath`
- **S3 JSON Structures**: Defined for each step
- **ASL Definition**: Complete Step Functions definition included
- **files[]**: Simplified to URL strings only

---

## Environment Variables

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              ENVIRONMENT VARIABLES                                       │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Variable              │ Description                      │ Example                    │
│   ──────────────────────┼──────────────────────────────────┼────────────────────────────│
│   S3_BUCKET             │ Bucket for all job data          │ docgen-dev-bucket          │
│   CALLBACK_URL          │ On-Prem callback endpoint        │ https://onprem.local/api/  │
│                         │                                  │ docgen/callback            │
│   BEDROCK_MODEL_ID      │ Claude model ID                  │ anthropic.claude-3-sonnet- │
│                         │                                  │ 20240229-v1:0              │
│   DOCUPIPE_API_URL      │ OCR service URL                  │ https://docupipe.api/v1    │
│   DOCUPIPE_API_KEY      │ OCR service key                  │ ****                       │
│   DYNAMODB_TABLE        │ Templates table                  │ docgen-templates           │
│   LOG_LEVEL             │ Logging level                    │ INFO                       │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Initial Input

```json
{
  "doc_type": "summary_report",
  "reference_id": "case-123"
}
```

**Note:** `job_id` is NOT in input - it's generated from `$$.Execution.Name`

---

## State Flow (Detailed)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              STATE FLOW - DETAILED I/O                                   │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Initial Input:                                                                        │
│   { doc_type, reference_id }                                                            │
│                                                                                         │
│   ═══════════════════════════════════════════════════════════════════════════════════   │
│                                                                                         │
│   STEP 1: GetBuildInstructions                                                          │
│   ─────────────────────────────────────────────────────────────────────────────────     │
│   Lambda: docgen-doc-builder                                                            │
│   Action: get_template                                                                  │
│   ResultPath: $.step1                                                                   │
│                                                                                         │
│   Input:  { doc_type, reference_id }                                                    │
│   Adds:   job_id (from $$.Execution.Name)                                               │
│                                                                                         │
│   Output: {                                                                             │
│     doc_type: "summary_report",                                                         │
│     reference_id: "case-123",                                                           │
│     step1: {                                                                            │
│       job_id: "docgen-abc123-def456",                                                   │
│       template: {                                                                       │
│         sections: ["intro", "background", "findings", "recommendations"],               │
│         instructions: { intro: "...", background: "...", ... },                         │
│         api_endpoints: {                                                                │
│           metadata_url: "/api/cases/{reference_id}/metadata",                           │
│           files_list_url: "/api/cases/{reference_id}/files"                             │
│         }                                                                               │
│       }                                                                                 │
│     }                                                                                   │
│   }                                                                                     │
│                                                                                         │
│   ═══════════════════════════════════════════════════════════════════════════════════   │
│                                                                                         │
│   STEP 2: FetchMetadata                                                                 │
│   ─────────────────────────────────────────────────────────────────────────────────     │
│   Lambda: docgen-data-fetcher                                                           │
│   Action: fetch_metadata                                                                │
│   ResultPath: $.step2                                                                   │
│                                                                                         │
│   Input:  { ..., step1.template.api_endpoints }                                         │
│                                                                                         │
│   Output: {                                                                             │
│     ...,                                                                                │
│     step2: {                                                                            │
│       metadata: {                                                                       │
│         data: {                                                                         │
│           case_number: "12345",                                                         │
│           case_name: "תיק לדוגמה",                                                       │
│           created_date: "2024-01-15"                                                    │
│         },                                                                              │
│         files: [                                                                        │
│           "https://onprem.local/api/files/f-001/content",                               │
│           "https://onprem.local/api/files/f-002/content",                               │
│           "https://onprem.local/api/files/f-003/content"                                │
│         ]                                                                               │
│       }                                                                                 │
│     }                                                                                   │
│   }                                                                                     │
│                                                                                         │
│   ═══════════════════════════════════════════════════════════════════════════════════   │
│                                                                                         │
│   STEP 3: FetchAllFiles (MAP)                                                           │
│   ─────────────────────────────────────────────────────────────────────────────────     │
│   Lambda: docgen-data-fetcher                                                           │
│   Action: fetch_file                                                                    │
│   MaxConcurrency: 5                                                                     │
│   ItemsPath: $.step2.metadata.files                                                     │
│   ResultPath: $.step3                                                                   │
│                                                                                         │
│   Each iteration receives: "https://onprem.local/api/files/f-001/content"               │
│                                                                                         │
│   Each iteration returns: {                                                             │
│     url: "https://onprem.local/api/files/f-001/content",                                │
│     s3_key: "jobs/docgen-abc123/files/f-001.pdf"                                        │
│   }                                                                                     │
│                                                                                         │
│   Output: {                                                                             │
│     ...,                                                                                │
│     step3: [                                                                            │
│       { url: "...", s3_key: "jobs/{job_id}/files/f-001.pdf" },                          │
│       { url: "...", s3_key: "jobs/{job_id}/files/f-002.pdf" },                          │
│       { url: "...", s3_key: "jobs/{job_id}/files/f-003.pdf" }                           │
│     ]                                                                                   │
│   }                                                                                     │
│                                                                                         │
│   ═══════════════════════════════════════════════════════════════════════════════════   │
│                                                                                         │
│   STEP 4: ExtractAllOCR (MAP)                                                           │
│   ─────────────────────────────────────────────────────────────────────────────────     │
│   Lambda: docgen-ocr-processor                                                          │
│   Action: extract_ocr                                                                   │
│   MaxConcurrency: 5 (DocuPipe rate limit)                                               │
│   ItemsPath: $.step3                                                                    │
│   ResultPath: $.step4                                                                   │
│                                                                                         │
│   Each iteration returns: {                                                             │
│     source_file: "f-001.pdf",                                                           │
│     s3_key: "jobs/{job_id}/extracted/f-001.json",                                       │
│     pages_count: 5                                                                      │
│   }                                                                                     │
│                                                                                         │
│   S3 JSON (jobs/{job_id}/extracted/f-001.json):                                         │
│   {                                                                                     │
│     "source_file": "f-001.pdf",                                                         │
│     "pages_count": 5,                                                                   │
│     "extracted_at": "2025-01-18T10:30:00Z",                                             │
│     "text": "כל הטקסט שחולץ מהמסמך...",                                                  │
│     "pages": [                                                                          │
│       { "page": 1, "text": "טקסט עמוד 1..." },                                          │
│       { "page": 2, "text": "טקסט עמוד 2..." }                                           │
│     ]                                                                                   │
│   }                                                                                     │
│                                                                                         │
│   ═══════════════════════════════════════════════════════════════════════════════════   │
│                                                                                         │
│   STEP 5: SummarizeContent (MAP)                                                        │
│   ─────────────────────────────────────────────────────────────────────────────────     │
│   Lambda: docgen-ai-processor                                                           │
│   Action: summarize                                                                     │
│   MaxConcurrency: 10 (Claude 3 Sonnet: 500 RPM)                                         │
│   ItemsPath: $.step4                                                                    │
│   ResultPath: $.step5                                                                   │
│                                                                                         │
│   Each iteration returns: {                                                             │
│     source_file: "f-001.pdf",                                                           │
│     s3_key: "jobs/{job_id}/summaries/f-001.json",                                       │
│     token_count: 1250                                                                   │
│   }                                                                                     │
│                                                                                         │
│   S3 JSON (jobs/{job_id}/summaries/f-001.json):                                         │
│   {                                                                                     │
│     "source_file": "f-001.pdf",                                                         │
│     "source_s3_key": "jobs/{job_id}/extracted/f-001.json",                              │
│     "summarized_at": "2025-01-18T10:35:00Z",                                            │
│     "token_count": 1250,                                                                │
│     "summary": "תקציר המסמך: מדובר בהסכם..."                                             │
│   }                                                                                     │
│                                                                                         │
│   ═══════════════════════════════════════════════════════════════════════════════════   │
│                                                                                         │
│   STEP 6: RedactPII (MAP)                                                               │
│   ─────────────────────────────────────────────────────────────────────────────────     │
│   Lambda: docgen-ai-processor                                                           │
│   Action: redact_pii                                                                    │
│   MaxConcurrency: 15                                                                    │
│   ItemsPath: $.step5                                                                    │
│   ResultPath: $.step6                                                                   │
│                                                                                         │
│   Each iteration returns: {                                                             │
│     source_file: "f-001.pdf",                                                           │
│     s3_key: "jobs/{job_id}/redacted/f-001.json",                                        │
│     redaction_count: 7                                                                  │
│   }                                                                                     │
│                                                                                         │
│   S3 JSON (jobs/{job_id}/redacted/f-001.json):                                          │
│   {                                                                                     │
│     "source_file": "f-001.pdf",                                                         │
│     "source_s3_key": "jobs/{job_id}/summaries/f-001.json",                              │
│     "redacted_at": "2025-01-18T10:40:00Z",                                              │
│     "redaction_count": 7,                                                               │
│     "redactions": [                                                                     │
│       { "type": "name", "original": "יוסי כהן", "replacement": "[שם]" },                 │
│       { "type": "id", "original": "123456789", "replacement": "[ת.ז.]" },               │
│       { "type": "phone", "original": "050-1234567", "replacement": "[טלפון]" }          │
│     ],                                                                                  │
│     "redacted_text": "תקציר המסמך: מדובר בהסכם בין [שם] ת.ז. [ת.ז.]..."                 │
│   }                                                                                     │
│                                                                                         │
│   ═══════════════════════════════════════════════════════════════════════════════════   │
│                                                                                         │
│   STEP 7: GenerateParagraphs                                                            │
│   ─────────────────────────────────────────────────────────────────────────────────     │
│   Lambda: docgen-ai-processor                                                           │
│   Action: generate_paragraphs                                                           │
│   ResultPath: $.step7                                                                   │
│                                                                                         │
│   Input: step6 (redacted[]) + step1.template.instructions                               │
│                                                                                         │
│   Output: {                                                                             │
│     ...,                                                                                │
│     step7: {                                                                            │
│       paragraphs_s3_key: "jobs/{job_id}/output/paragraphs.json"                         │
│     }                                                                                   │
│   }                                                                                     │
│                                                                                         │
│   S3 JSON (jobs/{job_id}/output/paragraphs.json):                                       │
│   {                                                                                     │
│     "generated_at": "2025-01-18T10:43:00Z",                                             │
│     "paragraphs": [                                                                     │
│       { "section_id": "intro", "title": "מבוא", "order": 1, "text": "..." },            │
│       { "section_id": "background", "title": "רקע", "order": 2, "text": "..." },        │
│       { "section_id": "findings", "title": "ממצאים", "order": 3, "text": "..." },       │
│       { "section_id": "recommendations", "title": "המלצות", "order": 4, "text": "..." } │
│     ]                                                                                   │
│   }                                                                                     │
│                                                                                         │
│   ═══════════════════════════════════════════════════════════════════════════════════   │
│                                                                                         │
│   STEP 8: AssembleAndCallback                                                           │
│   ─────────────────────────────────────────────────────────────────────────────────     │
│   Lambda: docgen-doc-builder                                                            │
│   Action: assemble_and_callback                                                         │
│   ResultPath: $.step8                                                                   │
│                                                                                         │
│   Input: step7.paragraphs_s3_key + step2.metadata.data + job_id                         │
│                                                                                         │
│   Output: {                                                                             │
│     ...,                                                                                │
│     step8: {                                                                            │
│       document_s3_key: "jobs/{job_id}/output/document.json",                            │
│       callback_status: "sent"                                                           │
│     }                                                                                   │
│   }                                                                                     │
│                                                                                         │
│   S3 JSON (jobs/{job_id}/output/document.json) - Full Document:                         │
│   {                                                                                     │
│     "job_id": "docgen-abc123-def456",                                                   │
│     "doc_type": "summary_report",                                                       │
│     "reference_id": "case-123",                                                         │
│     "status": "COMPLETED",                                                              │
│     "created_at": "2025-01-18T10:30:00Z",                                               │
│     "completed_at": "2025-01-18T10:45:00Z",                                             │
│                                                                                         │
│     "document": "מבוא\n\nזהו דוח סיכום המתאר את...\n\nרקע\n\n                            │
│                  התיק נפתח בתאריך...\n\nממצאים\n\n1. נמצא כי...\n\n                      │
│                  המלצות\n\nלאור הממצאים מומלץ...",                                       │
│                                                                                         │
│     "metadata": {                                                                       │
│       "word_count": 2500,                                                               │
│       "source_files_count": 3,                                                          │
│       "total_redactions": 15,                                                           │
│       "processing_time_seconds": 900                                                    │
│     }                                                                                   │
│   }                                                                                     │
│                                                                                         │
│   Callback: POST to CALLBACK_URL (env var) with same JSON                               │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## S3 Bucket Structure

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                  S3 BUCKET STRUCTURE                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   s3://docgen-{env}-bucket/                                                             │
│   │                                                                                     │
│   └── jobs/                                                                             │
│       └── {job_id}/                                                                     │
│           │                                                                             │
│           ├── files/                    ← Step 3: Original PDFs                         │
│           │   ├── f-001.pdf                                                             │
│           │   ├── f-002.pdf                                                             │
│           │   └── f-003.pdf                                                             │
│           │                                                                             │
│           ├── extracted/                ← Step 4: OCR JSON                              │
│           │   ├── f-001.json               { text, pages[], pages_count }               │
│           │   ├── f-002.json                                                            │
│           │   └── f-003.json                                                            │
│           │                                                                             │
│           ├── summaries/                ← Step 5: Summary JSON                          │
│           │   ├── f-001.json               { summary, token_count }                     │
│           │   ├── f-002.json                                                            │
│           │   └── f-003.json                                                            │
│           │                                                                             │
│           ├── redacted/                 ← Step 6: Redacted JSON                         │
│           │   ├── f-001.json               { redacted_text, redactions[] }              │
│           │   ├── f-002.json                                                            │
│           │   └── f-003.json                                                            │
│           │                                                                             │
│           └── output/                   ← Step 7-8: Final output                        │
│               ├── paragraphs.json          { paragraphs[] }                             │
│               └── document.json            { document (full text), metadata }           │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## ASL Definition (Step Functions)

```json
{
  "Comment": "DocGen MVP - Document Generation Pipeline",
  "StartAt": "GetBuildInstructions",
  "States": {
    
    "GetBuildInstructions": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:docgen-doc-builder",
      "Parameters": {
        "action": "get_template",
        "doc_type.$": "$.doc_type",
        "job_id.$": "$$.Execution.Name"
      },
      "ResultPath": "$.step1",
      "Next": "FetchMetadata",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "HandleError"
      }]
    },

    "FetchMetadata": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:docgen-data-fetcher",
      "Parameters": {
        "action": "fetch_metadata",
        "job_id.$": "$.step1.job_id",
        "reference_id.$": "$.reference_id",
        "api_endpoints.$": "$.step1.template.api_endpoints"
      },
      "ResultPath": "$.step2",
      "Next": "FetchAllFiles",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "HandleError"
      }]
    },

    "FetchAllFiles": {
      "Type": "Map",
      "ItemsPath": "$.step2.metadata.files",
      "MaxConcurrency": 5,
      "Parameters": {
        "action": "fetch_file",
        "job_id.$": "$.step1.job_id",
        "file_url.$": "$$.Map.Item.Value"
      },
      "Iterator": {
        "StartAt": "FetchSingleFile",
        "States": {
          "FetchSingleFile": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:docgen-data-fetcher",
            "End": true
          }
        }
      },
      "ResultPath": "$.step3",
      "Next": "ExtractAllOCR",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "HandleError"
      }]
    },

    "ExtractAllOCR": {
      "Type": "Map",
      "ItemsPath": "$.step3",
      "MaxConcurrency": 5,
      "Parameters": {
        "action": "extract_ocr",
        "job_id.$": "$.step1.job_id",
        "file.$": "$$.Map.Item.Value"
      },
      "Iterator": {
        "StartAt": "ExtractSingleFile",
        "States": {
          "ExtractSingleFile": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:docgen-ocr-processor",
            "End": true
          }
        }
      },
      "ResultPath": "$.step4",
      "Next": "SummarizeContent",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "HandleError"
      }]
    },

    "SummarizeContent": {
      "Type": "Map",
      "ItemsPath": "$.step4",
      "MaxConcurrency": 10,
      "Parameters": {
        "action": "summarize",
        "job_id.$": "$.step1.job_id",
        "extracted.$": "$$.Map.Item.Value"
      },
      "Iterator": {
        "StartAt": "SummarizeSingleFile",
        "States": {
          "SummarizeSingleFile": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:docgen-ai-processor",
            "End": true
          }
        }
      },
      "ResultPath": "$.step5",
      "Next": "RedactPII",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "HandleError"
      }]
    },

    "RedactPII": {
      "Type": "Map",
      "ItemsPath": "$.step5",
      "MaxConcurrency": 15,
      "Parameters": {
        "action": "redact_pii",
        "job_id.$": "$.step1.job_id",
        "summary.$": "$$.Map.Item.Value"
      },
      "Iterator": {
        "StartAt": "RedactSingleFile",
        "States": {
          "RedactSingleFile": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:docgen-ai-processor",
            "End": true
          }
        }
      },
      "ResultPath": "$.step6",
      "Next": "GenerateParagraphs",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "HandleError"
      }]
    },

    "GenerateParagraphs": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:docgen-ai-processor",
      "Parameters": {
        "action": "generate_paragraphs",
        "job_id.$": "$.step1.job_id",
        "redacted.$": "$.step6",
        "template.$": "$.step1.template"
      },
      "ResultPath": "$.step7",
      "Next": "AssembleAndCallback",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "HandleError"
      }]
    },

    "AssembleAndCallback": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:docgen-doc-builder",
      "Parameters": {
        "action": "assemble_and_callback",
        "job_id.$": "$.step1.job_id",
        "doc_type.$": "$.doc_type",
        "reference_id.$": "$.reference_id",
        "paragraphs_s3_key.$": "$.step7.paragraphs_s3_key",
        "metadata.$": "$.step2.metadata.data"
      },
      "ResultPath": "$.step8",
      "Next": "Success",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "HandleError"
      }]
    },

    "HandleError": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:docgen-doc-builder",
      "Parameters": {
        "action": "send_error_callback",
        "job_id.$": "$.step1.job_id",
        "doc_type.$": "$.doc_type",
        "reference_id.$": "$.reference_id",
        "error.$": "$.error"
      },
      "Next": "Failed"
    },

    "Success": {
      "Type": "Succeed"
    },

    "Failed": {
      "Type": "Fail",
      "Error": "DocumentGenerationFailed",
      "Cause": "Pipeline failed - error callback sent to On-Prem"
    }
  }
}
```

---

## Lambda Functions Summary

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              LAMBDA FUNCTIONS SUMMARY                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Function              │ Actions                    │ Steps    │ VPC Required          │
│   ──────────────────────┼────────────────────────────┼──────────┼───────────────────────│
│   docgen-doc-builder    │ get_template               │ 1        │ No                    │
│                         │ assemble_and_callback      │ 8        │ Yes (callback)        │
│                         │ send_error_callback        │ Error    │ Yes (callback)        │
│   ──────────────────────┼────────────────────────────┼──────────┼───────────────────────│
│   docgen-data-fetcher   │ fetch_metadata             │ 2        │ Yes (On-Prem API)     │
│                         │ fetch_file                 │ 3        │ Yes (On-Prem API)     │
│   ──────────────────────┼────────────────────────────┼──────────┼───────────────────────│
│   docgen-ocr-processor  │ extract_ocr                │ 4        │ No (DocuPipe API)     │
│   ──────────────────────┼────────────────────────────┼──────────┼───────────────────────│
│   docgen-ai-processor   │ summarize                  │ 5        │ No                    │
│                         │ redact_pii                 │ 6        │ No                    │
│                         │ generate_paragraphs        │ 7        │ No                    │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Lambda Layer (Shared)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                  LAMBDA LAYER                                            │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   layers/shared/python/docgen_common/                                                   │
│   │                                                                                     │
│   ├── __init__.py                                                                       │
│   ├── config.py           # Environment variables (S3_BUCKET, CALLBACK_URL, etc.)       │
│   ├── s3_utils.py         # read_json(), write_json(), upload_file()                    │
│   ├── models.py           # Pydantic models                                             │
│   ├── errors.py           # Custom exceptions                                           │
│   ├── callback.py         # send_callback(), retry logic                                │
│   └── bedrock.py          # Bedrock client wrapper                                      │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Callback Payload (On-Prem)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              CALLBACK PAYLOAD                                            │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Endpoint: POST {CALLBACK_URL} (environment variable)                                  │
│                                                                                         │
│   Headers:                                                                              │
│   ─────────────────────────────────────────────────────────────────────────────────     │
│   Content-Type: application/json                                                        │
│   X-DocGen-Job-Id: docgen-abc123-def456                                                 │
│   X-DocGen-Timestamp: 2025-01-18T10:45:00Z                                              │
│   X-DocGen-Signature: sha256=a1b2c3d4e5f6...                                            │
│                                                                                         │
│   SUCCESS Payload:                                                                      │
│   ─────────────────────────────────────────────────────────────────────────────────     │
│   {                                                                                     │
│     "job_id": "docgen-abc123-def456",                                                   │
│     "doc_type": "summary_report",                                                       │
│     "reference_id": "case-123",                                                         │
│     "status": "COMPLETED",                                                              │
│     "created_at": "2025-01-18T10:30:00Z",                                               │
│     "completed_at": "2025-01-18T10:45:00Z",                                             │
│                                                                                         │
│     "document": "מבוא\n\nזהו דוח סיכום...\n\nרקע\n\n...\n\nממצאים\n\n...",              │
│                                                                                         │
│     "metadata": {                                                                       │
│       "word_count": 2500,                                                               │
│       "source_files_count": 3,                                                          │
│       "total_redactions": 15,                                                           │
│       "processing_time_seconds": 900                                                    │
│     }                                                                                   │
│   }                                                                                     │
│                                                                                         │
│   FAILED Payload:                                                                       │
│   ─────────────────────────────────────────────────────────────────────────────────     │
│   {                                                                                     │
│     "job_id": "docgen-abc123-def456",                                                   │
│     "doc_type": "summary_report",                                                       │
│     "reference_id": "case-123",                                                         │
│     "status": "FAILED",                                                                 │
│     "created_at": "2025-01-18T10:30:00Z",                                               │
│     "failed_at": "2025-01-18T10:35:00Z",                                                │
│                                                                                         │
│     "error": {                                                                          │
│       "code": "OCR_EXTRACTION_FAILED",                                                  │
│       "message": "DocuPipe timeout after 5 minutes",                                    │
│       "step": "ExtractAllOCR",                                                          │
│       "details": { ... }                                                                │
│     }                                                                                   │
│   }                                                                                     │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Error Codes

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                  ERROR CODES                                             │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Code                       │ Step    │ Description                                    │
│   ───────────────────────────┼─────────┼────────────────────────────────────────────────│
│   TEMPLATE_NOT_FOUND         │ 1       │ doc_type not found in DynamoDB                 │
│   METADATA_FETCH_FAILED      │ 2       │ Failed to fetch from On-Prem API               │
│   FILE_FETCH_FAILED          │ 3       │ Failed to download file                        │
│   OCR_EXTRACTION_FAILED      │ 4       │ DocuPipe processing failed                     │
│   SUMMARIZATION_FAILED       │ 5       │ Bedrock summarization failed                   │
│   PII_REDACTION_FAILED       │ 6       │ PII detection/redaction failed                 │
│   GENERATION_FAILED          │ 7       │ Paragraph generation failed                    │
│   ASSEMBLY_FAILED            │ 8       │ Document assembly failed                       │
│   CALLBACK_FAILED            │ 8       │ Failed to send callback (after retries)        │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Timing Estimates

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              TIMING ESTIMATES                                            │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Model: Claude 3 Sonnet (500 RPM)                                                      │
│   Files: 50 PDFs                                                                        │
│                                                                                         │
│   Step                        │ Time       │ Notes                                      │
│   ────────────────────────────┼────────────┼────────────────────────────────────────────│
│   1. GetBuildInstructions     │ ~1 sec     │ DynamoDB read                              │
│   2. FetchMetadata            │ ~2 sec     │ Single API call                            │
│   3. FetchAllFiles (MAP)      │ ~30 sec    │ maxConcurrency: 5                          │
│   4. ExtractAllOCR (MAP)      │ ~2-3 min   │ maxConcurrency: 5, DocuPipe async          │
│   5. SummarizeContent (MAP)   │ ~1 min     │ maxConcurrency: 10                         │
│   6. RedactPII (MAP)          │ ~30 sec    │ maxConcurrency: 15                         │
│   7. GenerateParagraphs       │ ~30 sec    │ Single call, 4 sections                    │
│   8. AssembleAndCallback      │ ~5 sec     │ S3 + HTTP POST                             │
│   ────────────────────────────┼────────────┼────────────────────────────────────────────│
│   TOTAL                       │ ~5-6 min   │ For 50 files                               │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Cost Estimates

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              COST ESTIMATES                                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Claude 3 Sonnet Pricing:                                                              │
│   • Input:  $3.00 / 1M tokens                                                           │
│   • Output: $15.00 / 1M tokens                                                          │
│                                                                                         │
│   Per Document (50 files, ~500 pages):                                                  │
│   ───────────────────────────────────────────────────────────────────────────────────   │
│   Step 5 (Summarize):  ~100K input + ~20K output = $0.30 + $0.30 = $0.60               │
│   Step 6 (Redact):     ~20K input + ~20K output  = $0.06 + $0.30 = $0.36               │
│   Step 7 (Generate):   ~30K input + ~5K output   = $0.09 + $0.08 = $0.17               │
│   ───────────────────────────────────────────────────────────────────────────────────   │
│   TOTAL BEDROCK COST: ~$1.00-1.50 per document                                          │
│                                                                                         │
│   Other Costs:                                                                          │
│   • Lambda: ~$0.05                                                                      │
│   • S3: ~$0.01                                                                          │
│   • Step Functions: ~$0.05                                                              │
│   • DocuPipe (OCR): varies                                                              │
│   ───────────────────────────────────────────────────────────────────────────────────   │
│   TOTAL AWS COST: ~$1.10-1.60 per document (excluding OCR)                              │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Development Checklist

### AWS Side
- [ ] Lambda Layer (shared code)
- [ ] Lambda: docgen-doc-builder (get_template, assemble_and_callback)
- [ ] Lambda: docgen-data-fetcher (fetch_metadata, fetch_file)
- [ ] Lambda: docgen-ocr-processor (extract_ocr)
- [ ] Lambda: docgen-ai-processor (summarize, redact_pii, generate_paragraphs)
- [ ] Step Functions state machine
- [ ] API Gateway endpoints
- [ ] DynamoDB table (templates)
- [ ] S3 bucket
- [ ] VPC + VPN configuration
- [ ] IAM roles and policies
- [ ] CloudWatch alarms
- [ ] **REQUEST QUOTA INCREASE FOR CLAUDE 3.5/4** (start early!)

### On-Prem Side
- [ ] POST {CALLBACK_URL} endpoint
- [ ] Signature validation (HMAC-SHA256)
- [ ] Idempotency check (duplicate job_id)
- [ ] Handle COMPLETED status
- [ ] Handle FAILED status
- [ ] Return proper HTTP codes (200/400/500)
- [ ] Logging
