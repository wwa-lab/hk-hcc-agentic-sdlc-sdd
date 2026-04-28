# Requirement Control Plane User Stories

## Purpose

This document converts the Requirement Control Plane requirements into user
stories and acceptance criteria. It focuses on the next-stage alignment of
Requirement Management with the platform control-plane model.

## Traceability

- Requirements: [requirement-control-plane-requirements.md](../01-requirements/requirement-control-plane-requirements.md)
- Existing Requirement SDD: [requirement-requirements.md](../01-requirements/requirement-requirements.md)

## Personas

- **Business Analyst / Business User**: reviews business sources and SDD docs,
  comments, and approves or requests changes.
- **Developer / Technical Lead**: uses GitHub and CLI agents to generate,
  update, and review SDD artifacts.
- **Delivery Manager**: monitors freshness and traceability.
- **Platform Admin**: configures profiles and source integrations.

## Stories

### Story RCP-01: Register BAU source references

As a Business Analyst, I want to attach Jira and Confluence references to a
requirement, so that the team can trace SDD documents back to BAU sources.

**Requirement refs:** REQ-RCP-10, REQ-RCP-11

**Acceptance criteria:**

- Given I am on Requirement detail, when I add a Jira or Confluence URL, then a
  source reference card is created.
- Given a source reference is created, then it shows source type, title, URL,
  external ID when available, fetched time, and freshness state.
- Given the URL cannot be resolved, then the source reference remains visible
  with an error state and retry action.

### Story RCP-02: View source freshness

As a Delivery Manager, I want to see whether Jira or Confluence changed after
SDD documents were generated, so that stale requirements are visible.

**Requirement refs:** REQ-RCP-12, REQ-RCP-60

**Acceptance criteria:**

- Given a source has `sourceUpdatedAt` after a document commit time, then the
  document shows Source Changed.
- Given source freshness is unknown, then the page labels it Unknown instead of
  implying it is fresh.
- Given source metadata is refreshed, then freshness indicators update without
  losing existing comments or approvals.

### Story RCP-03: View latest SDD document from GitHub

As a Business User, I want to read the latest SDD document from GitHub inside
Control Tower, so that I do not need to navigate GitHub for basic review.

**Requirement refs:** REQ-RCP-20, REQ-RCP-21, REQ-RCP-23, REQ-RCP-24

**Acceptance criteria:**

- Given a document is indexed, when I open it, then the UI renders Markdown
  fetched from GitHub.
- Given the document opens, then the page shows repo, path, commit SHA, and an
  Open in GitHub link.
- Given GitHub fetch fails, then the document panel shows a retryable error
  without breaking the rest of Requirement detail.

### Story RCP-04: Comment on a document version

As a Business User, I want to comment on the document version I reviewed, so
that feedback is auditable even if the document changes later.

**Requirement refs:** REQ-RCP-30, REQ-RCP-32, REQ-RCP-33

**Acceptance criteria:**

- Given I am viewing a document, when I submit a comment, then the comment stores
  document ID, commit SHA, blob SHA, reviewer, text, and timestamp.
- Given the document later changes, then my old comment remains attached to the
  previous version.
- Given comments exist for multiple versions, then the review history shows the
  version each comment applies to.

### Story RCP-05: Approve or request changes

As a Business User, I want to approve or request changes on a document version,
so that engineering can proceed with clear business sign-off.

**Requirement refs:** REQ-RCP-31, REQ-RCP-32, REQ-RCP-61

**Acceptance criteria:**

- Given I approve a document, then approval is tied to the current commit and
  blob SHA.
- Given the document changes after approval, then the approval is marked stale.
- Given I request changes, then the requirement shows an open business review
  issue until a later version is reviewed.

### Story RCP-06: Render Standard Java SDD stages

As a Developer, I want Java projects to show the familiar Standard SDD document
chain, so that existing SDD conventions continue to work.

**Requirement refs:** REQ-RCP-40, REQ-RCP-41, REQ-RCP-43

