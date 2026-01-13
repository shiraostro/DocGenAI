# DocGen MVP - Step Functions Architecture (Final v4)

## Changes from v3
- **STEP 6 → STEP 1**: GetBuildInstructions moved to first step (contains API endpoints per doc_type)
- **STEP 5 (SummarizeContent)**: Now **Map State** for parallel processing
- **STEP 6 (RedactPII)**: Now **Map State** for parallel processing
- **Sequential Flow**: Steps 5→6 run sequentially (Option A) for simpler implementation
- **Model**: Claude 3 Sonnet for MVP (better rate limits)

---

## Bedrock Model Selection

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              BEDROCK MODEL SELECTION                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ★ MVP MODEL: Claude 3 Sonnet (anthropic.claude-3-sonnet-20240229-v1:0)                │
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │ WHY CLAUDE 3 SONNET FOR MVP?                                                    │   │
│  │                                                                                 │   │
│  │  ✅ Rate Limits: 500 RPM (vs 1-2 RPM for Claude 3.5/4)                         │   │
│  │  ✅ TPM: ~100,000 tokens/min (vs ~2,000-4,000 for newer models)                │   │
│  │  ✅ Quality: 85-90% of Claude 3.5/4 quality - sufficient for MVP              │   │
│  │  ✅ Hebrew: Good Hebrew support                                                │   │
│  │  ✅ Cost: Similar pricing, no premium                                          │   │
│  │  ✅ Context: 200K tokens (same as newer models)                                │   │
│  │                                                                                 │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │ MODEL COMPARISON                                                                │   │
│  │                                                                                 │   │
│  │  Model              │ RPM Default │ Hebrew   │ Quality │ Recommendation        │   │
│  │  ───────────────────┼─────────────┼──────────┼─────────┼─────────────────────  │   │
│  │  Claude 3 Sonnet    │ 500 ✅      │ Good     │ 85%     │ ★ MVP                 │   │
│  │  Claude 3.5 Sonnet  │ 1-2 ❌      │ Better   │ 95%     │ After quota increase  │   │
│  │  Claude Sonnet 4    │ 1-2 ❌      │ Best     │ 100%    │ Production (future)   │   │
│  │                                                                                 │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │ ⚠️  IMPORTANT: REQUEST QUOTA INCREASE NOW!                                      │   │
│  │                                                                                 │   │
│  │  For future upgrade to Claude 3.5/4 Sonnet, request quota increase:            │   │
│  │                                                                                 │   │
│  │  1. AWS Console → Service Quotas → Amazon Bedrock                              │   │
│  │  2. Request increase for:                                                       │   │
│  │     • InvokeModel requests per minute: 50-100 RPM                              │   │
│  │     • Tokens per minute: 100,000-300,000 TPM                                   │   │
│  │  3. Alternative: Contact TAM/SA/AM for internal quota request                  │   │
│  │                                                                                 │   │
│  │  ⏰ Process takes days/weeks - START EARLY!                                     │   │
│  │                                                                                 │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │ 🇮🇱 BEST MODEL FOR HEBREW (After Quota Increase)                                │   │
│  │                                                                                 │   │
│  │  Recommendation: Claude Sonnet 4 (anthropic.claude-sonnet-4-20250514-v1:0)     │   │
│  │                                                                                 │   │
│  │  • Best Hebrew language understanding and generation                           │   │
│  │  • Excellent RTL text handling                                                 │   │
│  │  • Superior context retention for Hebrew documents                             │   │
│  │  • Better understanding of Hebrew legal/business terminology                   │   │
│  │                                                                                 │   │
│  │  Alternative: Claude 3.5 Sonnet v2 - also excellent Hebrew support            │   │
│  │                                                                                 │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│  CONFIGURATION (Environment Variable):                                                  │
│  ─────────────────────────────────────────────────────────────────────────────────     │
│  # MVP (default)                                                                        │
│  BEDROCK_MODEL_ID=anthropic.claude-3-sonnet-20240229-v1:0                              │
│                                                                                         │
│  # After quota increase (Phase 2)                                                       │
│  BEDROCK_MODEL_ID=anthropic.claude-sonnet-4-20250514-v1:0                              │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    SYSTEM OVERVIEW                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   ┌──────────────────────────────────────────────────────────────────────────────────┐ │
│   │                                                                                  │ │
│   │   On-Prem System                              AWS Cloud                          │ │
│   │   ┌─────────────┐                            ┌─────────────────────────────────┐ │ │
│   │   │             │ ─── POST /documents ──────▶│         API Gateway             │ │ │
│   │   │   Client    │ ◀─── { job_id, status } ────│                                 │ │ │
│   │   │   System    │                            └────────────┬────────────────────┘ │ │
│   │   │             │                                         │                      │ │
│   │   │             │                                         ▼                      │ │
│   │   │             │                            ┌─────────────────────────────────┐ │ │
│   │   │             │                            │       Step Functions            │ │ │
│   │   │             │                            │       (8 Steps)                 │ │ │
│   │   │             │                            └────────────┬────────────────────┘ │ │
│   │   │             │                                         │                      │ │
│   │   │             │                                         ▼                      │ │
│   │   │  ┌───────┐  │                            ┌─────────────────────────────────┐ │ │
│   │   │  │Callback│ │ ◀─── POST (JSON result) ────│      Lambda (doc-builder)      │ │ │
│   │   │  │Endpoint│ │ ─── 200 OK ───────────────▶│                                 │ │ │
│   │   │  └───────┘  │                            └─────────────────────────────────┘ │ │
│   │   └─────────────┘                                                                │ │
│   │                                                                                  │ │
│   └──────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## Flow Summary

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    REQUEST FLOW                                         │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   1. On-Prem sends POST /documents                                                      │
│      {                                                                                  │
│        "doc_type": "summary_report",                                                    │
│        "reference_id": "case-12345",                                                    │
│        "callback_url": "https://onprem.company.local/api/docgen/callback"               │
│      }                                                                                  │
│                                                                                         │
│   2. AWS returns immediately:                                                           │
│      {                                                                                  │
│        "job_id": "job-abc-123",                                                         │
│        "status": "PROCESSING"                                                           │
│      }                                                                                  │
│                                                                                         │
│   3. Step Functions processes (Steps 1-8) - SEQUENTIAL FLOW                            │
│                                                                                         │
│   4. AWS sends callback to On-Prem with full JSON result                                │
│                                                                                         │
│   5. On-Prem saves document and updates UI                                              │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## Lambda Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                  LAMBDA ARCHITECTURE                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   ┌───────────────────────────────────────────────────────────────────────────────────┐  │
│   │                        docgen-shared (Lambda Layer)                              │  │
│   │                                                                                  │  │
│   │   • S3 utilities    • Error handling    • Logging    • Common models            │  │
│   └───────────────────────────────────────────────────────────────────────────────────┘  │
│                                          │                                              │
│            ┌─────────────────────────────┼─────────────────────────────┐               │
│            │                             │                             │               │
│            ▼                             ▼                             ▼               │
│   ┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐         │
│   │ docgen-data-fetcher │   │ docgen-ocr-processor│   │ docgen-ai-processor │         │
│   │                     │   │                     │   │                     │         │
│   │ Steps: 2, 3         │   │ Step: 4             │   │ Steps: 5, 6, 7      │         │
│   │ Memory: 512MB       │   │ Memory: 256MB       │   │ Memory: 1024MB      │         │
│   │ Timeout: 2min       │   │ Timeout: 10min      │   │ Timeout: 10min      │         │
│   │                     │   │                     │   │                     │         │
│   │ • fetch_metadata()  │   │ • extract_ocr()     │   │ • summarize()       │         │
│   │ • fetch_file()      │   │ • poll_docupipe()   │   │ • redact_pii()      │         │
│   │                     │   │                     │   │ • gen_paragraph()   │         │
│   └─────────────────────┘   └─────────────────────┘   └─────────────────────┘         │
│            │                             │                             │               │
│            │                             │                             │               │
│            │                             ▼                             │               │
│            │                    ┌─────────────────────┐                │               │
│            │                    │docgen-doc-builder   │                │               │
│            │                    │                     │                │               │
│            │                    │ Steps: 1, 8         │                │               │
│            │                    │ Memory: 256MB       │                │               │
│            │                    │ Timeout: 1min       │                │               │
│            │                    │                     │                │               │
│            │                    │ • get_template()  ★ │                │               │
│            │                    │ • assemble_doc()    │                │               │
│            │                    │ • send_callback() ★ │                │               │
│            │                    └─────────────────────┘                │               │
│            │                             │                             │               │
│            └─────────────────────────────┼─────────────────────────────┘               │
│                                          │                                              │
│                                          ▼                                              │
│                              External Services:                                         │
│                     On-Prem API  │  DocuPipe  │  Bedrock  │  S3  │  DynamoDB           │
│                                       (Claude 3 Sonnet)                                 │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## Step Functions State Machine (Sequential Flow)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              STEP FUNCTIONS STATE MACHINE                               │
│                                  (SEQUENTIAL FLOW)                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│  ║ STEP 1: GetBuildInstructions ★ Contains API Endpoints per doc_type               ║   │
│  ║                                                                                  ║   │
│  ║ ┌────────────────────┐       ┌────────────┐                                     ║   │
│  ║ │ docgen-doc-builder │──────▶│  DynamoDB  │                                     ║   │
│  ║ │ .get_template()    │       │ Templates  │                                     ║   │
│  ║ └────────────────────┘       └────────────┘                                     ║   │
│  ║                                                                                  ║   │
│  ║   Input:  doc_type = "summary_report"                                           ║   │
│  ║   Output: {                                                                      ║   │
│  ║     sections: ["מבוא", "רקע", "ממצאים", "המלצות"],                              ║   │
│  ║     instructions: { "מבוא": "...", "רקע": "...", ... },                          ║   │
│  ║     api_endpoints: {                                                            ║   │
│  ║       "metadata_url": "https://onprem.local/api/cases/{id}/metadata",           ║   │
│  ║       "files_url": "https://onprem.local/api/cases/{id}/files",                 ║   │
│  ║       "file_content_url": "https://onprem.local/api/files/{file_id}/content"    ║   │
│  ║     }                                                                            ║   │
│  ║   }                                                                              ║   │
│  ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                          │                                              │
│                                          ▼                                              │
│  ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│  ║ STEP 2: FetchMetadata                                                            ║   │
│  ║                                                                                  ║   │
│  ║ ┌────────────────────┐       ┌────────────┐       ┌────────────┐                ║   │
│  ║ │ docgen-data-fetcher│──────▶│  On-Prem   │──────▶│     S3     │                ║   │
│  ║ │ .fetch_metadata()  │  VPN  │    API     │       │ metadata/  │                ║   │
│  ║ └────────────────────┘       └────────────┘       └────────────┘                ║   │
│  ║                                                                                  ║   │
│  ║ Input:  { doc_type, reference_id, callback_url, api_endpoints }                 ║   │
│  ║ Output: { data: {...}, files: ["doc1.pdf", ...] }                               ║   │
│  ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                          │                                              │
│                                          ▼                                              │
│  ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│  ║ STEP 3: FetchAllFiles (MAP STATE - Parallel)                                     ║   │
│  ║                                                                                  ║   │
│  ║   maxConcurrency: 5                                                              ║   │
│  ║                                                                                  ║   │
│  ║   ┌──────────┐       ┌────────────────────┐       ┌────────────┐                ║   │
│  ║   │ file_1   │──────▶│                    │       │            │                ║   │
│  ║   ├──────────┤       │ docgen-data-fetcher│──────▶│     S3     │                ║   │
│  ║   │ file_2   │──────▶│ .fetch_file()      │  VPN  │   files/   │                ║   │
│  ║   ├──────────┤       │                    │       │            │                ║   │
│  ║   │   ...    │──────▶│ (On-Prem API →     │       │            │                ║   │
│  ║   ├──────────┤       │  BASE64 → S3)      │       │            │                ║   │
│  ║   │ file_50  │       └────────────────────┘       └────────────┘                ║   │
│  ║   └──────────┘                                                                   ║   │
│  ║                                                                                  ║   │
│  ║   Output: [{ file: "doc1.pdf", s3_path: "s3://..." }, ...]                      ║   │
│  ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                          │                                              │
│                                          ▼                                              │
│  ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│  ║ STEP 4: ExtractAllOCR (MAP STATE - Parallel)                                     ║   │
│  ║                                                                                  ║   │
│  ║   maxConcurrency: 5  (DocuPipe rate limit)                                       ║   │
│  ║                                                                                  ║   │
│  ║   ┌──────────┐       ┌────────────────────┐       ┌────────────┐                ║   │
│  ║   │ file_1   │──────▶│                    │       │            │                ║   │
│  ║   ├──────────┤       │docgen-ocr-processor│──────▶│     S3     │                ║   │
│  ║   │ file_2   │──────▶│ .extract_ocr()     │       │ extracted/ │                ║   │
│  ║   ├──────────┤       │                    │       │            │                ║   │
│  ║   │   ...    │──────▶│ (submit + poll     │       │            │                ║   │
│  ║   ├──────────┤       │  DocuPipe API)     │       │            │                ║   │
│  ║   │ file_50  │       └────────────────────┘       └────────────┘                ║   │
│  ║   └──────────┘                                                                   ║   │
│  ║                                                                                  ║   │
│  ║   Output: [{ file: "doc1.pdf", text_s3: "s3://.../extracted/doc1.json" }, ...]  ║   │
│  ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                          │                                              │
│                                          ▼                                              │
│  ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│  ║ STEP 5: SummarizeContent (MAP STATE - Parallel)                                  ║   │
│  ║                                                                                  ║   │
│  ║   maxConcurrency: 10  (Claude 3 Sonnet: 500 RPM allows high concurrency)        ║   │
│  ║   Model: Claude 3 Sonnet                                                         ║   │
│  ║                                                                                  ║   │
│  ║   ┌──────────────┐       ┌────────────────────┐       ┌────────────┐            ║   │
│  ║   │ extracted_1  │──────▶│                    │       │            │            ║   │
│  ║   ├──────────────┤       │ docgen-ai-processor│──────▶│     S3     │            ║   │
│  ║   │ extracted_2  │──────▶│ .summarize()       │       │  summary/  │            ║   │
│  ║   ├──────────────┤       │                    │       │            │            ║   │
│  ║   │ extracted_3  │──────▶│  (Bedrock Claude)  │       │            │            ║   │
│  ║   ├──────────────┤       │                    │       │            │            ║   │
│  ║   │    ...       │       └────────────────────┘       └────────────┘            ║   │
│  ║   └──────────────┘                                                               ║   │
│  ║                                                                                  ║   │
│  ║   • סיכום כל מסמך בנפרד (parallel)                                               ║   │
│  ║   • איחוד סיכומים בסוף ה-Map                                                     ║   │
│  ║   • אם מסמך > 50K tokens → chunking פנימי                                        ║   │
│  ║                                                                                  ║   │
│  ║   Output: [{ file: "doc1.pdf", summary_s3: "s3://.../summary/doc1.json" }, ...] ║   │
│  ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                          │                                              │
│                                          ▼  (SEQUENTIAL - after summarize completes)    │
│  ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│  ║ STEP 6: RedactPII (MAP STATE - Parallel)                                         ║   │
│  ║                                                                                  ║   │
│  ║   maxConcurrency: 15  (Claude 3 Sonnet: 500 RPM allows high concurrency)        ║   │
│  ║   Model: Claude 3 Sonnet                                                         ║   │
│  ║                                                                                  ║   │
│  ║   ┌──────────────┐       ┌────────────────────┐       ┌────────────┐            ║   │
│  ║   │ summary_1    │──────▶│                    │       │            │            ║   │
│  ║   ├──────────────┤       │ docgen-ai-processor│──────▶│     S3     │            ║   │
│  ║   │ summary_2    │──────▶│ .redact_pii()      │       │  redacted/ │            ║   │
│  ║   ├──────────────┤       │                    │       │            │            ║   │
│  ║   │ summary_3    │──────▶│ (Bedrock Guardrails│       │            │            ║   │
│  ║   ├──────────────┤       │  or Claude)        │       │            │            ║   │
│  ║   │    ...       │       └────────────────────┘       └────────────┘            ║   │
│  ║   └──────────────┘                                                               ║   │
│  ║                                                                                  ║   │
│  ║   • Runs AFTER Step 5 completes (Sequential)                                    ║   │
│  ║   • Redacts only summarized content (smaller, more efficient)                   ║   │
│  ║   • זיהוי PII: שמות, ת.ז., טלפונים, כתובות, מיילים                              ║   │
│  ║   • זיהוי מידע מסווג לפי כללי הארגון                                            ║   │
│  ║   • החלפה ב-[REDACTED] או ****                                                  ║   │
│  ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                          │                                              │
│                                          ▼                                              │
│  ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│  ║ STEP 7: GenerateParagraphs (SEQUENTIAL - by design)                              ║   │
│  ║                                                                                  ║   │
│  ║   Model: Claude 3 Sonnet                                                         ║   │
│  ║                                                                                  ║   │
│  ║ ┌────────────────────┐       ┌────────────┐       ┌────────────┐                ║   │
│  ║ │ docgen-ai-processor│──────▶│  Bedrock   │──────▶│     S3     │                ║   │
│  ║ │ .gen_paragraph()   │       │   Claude   │       │paragraphs/ │                ║   │
│  ║ └────────────────────┘       └────────────┘       └────────────┘                ║   │
│  ║                                                                                  ║   │
│  ║   ┌─────────────────────────────────────────────────────────────────────────┐   ║   │
│  ║   │                                                                         │   ║   │
│  ║   │    Section 1      Section 2      Section 3      Section 4               │   ║   │
│  ║   │    ┌──────┐       ┌──────┐       ┌──────┐       ┌──────┐               │   ║   │
│  ║   │    │ מבוא │──────▶│  רקע │──────▶│ממצאים│──────▶│המלצות│               │   ║   │
│  ║   │    └──────┘       └──────┘       └──────┘       └──────┘               │   ║   │
│  ║   │        │              │              │              │                   │   ║   │
│  ║   │        ▼              ▼              ▼              ▼                   │   ║   │
│  ║   │    Bedrock        Bedrock        Bedrock        Bedrock                 │   ║   │
│  ║   │    (with          (with          (with          (with                   │   ║   │
│  ║   │    context)       prev para)     prev para)     prev para)              │   ║   │
│  ║   │                                                                         │   ║   │
│  ║   └─────────────────────────────────────────────────────────────────────────┘   ║   │
│  ║                                                                                  ║   │
│  ║   • סדרתי - כל פיסקה מקבלת את הקודמת כהקשר                                     ║   │
│  ║   • שומר על קוהרנטיות ורצף לוגי                                                ║   │
│  ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                          │                                              │
│                                          ▼                                              │
│  ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│  ║ STEP 8: AssembleAndCallback                                                      ║   │
│  ║                                                                                  ║   │
│  ║ ┌────────────────────┐       ┌────────────┐       ┌────────────┐                ║   │
│  ║ │ docgen-doc-builder │──────▶│  Assemble  │──────▶│  Callback  │                ║   │
│  ║ │ .assemble_doc()    │       │    JSON    │       │  to OnPrem │                ║   │
│  ║ └────────────────────┘       └────────────┘       └─────┬──────┘                ║   │
│  ║                                                         │                        ║   │
│  ║   ┌─────────────────────────────────────────────────────┼────────────────────┐  ║   │
│  ║   │                                                     │                    │  ║   │
│  ║   │  1. Read all paragraphs from S3                     │                    │  ║   │
│  ║   │  2. Assemble JSON structure                         │                    │  ║   │
│  ║   │  3. Save to S3 (backup)                             │                    │  ║   │
│  ║   │  4. Update DynamoDB status                          ▼                    │  ║   │
│  ║   │  5. Send callback to On-Prem  ──────────▶  ┌──────────────┐              │  ║   │
│  ║   │                                            │   On-Prem    │              │  ║   │
│  ║   │     Retry: 3 attempts                      │   Callback   │              │  ║   │
│  ║   │     Backoff: exponential + jitter          │   Endpoint   │              │  ║   │
│  ║   │     Fallback: DLQ                          └──────────────┘              │  ║   │
│  ║   │                                                                          │  ║   │
│  ║   └──────────────────────────────────────────────────────────────────────────┘  ║   │
│  ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## Timing Estimates (Claude 3 Sonnet)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         TIMING ESTIMATES (CLAUDE 3 SONNET)                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Model: Claude 3 Sonnet (500 RPM, ~100K TPM)                                          │
│                                                                                         │
│   STEP                        │ 10 FILES    │ 50 FILES    │ NOTES                      │
│   ────────────────────────────┼─────────────┼─────────────┼────────────────────────────│
│   1. Get Instructions         │ ~1 sec      │ ~1 sec      │ DynamoDB read + APIs       │
│   2. Fetch Metadata           │ ~2 sec      │ ~2 sec      │ Single API call            │
│   3. Fetch Files (parallel)   │ ~10 sec     │ ~30 sec     │ maxConcurrency: 5          │
│   4. OCR Extract (parallel)   │ ~1 min      │ ~3 min      │ maxConcurrency: 5          │
│   5. Summarize (parallel)     │ ~15 sec     │ ~30 sec     │ maxConcurrency: 10         │
│   6. Redact PII (parallel)    │ ~10 sec     │ ~20 sec     │ maxConcurrency: 15         │
│   7. Generate (sequential)    │ ~30 sec     │ ~30 sec     │ 4 sections × ~8s           │
│   8. Assemble + Callback      │ ~3 sec      │ ~3 sec      │ Assemble + HTTP POST       │
│   ────────────────────────────┼─────────────┼─────────────┼────────────────────────────│
│   TOTAL                       │ ~2.5 min    │ ~5-6 min    │ ✅ Achievable with C3S     │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│                                                                                         │
│   Bedrock Requests per Document:                                                        │
│   • Step 5 (Summarize): 10-50 requests                                                 │
│   • Step 6 (Redact): 10-50 requests                                                    │
│   • Step 7 (Generate): 4 requests                                                      │
│   • TOTAL: ~24-104 requests per document                                               │
│                                                                                         │
│   With 500 RPM: No rate limiting issues! ✅                                             │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## Quick Reference

