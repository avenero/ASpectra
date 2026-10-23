# ASpectra Platform overview

## 1. What ASpectra is

ASpectra is a **multi-tenant SaaS platform that turns requirements into executable, 
catalog-bound test scenarios and runs them deterministically, with humans owning every consequential 
decision.** 
A customer brings requirements — user stories with acceptance criteria, API and product documentation, 
OpenAPI specs, Postman collections, existing `.feature` files — or simply writes test intent in their own words. 
ASpectra maps that intent onto a **published catalog of real, developer-authored executable steps**, shows the 
customer exactly what the mapping means, and only then compiles, seals and (after an explicit human approval and 
a separate explicit Run) executes it against the customer's own system under test. It returns immutable execution facts, 
evidence and a deterministic report. Product scope is defined in [../alcance.md](../alcance.md), the only source 
of truth for it; requirements derived from it are in [../requirements.md](../requirements.md).

It is a QA/test-automation product, not an agent framework. 
The governing thesis, repeated verbatim across every board:

> **AI proposes. Humans authorize. Core decides.**
> *Catalog-grounded test specifications, human-owned execution.*

## 2. What makes it different from "an LLM that writes tests"

The design is built around one refusal: **AI never produces executable code, never approves, and never 
launches.** Scenarios can only be composed from steps that already exist as real Java implementations 
published in an immutable catalog. If a requirement cannot be expressed with existing steps, that is 
surfaced as a **visible capability gap** — it is never papered over with an invented step, a guessed 
assertion or a nearest-match binding. New executable capability requires a developer to write a Core 
handler, with tests, in a released version.


## 3. Repository layout

## 4. End-to-end flow, one line per stage

## 5. Cross-repository mechanics

## 6. Complete Stack