**Acceptance criteria:**

- Given the active profile is `standard-java-sdd`, then Requirement detail shows
  Requirement, User Story, Spec, Architecture, Design, Tasks, Code, and Test
  stages.
- Given a stage has no document indexed, then it displays Missing Document.
- Given a stage has one or more GitHub documents, then it displays those
  documents with status and freshness.

### Story RCP-07: Render IBM i SDD stages

As an IBM i delivery team member, I want Requirement Management to show the IBM i
document chain, so that BAU enhancement work is not forced into a Java SDD shape.

**Requirement refs:** REQ-RCP-40, REQ-RCP-42, REQ-RCP-43

**Acceptance criteria:**

- Given the active profile is `ibm-i`, then the UI shows Requirement
  Normalizer, Functional Spec, Technical Design, Program Spec, File Spec, UT
  Plan, Test Scaffold, and Review stages.
- Given an IBM i stage has tier information, then the UI shows L1/L2/L3 where
  available.
- Given an IBM i review gate is missing, then the UI highlights the missing gate
  without blocking document viewing.

### Story RCP-08: Request a CLI agent run

As a Developer or Technical Lead, I want to request an agent run from
Requirement detail, so that a CLI agent can generate or update SDD documents
from the correct sources.

**Requirement refs:** REQ-RCP-50, REQ-RCP-51, REQ-RCP-52

**Acceptance criteria:**

- Given I request an agent run, then Control Tower creates an execution manifest.
- Given the manifest is created, then it includes sources, profile, repo, branch,
  document references, output expectations, and constraints.
- Given an agent run is requested, then UI status is Queued or Awaiting CLI
  Execution rather than pretending the UI generated the artifact.

### Story RCP-09: Agent callback updates status and artifacts

As a Developer, I want the CLI agent to report status and artifact links back to
Control Tower, so that business users can see the outcome.

**Requirement refs:** REQ-RCP-53, REQ-RCP-54

**Acceptance criteria:**

- Given an agent completes successfully, then Requirement detail shows completed
  status, summary, GitHub PR URL, and artifact links.
- Given an agent fails, then Requirement detail shows failure reason and any
  partial artifacts.
- Given an agent reports a stale context error, then the UI shows which manifest
  input was stale.

### Story RCP-10: View traceability across sources, docs, reviews, and runs

As a Delivery Manager, I want a single traceability view, so that I can see how a
requirement moved from BAU input to SDD docs and review decisions.

**Requirement refs:** REQ-RCP-62, REQ-RCP-63

**Acceptance criteria:**

- Given a requirement has sources, docs, reviews, and agent runs, then the
  traceability view connects them in order.
- Given an expected document is missing, then it appears as Missing Document.
- Given a source or document is stale, then the traceability view includes the
  freshness state.

### Story RCP-11: Preserve Normalize with AI as assisted intake

As a Business Analyst, I want Normalize with AI to remain available for messy
input, so that I can create structured requirement drafts while still keeping
GitHub docs as the SDD source of truth.

**Requirement refs:** REQ-RCP-01, REQ-RCP-02, REQ-RCP-10

**Acceptance criteria:**

- Given I normalize pasted input, then the output is a draft or source
  extraction, not an authoritative SDD document.
- Given I confirm the draft, then source reference and requirement metadata are
  created.
- Given downstream SDD docs are needed, then the UI points to agent run request
  or GitHub docs rather than treating the draft as final SDD content.

### Story RCP-12: Admin configures or selects SDD profile

As a Platform Admin, I want projects to resolve an SDD profile, so that each
system type gets the right document chain and skill guidance.

**Requirement refs:** REQ-RCP-40, REQ-RCP-44, REQ-RCP-74

**Acceptance criteria:**

- Given a project has a configured profile, then Requirement detail renders that
  profile.
- Given no project profile is configured, then Standard Java SDD is used as a
  default.
- Given a profile defines stages and skills, then UI renders stages from the
  profile and creates manifests using those skill bindings.
