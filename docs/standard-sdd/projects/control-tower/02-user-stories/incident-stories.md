# Incident Management Stories

## Scope

This document defines the user stories for the Incident Management page —
the AI-native operational command center of the Agentic SDLC Control Tower.

## Traceability

- Requirements: [incident-requirements.md](../01-requirements/incident-requirements.md)
- PRD: [agentic_sdlc_control_tower_prd_v0.9.md](../01-requirements/agentic_sdlc_control_tower_prd_v0.9.md) §11.10, §13.1, §14

---

## Story S1: View Active Incident List with Severity and AI Handling Context

As an SRE or DevOps engineer,
I want to see a list of active incidents with severity, status, and handler type,
so that I can quickly assess the current operational situation and know who is handling what.

### Acceptance Criteria

- The incident page shows a list/table of incidents in the current workspace
- Each incident row displays: ID, title, priority (P1–P4), status, handler type (AI / Human / Hybrid)
- Active incidents appear by default; resolved incidents are available via a tab or filter
- Incidents are sortable by severity, recency, and duration
- Critical incidents (P1) are visually flagged with the crimson accent
- A severity distribution summary (e.g., "2 P1, 3 P2, 1 P3") is visible at the top
- Clicking an incident opens the full detail view

### Edge Notes

- V1 uses mocked incident data; real data comes in Phase B
- Filtering by date range, handler type, and status is required; advanced search is out of scope
- Pagination is not required for V1 (expect <50 active incidents per workspace)

### Requirement Refs

REQ-INC-80, REQ-INC-81, REQ-INC-10

---

## Story S2: Review Incident Header and AI Control Posture

As a team lead or SRE,
I want to see the full incident header with AI handling context, autonomy level, and control mode,
so that I understand who is in control, what AI is allowed to do, and where the incident stands.

### Acceptance Criteria

- Incident detail view shows a header section with: ID, title, priority, current status
- Header displays handler type: AI, Human, or Hybrid — with visual indicator
- Header shows current autonomy level (e.g., "Level 3: Auto-execute with post-approval")
- Header shows current control mode: Auto / Approval / Manual
- Key timestamps are visible: detected, acknowledged, resolved (or "in progress" duration)
- Status follows the defined state machine: Detected → AI Investigating → AI Diagnosed → Action Proposed → Pending Approval → Executing → Resolved → Learning → Closed
- Escalated and Manual Override states are supported

### Edge Notes

- V1 does not support changing autonomy level from the UI (display only)
- State machine transitions are reflected in mock data to demonstrate each state

### Requirement Refs

REQ-INC-10, REQ-INC-11, REQ-INC-51

---

## Story S3: Follow AI Diagnosis in Real-Time Feed

As an SRE,
I want to follow the AI's diagnostic reasoning as a live feed,
so that I can understand what the AI is investigating, what it found, and what it concludes.

### Acceptance Criteria

- Incident detail includes an "AI Diagnosis" card showing a chronological feed
- Each feed entry has: timestamp, text description, entry type (analysis / finding / suggestion / conclusion)
- Feed entries appear in log-style format using the technical monospace font (JetBrains Mono)
- Suggestions are visually distinguished (e.g., cyan accent color)
- Feed shows detected signals, their sources, and correlation with recent changes
- Root cause hypothesis is displayed with a confidence indicator (e.g., "High / Medium / Low")
- Affected components and blast radius are listed when available

### Edge Notes

- V1 uses mocked feed data; real-time streaming comes in future versions
- Feed should feel "live" even with static data — entries are timestamped and ordered

### Requirement Refs

REQ-INC-30, REQ-INC-31, REQ-INC-93

---

## Story S4: Review AI/Skill Execution Timeline

As a team lead or platform admin,
I want to see a timeline of all AI skill executions related to an incident,
so that I can audit what the AI did, in what order, and with what results.

### Acceptance Criteria

- Incident detail includes a "Skill Execution Timeline" section
- Timeline shows each skill execution: skill name, start/end timestamps, status (running / completed / failed / pending approval)
- Skill types include: incident-detection, incident-correlation, incident-diagnosis, incident-remediation, incident-learning
- Each entry shows a brief input summary and output/result summary
- Evidence references are linked where available
- Timeline is ordered chronologically (oldest first)

### Edge Notes

- V1 uses mocked skill execution data
- Timeline may be combined visually with the diagnosis feed or displayed as a separate card
- Detailed skill execution logs belong to AI Center; this view shows the summary

### Requirement Refs

REQ-INC-20, REQ-INC-21

---

## Story S5: Approve or Reject AI-Proposed Actions

As an SRE or team lead,
I want to see AI-proposed actions and approve or reject them,
so that I maintain governance control over automated responses while benefiting from AI speed.

### Acceptance Criteria

- Incident detail includes an "AI Actions" card listing all actions taken or proposed
- Each action shows: description, type (automated / requires approval), execution status (pending / approved / rejected / executed / rolled back), timestamp
- Actions pending approval have visible Approve and Reject buttons
- Approving triggers the action; rejecting logs the rejection with a reason prompt
- Action history shows who approved/rejected and when
- Impact assessment is visible for each proposed action (e.g., "Scales pod replicas from 3 to 5")
- Rollback capability is indicated for reversible actions

