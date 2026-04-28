# Project Context

## Purpose

This document captures the stable background for the project so downstream
requirements, specs, and tasks do not need to restate it repeatedly.

## Product Summary

`Agentic SDLC Control Tower` is an AI-native enterprise web platform for
visualizing, governing, and operating the end-to-end software delivery chain.

Operational short name:

- `SDLC Tower`

Formal long name:

- `Agentic SDLC Control Tower`

## Problem Statement

Enterprise delivery work is fragmented across multiple systems and phases:

- requirements, design, code, test, deploy, and incident data live in different tools
- AI is often treated as an assistant, not an explicit execution participant
- governance capabilities such as templates, policies, permissions, and audit are scattered
- organizations need both platform reuse and team-level isolation

The product needs to provide one governed control surface without pretending to
replace every existing tool on day one.

## Working Assumptions

- Desktop-first web application
- Multi-team environment with `Workspace` as the main isolation boundary
- Shared platform capabilities with isolated data, policy, audit, and credential scopes
- `Spec Driven Development` is the delivery model
- Existing enterprise systems remain the source of record for parts of the flow in V1

## Stable Context Model

Every business page should be able to render the same context model:

- `Workspace`
- `Application`
- `SNOW Group`
- `Project`
- `Environment`

## Delivery Principle

Before code generation, we align documents in this order:

`context -> requirements -> user stories -> spec -> architecture -> design -> tasks -> code`

## Foundation Slice

The first implementation slice for the repo foundation is:

- shared application shell

This slice is intentionally narrow and includes:

- primary navigation
- top context bar
- page title/action region
- global search, notification, and audit entry points
- persistent AI Command Panel container

It excludes page-specific business modules such as dashboard metrics, incident
diagnosis, platform configuration detail, and report content.
