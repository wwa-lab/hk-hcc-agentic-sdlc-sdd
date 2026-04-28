# Incident Management — Data Flow

## Overview

This document describes how data moves through the Incident Management module
at runtime. It covers the major data flows: page load (list and detail),
governance actions (approve/reject), and the relationship between frontend
state, API responses, and backend domain objects.

---

## 1. Incident List Load

```
┌─────────────┐     navigate      ┌──────────────────┐
│  Shared      │ ─────────────►   │  IncidentList     │
│  Shell Nav   │   /incidents     │  View.vue         │
└─────────────┘                   └────────┬─────────┘
                                           │ onMounted
                                           ▼
                                  ┌──────────────────┐
                                  │  Pinia Store      │
                                  │  fetchIncidentList│
                                  └────────┬─────────┘
                                           │ fetchJson<T>
                                           ▼
                                  ┌──────────────────┐
                                  │  API Client       │
                                  │  GET /api/v1/     │
                                  │  incidents        │
                                  └────────┬─────────┘
                                           │ REST / JSON
                                           ▼
┌──────────────────────────────────���───────────────────┐
│  Spring Boot                                         │
│  IncidentController → IncidentService                │
│  Returns: ApiResponse<IncidentListDto>               │
└──────────────────────────────────────────────────────┘
```

### Response Shape

```
ApiResponse<IncidentListDto>
├── data
│   ├── severityDistribution: { p1: 2, p2: 3, p3: 1, p4: 0 }
│   └── incidents[]
│       ├── id: "INC-0422"
│       ├── title: "API Gateway Latency Spike"
│       ├── priority: "P1"
│       ├── status: "AI_INVESTIGATING"
│       ├── handlerType: "AI"
│       ├── controlMode: "Approval"
│       ├── detectedAt: "2026-04-16T09:40:00Z"
│       └── duration: "PT1H23M"
└── error: null
```

### Filter Flow

```
User selects filter ──► Store updates filterState ──► Re-fetch with query params
                                                      GET /api/v1/incidents?priority=P1&status=ACTIVE
```

---

## 2. Incident Detail Load

```
┌─────────────┐   click row    ┌──────────────────┐
│  Incident    │ ──────────►   │  IncidentDetail   │
│  List View   │  :incidentId  │  View.vue         │
└─────────────┘                └────────┬─────────┘
                                        │ onMounted
                                        ▼
                               ┌──────────────────┐
                               │  Pinia Store      │
                               │  fetchIncident    │
                               │  Detail(id)       │
                               └────────┬─────────┘
                                        │ fetchJson<T>
                                        ▼
                               ┌──────────────────┐
                               │  GET /api/v1/     │
                               │  incidents/:id    │
                               └────────┬─────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────┐
│  IncidentService.getIncidentDetail(id)               │
│  Assembles 7 sections, each wrapped in               │
│  SectionResult<T> for independent error isolation     │
└──────────────────────────────────────────────────────┘
```

### Response Shape

```
ApiResponse<IncidentDetailDto>
├── data
│   ├── header: SectionResult<IncidentHeaderDto>
│   │   └── data: { id, title, priority, status, handlerType,
│   │               controlMode, autonomyLevel, timestamps }
│   ├── diagnosis: SectionResult<DiagnosisFeedDto>
│   │   └── data: { entries[]: { timestamp, text, entryType },
│   │               rootCause: { hypothesis, confidence },
│   │               affectedComponents[] }
│   ├── skillTimeline: SectionResult<SkillTimelineDto>
│   │   └── data: { executions[]: { skillName, startTime, endTime,
│   │                                status, inputSummary, outputSummary } }
│   ├── actions: SectionResult<IncidentActionsDto>
│   │   └─��� data: { actions[]: { id, description, actionType,
│   │                             executionStatus, timestamp,
│   │                             impactAssessment, isRollbackable } }
│   ├── governance: SectionResult<GovernanceDto>
│   │   └── data: { entries[]: { actor, timestamp, actionTaken,
│   │                             reason, policyRef } }
│   ├── sdlcChain: SectionResult<SdlcChainDto>
│   │   └── data: { links[]: { artifactType, artifactId,
│   │                           artifactTitle, routePath } }
│   └── learning: SectionResult<AiLearningDto>
│       └── data: { rootCause, patternIdentified,
│                   preventionRecommendations[],
│                   knowledgeBaseEntryCreated } | null
└── error: null
```

### Card-Level Error Isolation

Each card reads its own `SectionResult<T>`:

```
IncidentDetailDto.diagnosis.data    → AI Diagnosis Card
IncidentDetailDto.diagnosis.error   → Card shows inline error

IncidentDetailDto.actions.data      → AI Actions Card
IncidentDetailDto.actions.error     → Card shows inline error
```

If one section fails on the backend, only that card shows an error. The rest
of the detail view remains functional.

---

## 3. Governance Action Flow (Approve / Reject)