| Step | Name | Lambda | Method | Type | Model |
|------|------|--------|--------|------|-------|
| 1 | GetBuildInstructions | docgen-doc-builder | get_template() | Task | - |
| 2 | FetchMetadata | docgen-data-fetcher | fetch_metadata() | Task | - |
| 3 | FetchAllFiles | docgen-data-fetcher | fetch_file() | **Map** | - |
| 4 | ExtractAllOCR | docgen-ocr-processor | extract_ocr() | **Map** | DocuPipe |
| 5 | SummarizeContent | docgen-ai-processor | summarize() | **Map** | Claude 3 Sonnet |
| 6 | RedactPII | docgen-ai-processor | redact_pii() | **Map** | Claude 3 Sonnet |
| 7 | GenerateParagraphs | docgen-ai-processor | gen_paragraph() | **Sequential** | Claude 3 Sonnet |
| 8 | AssembleAndCallback | docgen-doc-builder | assemble_doc() | Task | - |


## Callback Mechanism

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                  CALLBACK FLOW                                          │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   AWS (Lambda)                                           On-Prem                        │
│   ┌────────────────────┐                                ┌────────────────────┐         │
│   │                    │                                │                    │         │
│   │  1. Assemble JSON  │                                │                    │         │
│   │         │          │                                │                    │         │
│   │         ▼          │                                │                    │         │
│   │  2. Sign payload   │                                │                    │         │
│   │     (HMAC-SHA256)  │                                │                    │         │
│   │         │          │                                │                    │         │
│   │         ▼          │     POST + Headers + JSON      │                    │         │
│   │  3. Send callback  │ ──────────────────────────────▶│  4. Validate sig   │         │
│   │                    │                                │         │          │         │
│   │                    │                                │         ▼          │         │
│   │                    │                                │  5. Check duplicate│         │
│   │                    │                                │         │          │         │
│   │                    │                                │         ▼          │         │
│   │                    │           200 OK               │  6. Save to DB     │         │
│   │  7. Mark complete  │ ◀───────────────────────────────│         │          │         │
│   │                    │                                │         ▼          │         │
│   │                    │                                │  8. Update UI      │         │
│   │                    │                                │                    │         │
│   └────────────────────┘                                └────────────────────┘         │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Error Handling:                                                                       │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│                                                                                         │
│   On-Prem Returns    │ AWS Action                                                       │
│   ───────────────────┼──────────────────────────────────────────────────────────────   │
│   200 OK             │ Success - done                                                   │
│   400 Bad Request    │ Don't retry (bad payload)                                        │
│   500 Server Error   │ Retry up to 3 times with exponential backoff                     │
│   Timeout            │ Retry up to 3 times                                              │
│   All retries fail   │ Send to DLQ, On-Prem can poll GET /documents/{job_id}           │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## On-Prem Callback Endpoint Specification

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         ON-PREM CALLBACK ENDPOINT SPEC                                  │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Endpoint:  POST /api/docgen/callback                                                  │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Request Headers:                                                                      │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Content-Type: application/json                                                        │
│   X-DocGen-Job-Id: job-abc-123                                                          │
│   X-DocGen-Timestamp: 2025-01-12T14:35:42Z                                              │
│   X-DocGen-Signature: sha256=a1b2c3d4e5f6...                                            │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Signature Verification:                                                               │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   data_to_sign = "{timestamp}.{json_payload}"                                           │
│   expected_sig = HMAC-SHA256(secret, data_to_sign)                                      │
│   verify: request_signature == "sha256=" + expected_sig                                 │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Response Codes:                                                                       │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   200 OK              │ { "received": true }                │ Success                   │
│   400 Bad Request     │ { "error": "Invalid signature" }    │ Won't retry               │
│   500 Internal Error  │ { "error": "DB unavailable" }       │ Will retry                │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## JSON Response Structure

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              CALLBACK JSON PAYLOAD                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   SUCCESS Response:                                                                     │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   {                                                                                     │
│     "job_id": "job-abc-123",                                                            │
│     "status": "COMPLETED",                                                              │
│     "created_at": "2025-01-12T14:30:00Z",                                               │
│     "completed_at": "2025-01-12T14:35:42Z",                                             │
│     "processing_time_seconds": 342,                                                     │
│                                                                                         │
│     "request": {                                                                        │
│       "doc_type": "summary_report",                                                     │
│       "reference_id": "case-12345",                                                     │
│       "source_files_count": 50                                                          │
│     },                                                                                  │
│                                                                                         │
│     "document": {                                                                       │
│       "title": "דוח סיכום - תיק 12345",                                                │
│       "format": "structured_json",                                                      │
│                                                                                         │
│       "sections": [                                                                     │
│         {                                                                               │
│           "id": "intro",                                                                │
│           "title": "מבוא",                                                              │
│           "order": 1,                                                                   │
│           "content": "זהו דוח סיכום המתאר את...",                                      │
│           "paragraphs": [                                                               │
│             { "order": 1, "text": "פסקה ראשונה..." },                                  │
│             { "order": 2, "text": "פסקה שנייה..." }                                    │
│           ]                                                                             │
│         },                                                                              │
│         {                                                                               │
│           "id": "background",                                                           │
│           "title": "רקע",                                                               │
│           "order": 2,                                                                   │
│           "content": "...",                                                             │
│           "paragraphs": [...]                                                           │
│         },                                                                              │
│         {                                                                               │
│           "id": "findings",                                                             │
│           "title": "ממצאים",                                                            │
│           "order": 3,                                                                   │
│           "content": "...",                                                             │
│           "paragraphs": [...]                                                           │
│         },                                                                              │
│         {                                                                               │
│           "id": "recommendations",                                                      │
│           "title": "המלצות",                                                            │
│           "order": 4,                                                                   │
│           "content": "...",                                                             │
│           "paragraphs": [...]                                                           │
│         }                                                                               │
│       ],                                                                                │
│                                                                                         │
│       "metadata": {                                                                     │
│         "word_count": 2500,                                                             │
│         "source_files_used": 45,                                                        │
│         "pii_redactions_count": 23,                                                     │
│         "generation_model": "claude-3-sonnet"                                           │
│       }                                                                                 │
│     }                                                                                   │
│   }                                                                                     │
│                                                                                         │
│   FAILED Response:                                                                      │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   {                                                                                     │
│     "job_id": "job-abc-123",                                                            │
│     "status": "FAILED",                                                                 │
│     "created_at": "2025-01-12T14:30:00Z",                                               │
│     "failed_at": "2025-01-12T14:33:15Z",                                                │
│     "generation_model": "claude-3-sonnet",                                              │
│                                                                                         │
│     "request": {                                                                        │
│       "doc_type": "summary_report",                                                     │
│       "reference_id": "case-12345",                                                     │
│       "source_files_count": 50                                                          │
│     },                                                                                  │
│                                                                                         │
│     "error": {                                                                          │
│       "code": "OCR_EXTRACTION_FAILED",                                                  │
│       "message": "Failed to extract text from document",                                │
│       "failed_step": "ExtractAllOCR",                                                   │
│       "details": {                                                                      │
│         "failed_file": "doc23.pdf",                                                     │
│         "reason": "Corrupted PDF file"                                                  │
│       }                                                                                 │
│     },                                                                                  │
│                                                                                         │
│     "partial_results": {                                                                │
│       "files_processed": 22,                                                            │
│       "files_failed": 1,                                                                │
│       "files_remaining": 27                                                             │
│     }                                                                                   │
│   }                                                                                     │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## Error Handling

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                   ERROR HANDLING                                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   ┌─────────────┐       ┌─────────────┐       ┌─────────────┐       ┌─────────────┐    │
│   │  Any Step   │──────▶│   Catch     │──────▶│   Lambda    │──────▶│  Callback   │    │
│   │   Fails     │       │   Block     │       │error_handler│       │  (FAILED)   │    │
│   └─────────────┘       └─────────────┘       └─────────────┘       └──────┬──────┘    │
│                                                      │                     │            │
│                                                      │                     ▼            │
│                                                      │              ┌─────────────┐     │
│                                                      │              │   On-Prem   │     │
│                                                      │              │  receives   │     │
│                                                      │              │   error     │     │
│                                                      │              └─────────────┘     │
│                                                      │                                  │
│                                                      ▼                                  │
│                                               ┌─────────────┐                           │
│                                               │ CloudWatch  │                           │
│                                               │   Alarm     │                           │
│                                               └─────────────┘                           │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Error Codes:                                                                          │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   VALIDATION_FAILED       │ Invalid request parameters                                  │
│   TEMPLATE_NOT_FOUND      │ doc_type not found in DynamoDB templates                    │
│   METADATA_FETCH_FAILED   │ Failed to fetch from On-Prem API                            │
│   FILE_FETCH_FAILED       │ Failed to download file                                     │
│   OCR_EXTRACTION_FAILED   │ DocuPipe processing failed                                  │
│   SUMMARIZATION_FAILED    │ Bedrock summarization failed                                │
│   PII_REDACTION_FAILED    │ PII detection/redaction failed                              │
│   GENERATION_FAILED       │ Paragraph generation failed                                 │
│   ASSEMBLY_FAILED         │ Document assembly failed                                    │
│   CALLBACK_FAILED         │ Failed to send callback (after retries)                     │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## S3 Bucket Structure

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                  S3 BUCKET STRUCTURE                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   docgen-{env}-bucket/                                                                  │
│   │                                                                                     │
│   ├── jobs/                                                                             │
│   │   └── {job_id}/                                                                     │
│   │       ├── metadata/                                                                 │
│   │       │   └── metadata.json         # Step 2 output                                │
│   │       │                                                                             │
│   │       ├── files/                                                                    │
│   │       │   ├── doc1.pdf              # Step 3 output (original files)               │
│   │       │   ├── doc2.pdf                                                              │
│   │       │   └── ...                                                                   │
│   │       │                                                                             │
│   │       ├── extracted/                                                                │
│   │       │   ├── doc1.json             # Step 4 output (OCR text)                     │
│   │       │   ├── doc2.json                                                             │
│   │       │   └── ...                                                                   │
│   │       │                                                                             │
│   │       ├── summary/                                                                  │
│   │       │   ├── doc1_summary.json     # Step 5 output (per-file summaries)           │
│   │       │   ├── doc2_summary.json                                                     │
│   │       │   └── combined_summary.json # Merged summary                               │
│   │       │                                                                             │
│   │       ├── redacted/                                                                 │
│   │       │   └── redacted_summary.json # Step 6 output                                │
│   │       │                                                                             │
│   │       ├── paragraphs/                                                               │
│   │       │   ├── section_1.json        # Step 7 output                                │
│   │       │   ├── section_2.json                                                        │
│   │       │   └── ...                                                                   │
│   │       │                                                                             │
│   │       └── final/                                                                    │
│   │           └── document.json         # Step 8 output (final document)               │
│   │                                                                                     │
│   └── templates/                         # Document templates (backup)                  │
│       ├── summary_report.json                                                           │
│       ├── legal_brief.json                                                              │
│       └── ...                                                                           │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## Lambda Code Structure

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                LAMBDA CODE STRUCTURE                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   docgen-lambdas/                                                                       │
│   │                                                                                     │
│   ├── layers/                                                                           │
│   │   └── shared/                                                                       │
│   │       └── python/                                                                   │
│   │           └── docgen_common/                                                        │
│   │               ├── __init__.py                                                       │
│   │               ├── s3_utils.py          # S3 read/write helpers                     │
│   │               ├── models.py            # Pydantic models                           │
│   │               ├── errors.py            # Custom exceptions                         │
│   │               ├── callback.py          # Callback utilities                        │
│   │               ├── bedrock.py           # Bedrock client wrapper                    │
│   │               └── config.py            # Environment config (incl. model ID)       │
│   │                                                                                     │
│   ├── functions/                                                                        │
│   │   ├── data_fetcher/                                                                 │
│   │   │   ├── handler.py                   # fetch_metadata(), fetch_file()            │
│   │   │   └── requirements.txt             # requests, boto3                           │
│   │   │                                                                                 │
│   │   ├── ocr_processor/                                                                │
│   │   │   ├── handler.py                   # extract_ocr(), poll_docupipe()            │
│   │   │   └── requirements.txt             # requests, boto3                           │
│   │   │                                                                                 │
│   │   ├── ai_processor/                                                                 │
│   │   │   ├── handler.py                   # summarize(), redact_pii(), gen_paragraph()│
│   │   │   └── requirements.txt             # boto3 (bedrock-runtime)                   │
│   │   │                                                                                 │
│   │   └── doc_builder/                                                                  │
│   │       ├── handler.py                   # get_template(), assemble_doc()            │
│   │       ├── callback.py                  # send_callback(), retry logic              │
│   │       └── requirements.txt             # boto3, requests                           │
│   │                                                                                     │
│   ├── state_machine/                                                                    │
│   │   └── definition.asl.json              # Step Functions ASL                        │
│   │                                                                                     │
│   └── template.yaml                        # SAM template                              │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## DynamoDB Templates Table Schema

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                          DYNAMODB TEMPLATES TABLE                                       │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Table: docgen-templates                                                               │
│   Primary Key: doc_type (String)                                                        │
│                                                                                         │
│   Example Item:                                                                         │
│   {                                                                                     │
│     "doc_type": "summary_report",                                                       │
│     "display_name": "דוח סיכום",                                                        │
│     "version": "1.0",                                                                   │
│                                                                                         │
│     "api_endpoints": {                                                                  │
│       "metadata_url": "/api/cases/{reference_id}/metadata",                             │
│       "files_list_url": "/api/cases/{reference_id}/files",                              │
│       "file_content_url": "/api/files/{file_id}/content"                                │
│     },                                                                                  │
│                                                                                         │
│     "sections": [                                                                       │
│       {                                                                                 │
│         "id": "intro",                                                                  │
│         "title": "מבוא",                                                                │
│         "order": 1,                                                                     │
│         "required": true                                                                │
│       },                                                                                │
│       {                                                                                 │
│         "id": "background",                                                             │
│         "title": "רקע",                                                                 │
│         "order": 2,                                                                     │
│         "required": true                                                                │
│       },                                                                                │
│       ...                                                                               │
│     ],                                                                                  │
│                                                                                         │
│     "instructions": {                                                                   │
│       "intro": "כתוב מבוא קצר המסביר את מטרת הדוח...",                                  │
│       "background": "תאר את הרקע לתיק...",                                              │
│       ...                                                                               │
│     },                                                                                  │
│                                                                                         │
│     "pii_rules": {                                                                      │
│       "redact_names": true,                                                             │
│       "redact_id_numbers": true,                                                        │
│       "redact_addresses": true,                                                         │
│       "custom_patterns": ["pattern1", "pattern2"]                                       │
│     },                                                                                  │
│                                                                                         │
│     "created_at": "2025-01-01T00:00:00Z",                                               │
│     "updated_at": "2025-01-12T10:30:00Z"                                                │
│   }                                                                                     │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## API Examples

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              API REQUEST/RESPONSE EXAMPLES                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   POST /documents (Start Job)                                                           │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Request:                                                                              │
│   {                                                                                     │
│     "doc_type": "summary_report",                                                       │
│     "reference_id": "case-12345",                                                       │
│     "callback_url": "https://onprem.company.local/api/docgen/callback"                  │
│   }                                                                                     │
│                                                                                         │
│   Response (immediate):                                                                 │
│   {                                                                                     │
│     "job_id": "job-abc-123",                                                            │
│     "status": "PROCESSING"                                                              │
│   }                                                                                     │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   GET /documents/{job_id} (Fallback Polling)                                            │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Response (in progress):                                                               │
│   {                                                                                     │
│     "job_id": "job-abc-123",                                                            │
│     "status": "PROCESSING",                                                             │
│     "current_step": "ExtractAllOCR",                                                    │
│     "progress": "15/50 files processed"                                                 │
│   }                                                                                     │
│                                                                                         │
│   Response (completed - same as callback payload):                                      │
│   {                                                                                     │
│     "job_id": "job-abc-123",                                                            │
│     "status": "COMPLETED",                                                              │
│     "document": { ... }                                                                 │
│   }                                                                                     │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## Model Upgrade Path

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                               MODEL UPGRADE PATH                                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   PHASE 1: MVP (Sprint 1-2)                                                            │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Model: Claude 3 Sonnet                                                                │
│   Model ID: anthropic.claude-3-sonnet-20240229-v1:0                                    │
│                                                                                         │
│   ✅ 500 RPM - no rate limiting issues                                                  │
│   ✅ Good Hebrew support                                                                │
│   ✅ 85-90% quality of newer models                                                     │
│   ✅ Allows full pipeline testing                                                       │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│                                                                                         │
│   PHASE 2: Production (Sprint 3+) - After Quota Increase                               │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Model: Claude Sonnet 4 (Best for Hebrew)                                             │
│   Model ID: anthropic.claude-sonnet-4-20250514-v1:0                                    │
│                                                                                         │
│   ⚠️  Requires quota increase to 50-100 RPM                                             │
│   ✅ Best Hebrew language support                                                       │
│   ✅ Superior context retention                                                         │
│   ✅ Best understanding of Hebrew legal/business terms                                  │
│                                                                                         │
│   Alternative: Claude 3.5 Sonnet v2                                                     │
│   Model ID: anthropic.claude-3-5-sonnet-20241022-v2:0                                  │
│                                                                                         │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│                                                                                         │
│   CONFIGURATION CHANGE (Environment Variable Only):                                     │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│                                                                                         │
│   # config.py                                                                           │
│   BEDROCK_MODEL_ID = os.getenv(                                                        │
│       "BEDROCK_MODEL_ID",                                                               │
│       "anthropic.claude-3-sonnet-20240229-v1:0"  # Default for MVP                     │
│   )                                                                                     │
│                                                                                         │
│   # To upgrade, just change environment variable:                                       │
│   # BEDROCK_MODEL_ID=anthropic.claude-sonnet-4-20250514-v1:0                           │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## Development Checklist

