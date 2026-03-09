# Django-Web-Portal
┌─────────────────────┐
│      CustomUser     │
├─────────────────────┤
│ id (UUID)           │
│ email               │
│ full_name           │
│ role (ADMIN /       │
│   MARKETER /        │
│   REVIEWER)         │
│ property_name       │
│ is_active           │
│ created_at          │
└────────┬────────────┘
         │ 1
         │
         │ MANY
┌────────▼────────────┐
│    VideoUpload      │
├─────────────────────┤
│ id (UUID)           │
│ user (FK)           │
│ file_url (Azure)    │
│ caption             │
│ status (PENDING /   │
│   PROCESSING /      │
│   COMPLETE /        │
│   FAILED)           │
│ uploaded_at         │
└────────┬────────────┘
         │ 1
         │
         │ 1
┌────────▼────────────┐
│   AnalysisReport    │
├─────────────────────┤
│ id (UUID)           │
│ video (FK)          │
│ overall_status      │
│  (APPROVED /        │
│   WARNINGS /        │
│   AUTO_REJECTED)    │
│ created_at          │
└────────┬────────────┘
         │ 1
         │
         │ MANY
┌────────▼────────────┐
│   AnalysisResult    │
├─────────────────────┤
│ id (UUID)           │
│ report (FK)         │
│ guideline (FK)      │
│ status (PASS /      │
│   FAIL / WARNING)   │
│ detail              │
│ timestamp           │
└─────────────────────┘

┌─────────────────────┐
│  GuidelineCategory  │
├─────────────────────┤
│ id                  │
│ name (VISUAL /      │
│   AUDIO / CAPTION / │
│   METADATA)         │
└────────┬────────────┘
         │ 1
         │
         │ MANY
┌────────▼────────────┐
│     Guideline       │
├─────────────────────┤
│ id                  │
│ category (FK)       │
│ rule_text           │
│ severity            │
│  (AUTO_REJECT /     │
│   WARNING)          │
│ is_active           │
└─────────────────────┘


AUTH
────────────────────────────────────────────────
POST   /api/auth/register/           Admin only
POST   /api/auth/login/              Public
POST   /api/auth/logout/             Authenticated
POST   /api/auth/token/refresh/      Authenticated
GET    /api/auth/me/                 Authenticated

VIDEOS
────────────────────────────────────────────────
POST   /api/videos/upload/           Marketer + Admin
GET    /api/videos/                  Own videos (Marketer) / All (Admin+Reviewer)
GET    /api/videos/{id}/             Detail
DELETE /api/videos/{id}/             Admin only

REPORTS
────────────────────────────────────────────────
GET    /api/reports/{video_id}/      Owner + Admin + Reviewer
GET    /api/reports/                 Admin + Reviewer (all reports)

GUIDELINES
────────────────────────────────────────────────
GET    /api/guidelines/              Admin only
POST   /api/guidelines/              Admin only
PUT    /api/guidelines/{id}/         Admin only
DELETE /api/guidelines/{id}/         Admin only

┌─────────────────────────────────────────────────────────────┐
│                        REACT FRONTEND                        │
└──────────────────────────┬──────────────────────────────────┘
                           │
                  JWT in Authorization Header
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    DJANGO REST API                            │
│                                                              │
│  1. Validate JWT token                                       │
│  2. Check user role/permissions                              │
│  3. Save VideoUpload record (status = PENDING)               │
│  4. Upload file → Azure Blob Storage                         │
│  5. Trigger Celery task (async)                              │
│  6. Return { video_id, status: "PENDING" } immediately       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                    Celery Task Queue
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    CELERY WORKER                              │
│                                                              │
│  Status updated → PROCESSING                                 │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ PIPELINE                                             │    │
│  │                                                      │    │
│  │  STEP 1 — ffmpeg_service                            │    │
│  │    • Extract video metadata                         │    │
│  │    • Check orientation (width vs height)            │    │
│  │    • Check video duration                           │    │
│  │    • Extract first & last frames                    │    │
│  │    • Extract audio track                            │    │
│  │    • Sample frames every N seconds                  │    │
│  │                                                      │    │
│  │  STEP 2 — audio_service                             │    │
│  │    • Whisper → transcribe audio                     │    │
│  │    • Check for foul language                        │    │
│  │    • Detect artist song duration                    │    │
│  │    • Check for narration vs music                   │    │
│  │                                                      │    │
│  │  STEP 3 — vision_service                            │    │
│  │    • Send frames to Claude Vision API               │    │
│  │    • Guidelines injected into system prompt         │    │
│  │    • Detect: sparkle filters, black screens,        │    │
│  │      toilet seats, generic graphics                 │    │
│  │                                                      │    │
│  │  STEP 4 — caption_service                           │    │
│  │    • Send caption to Claude Text API                │    │
│  │    • Detect: empty, generic content, duplicate,     │    │
│  │      missing CTA, grammar issues                    │    │
│  │                                                      │    │
│  │  STEP 5 — report_service                            │    │
│  │    • Aggregate all results                          │    │
│  │    • Determine overall_status                       │    │
│  │    • Save AnalysisReport + AnalysisResults to DB    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  Status updated → COMPLETE                                   │
└──────────────────────────────────────────────────────────────┘
                           │
              React polls GET /api/reports/{video_id}/
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    REACT DASHBOARD                            │
│   Displays: Overall Status + Per-Rule Pass/Fail/Warning      │
└──────────────────────────────────────────────────────────────┘