### Edge Notes

- V1 mocks the approve/reject flow — buttons trigger state changes in mock data
- Actual API integration for action execution is Phase B
- Policy constraints that triggered the approval requirement should be visible

### Requirement Refs

REQ-INC-40, REQ-INC-41, REQ-INC-50

---

## Story S6: Review Human Governance History

As a platform admin or auditor,
I want to see the complete governance history for an incident,
so that I can audit all human decisions, overrides, and escalations.

### Acceptance Criteria

- Incident detail includes a "Governance" section
- Section shows: pending approvals, approval/rejection history, manual overrides, escalation history
- Each governance action displays: who, when, action taken, and reason
- Policy constraints that triggered governance actions are referenced
- Escalation path is visible when an incident was escalated from AI to human

### Edge Notes

- V1 uses mocked governance data
- Full audit export belongs to Audit Management in Platform Center
- Governance section may be combined visually with the AI Actions card

### Requirement Refs

REQ-INC-50, REQ-INC-52

---

## Story S7: Trace Incident Back to SDLC Chain

As a team lead or architect,
I want to see which SDLC artifacts (requirements, specs, code changes, deploys) are related to an incident,
so that I can understand the root cause in the context of the full delivery chain.

### Acceptance Criteria

- Incident detail includes a "Related SDLC Chain" section
- Chain shows related objects: Requirement, Spec, Design, Code Change, Test, Deployment
- Chain may be compressed (not all 11 nodes), but Spec is always visible
- Collapsed nodes are indicated with an expand control
- Each linked object shows: ID, title, and a navigation link to its respective page
- If no SDLC relationships exist, display "No linked artifacts" with an explanation

### Edge Notes

- V1 uses mocked SDLC chain links
- Actual cross-object linking requires backend integration in Phase B
- Navigation targets unimplemented module pages will show "Coming Soon" placeholder

### Requirement Refs

REQ-INC-60, REQ-INC-61

---

## Story S8: Read AI Learning and Prevention Recommendations

As a team lead or SRE,
I want to see what the AI learned from a resolved incident and its prevention recommendations,
so that I can improve system reliability and prevent recurrence.

### Acceptance Criteria

- Incident detail includes an "AI Learning" section (visible after incident is resolved or in Learning state)
- Section shows: confirmed root cause, pattern identified, prevention recommendations
- Prevention recommendations include suggested policy/configuration changes
- A "knowledge base entry created" indicator shows if the learning was captured
- Section references how this learning feeds back into the platform (e.g., updated detection rules)
- For unresolved incidents, this section shows "Learning will be available after resolution"

### Edge Notes

- V1 uses mocked learning data
- Actual AI learning integration is future scope
- This section provides the "why it happened and how to prevent it" narrative

### Requirement Refs

REQ-INC-70, REQ-INC-71

---

## Story S9: Navigate Between Incident List and Detail Views

As any incident page user,
I want seamless navigation between the incident list and detail views,
so that I can efficiently triage multiple incidents without losing context.

### Acceptance Criteria

- Incident list view is the default when navigating to the incident page
- Clicking an incident row transitions to the detail view
- Detail view has a back-navigation to return to the list
- Current workspace context is preserved across navigation
- Deep-linking to a specific incident via URL is supported (e.g., `/incidents/INC-0422`)
- Page renders inside the shared shell with context bar and AI panel

### Edge Notes

- V1 may use Vue Router params for incident detail routing
- Mobile-optimized navigation is out of scope

### Requirement Refs

REQ-INC-80, REQ-INC-81

---

## Story S10: Handle Loading, Error, and Empty States

As any user,
I want clear feedback when the incident page is loading, encounters an error, or has no data,
so that I always understand the current state and am never confused by a blank screen.

### Acceptance Criteria

- Loading state: skeleton placeholders or spinner shown while data loads
- Error state: error message with retry option when API call fails
- Empty state: "No active incidents" message with positive framing (e.g., "All clear — no active incidents in this workspace")
- Partial state: if some cards load but others fail, show per-card error states (not full page error)
- All states follow the shared design system visual treatment

### Edge Notes

- V1 uses mocked data so error states are only demonstrated, not triggered by real failures
- Empty state design should feel intentional, not broken

### Requirement Refs

REQ-INC-92

---

## Summary

| Story | Capability Domain | Primary Actor |
|-------|------------------|---------------|
| S1 | Incident list & triage | SRE / DevOps |
| S2 | Incident header & AI posture | Team lead / SRE |
| S3 | AI diagnosis feed | SRE |
| S4 | Skill execution timeline | Team lead / Admin |
| S5 | AI action approval | SRE / Team lead |
| S6 | Governance audit | Admin / Auditor |
| S7 | SDLC chain traceability | Team lead / Architect |
| S8 | AI learning & prevention | Team lead / SRE |
| S9 | Navigation & routing | All users |
| S10 | States (loading/error/empty) | All users |