### AWS Side
- [ ] Lambda Layer (shared code)
- [ ] Lambda: doc-builder (get_template - **STEP 1**)
- [ ] Lambda: data-fetcher
- [ ] Lambda: ocr-processor
- [ ] Lambda: ai-processor (Claude 3 Sonnet)
- [ ] Lambda: doc-builder + callback
- [ ] Step Functions state machine (sequential flow)
- [ ] API Gateway endpoints
- [ ] DynamoDB tables (templates with API endpoints)
- [ ] S3 bucket
- [ ] VPC + VPN configuration
- [ ] IAM roles and policies
- [ ] CloudWatch alarms
- [ ] DLQ for failed callbacks
- [ ] **REQUEST QUOTA INCREASE FOR CLAUDE 3.5/4** (start early!)

### On-Prem Side
- [ ] POST /api/docgen/callback endpoint
- [ ] Signature validation (HMAC-SHA256)
- [ ] Idempotency check (duplicate job_id)
- [ ] Save document to database
- [ ] Handle COMPLETED status
- [ ] Handle FAILED status
- [ ] Return proper HTTP codes (200/400/500)
- [ ] Logging
- [ ] Integration tests


## Cost Estimates

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                  COST ESTIMATES                                         │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Claude 3 Sonnet Pricing:                                                              │
│   • Input:  $3.00 / 1M tokens                                                          │
│   • Output: $15.00 / 1M tokens                                                         │
│                                                                                         │
│   Per Document (50 files, ~500 pages):                                                 │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   Step 5 (Summarize):  ~100K input + ~20K output = $0.30 + $0.30 = $0.60              │
│   Step 6 (Redact):     ~20K input + ~20K output  = $0.06 + $0.30 = $0.36              │
│   Step 7 (Generate):   ~30K input + ~5K output   = $0.09 + $0.08 = $0.17              │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   TOTAL BEDROCK COST: ~$1.00-1.50 per document                                         │
│                                                                                         │
│   Other Costs (estimates):                                                              │
│   • Lambda executions: ~$0.05                                                          │
│   • S3 storage: ~$0.01                                                                 │
│   • Step Functions: ~$0.05                                                             │
│   • DocuPipe (OCR): varies by contract                                                 │
│   ─────────────────────────────────────────────────────────────────────────────────    │
│   TOTAL AWS COST: ~$1.10-1.60 per document (excluding OCR)                             │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```


## Summary

| Aspect | Decision |
|--------|----------|
| **Model (MVP)** | Claude 3 Sonnet |
| **Model (Production)** | Claude Sonnet 4 (best Hebrew) |
| **Flow** | Sequential (Steps 5→6→7) |
| **Rate Limits** | 500 RPM (Claude 3 Sonnet) |
| **Processing Time** | ~5-6 minutes for 50 files |
| **Cost per Document** | ~$1.00-1.50 |
| **Action Required** | Request quota increase for future upgrade |