```
┌────────────────┐   click       ┌──────────────────┐
��  AI Actions    │ ─────────►    │  Pinia Store      │
│  Card          │  Approve      │  approveAction    │
│  (Approve btn) │               │  (incidentId,     │
└────────────────┘               │   actionId)       │
                                 └────────┬─────────┘
                                          │
                          ┌───────────────┴───────────────┐
                          │                               │
                          ▼                               ▼
                 ┌──────────────────┐           ┌──────────────────┐
                 │  POST /api/v1/   │           │  POST /api/v1/   │
                 │  incidents/:id/  │    OR     │  incidents/:id/  │
                 │  actions/:aid/   │           │  actions/:aid/   │
                 │  approve         │           │  reject           │
                 └────────┬────────┘           │  body: { reason } │
                          │                    └────────┬──────────┘
                          ▼                             ▼
┌──────────────────────────────────────────────────────────────┐
│  IncidentService                                             │
│  1. Validate action exists and is PENDING                    │
│  2. Transition action status (APPROVED / REJECTED)           │
│  3. Create GovernanceEntry (actor, timestamp, reason)        │
│  4. If approved: transition incident status to EXECUTING     │
│  5. Return updated sections                                  │
└──────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│  Frontend refreshes:                                         │
│  - AI Actions card (updated action status)                   │
│  - Governance card (new governance entry)                    │
│  - Header card (updated incident status if transitioned)     │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. State Machine Data Flow

The incident status state machine drives data transitions across multiple cards:

```
Status Transition              Cards Affected
─────────────────────────      ─────────────────────────
DETECTED → AI_INVESTIGATING    Header (status), Timeline (new skill entry)
AI_INVESTIGATING → AI_DIAGNOSED Header (status), Diagnosis (root cause)
AI_DIAGNOSED → ACTION_PROPOSED  Header (status), Actions (new action)
ACTION_PROPOSED → PENDING_APPROVAL  Header (status), Actions (pending badge)
PENDING_APPROVAL → EXECUTING   Header, Actions, Governance (new entry)
EXECUTING → RESOLVED           Header (status + resolved timestamp)
RESOLVED → LEARNING            Header (status), Learning (content appears)
LEARNING → CLOSED              Header (status)
Any → ESCALATED                Header (status + handler changes to Human)
Any → MANUAL_OVERRIDE          Header, Governance (override entry)
```

---

## 5. Navigation Data Flow

```
┌─────────────────────────────────────────────────┐
│  Entry Points                                    │
│                                                  │
│  Shell Nav (/incidents)    → Incident List       │
│  Dashboard Stability Card  → Incident List       │
│  Deep Link (/incidents/INC-0422) → Detail View   │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  Exit Points (from Incident Detail)              │
│                                                  │
│  Back button              → Incident List        │
│  SDLC Chain Link click    → Module Page          │
│  Shell Nav click          → Other Module         │
└─────────────────────────────────────────────────┘
```

### SDLC Chain Link Navigation

```
SdlcChainLink { artifactType: "deploy", artifactId: "DEP-041", routePath: "/deployments" }
    → router.push(routePath)
    → If module is implemented: renders module page
    → If module is not implemented: renders PlaceholderView with "Coming Soon"
```

---

## 6. Frontend Type ↔ Backend DTO Mapping

| Frontend Type (TypeScript) | Backend DTO (Java Record) | API Section |
|---|---|---|
| `IncidentListItem` | `IncidentListItemDto` | incidents[] |
| `SeverityDistribution` | `SeverityDistributionDto` | severityDistribution |
| `IncidentHeader` | `IncidentHeaderDto` | header |
| `DiagnosisEntry` | `DiagnosisEntryDto` | diagnosis.entries[] |
| `RootCauseHypothesis` | `RootCauseHypothesisDto` | diagnosis.rootCause |
| `SkillExecution` | `SkillExecutionDto` | skillTimeline.executions[] |
| `IncidentAction` | `IncidentActionDto` | actions.actions[] |
| `GovernanceEntry` | `GovernanceEntryDto` | governance.entries[] |
| `SdlcChainLink` | `SdlcChainLinkDto` | sdlcChain.links[] |
| `AiLearning` | `AiLearningDto` | learning |
| `SectionResult<T>` | `SectionResultDto<T>` | all sections |

All field names use camelCase in both TypeScript and JSON serialization to maintain
a 1:1 mapping between frontend and backend.

---

## 7. Phase A vs Phase B Data Sources

| Phase | Data Source | Store Behavior |
|---|---|---|
| Phase A | `mockData.ts` in frontend | Store imports mock data directly; no API call |
| Phase B | `GET /api/v1/incidents[/:id]` | Store calls API via `fetchJson<T>`; mock data retained as fallback |

The store's `fetchIncidentList()` and `fetchIncidentDetail(id)` functions check a
feature flag or environment variable to decide between mock and live data, following
the same pattern established in the Dashboard store.
