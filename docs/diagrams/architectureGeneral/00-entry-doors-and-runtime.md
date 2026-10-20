# Entry doors, launch paths and the target runtime

Status: Current-verified (code, 2026-10-20) for "Today"; Intended (from [alcance.md](../../alcance.md)) for
"Target" | Owner: TBD

A·Spectra is one execution runtime with one stored scenario model, reached through several entry doors
(alcance §1.2). This page is the map that ties the code to that decision: what exists today, where it
diverges from the decided scope, and the target shape. Requirement IDs refer to
[requirements.md](../../requirements.md).

## Today: five doors, two stored models, three launch paths

```mermaid
flowchart LR
    subgraph Doors["Entry doors"]
        D1["TMS connector import<br/>POST /api/projects/{p}/connectors/{c}/imports<br/>202 + durable job"]
        D2[".feature upload<br/>POST .../canonical-tests/feature-imports"]
        D3["Swagger / Postman / user story<br/>POST /api/projects/{p}/imports/*<br/>SpecDraftService"]
        D4["Scenario builder<br/>POST /api/features/{f}/scenarios"]
        D5["Gherkin paste<br/>POST /api/features/{f}/gherkin/paste"]
    end
    subgraph Models["Stored models (BE)"]
        CAN[("testcase BC<br/>CanonicalTestCase<br/>7-state review, versions")]
        DRAFT[("imports BC<br/>ScenarioDraft<br/>NEW / ACCEPTED / REJECTED")]
        EXE[("legacy model.project<br/>TestScenario (executable rows)")]
    end
    subgraph Launch["Launch paths"]
        L1["Execution plan<br/>POST /api/projects/{p}/execution-plans<br/>review gate: EXECUTABLE"]
        L2["Direct<br/>POST /api/executions<br/>no review gate"]
        L3["Bulk execute<br/>POST /api/features/{f}/scenarios/bulk-execute<br/>no gate, no runtime config"]
    end
    SVC["ExecutionApplicationService.launch<br/>one launch service"]
    CORE["qa-platformCore<br/>in the backend JVM<br/>(mobile agent only for MOBILE with agentUrl)"]
    SYNC["Result sync<br/>only plan-path runs of connector cases"]

    D1 --> CAN
    D2 --> CAN
    D3 --> DRAFT -->|"accept writes rows directly"| EXE
    D4 --> EXE
    D5 -->|"different repository path"| EXE
    CAN -->|"projection"| EXE
    EXE --> L1 & L2 & L3 --> SVC --> CORE
    L1 -.-> SYNC
```

What the diagram shows, and where it diverges from the decided scope:

| Today | Decision | Requirement |
|---|---|---|
| Spec and story drafts live in their own model and write executable rows on accept | The pipeline terminates in the canonical model | IMB-01, IMB-02 |
| Gherkin paste uses a different repository path than the scenario builder | Paste goes through the one stored model | AUT-03 |
| Only the plan path has a review gate | Correct: review is a console convention, not a backend invariant; the gate stays on the plan path only | EXE-01, REV-06 |
| Bulk execute carries no layer, Web, reporting or HTTP-engine configuration | Bulk execute carries the full runtime configuration | EXE-03 |
| Only plan-path runs of connector cases sync, and skips are silent | Recorded runs (plan, suite, feature) of connector cases sync; every skip is reported before the run | SYN-03, SYN-08 |
| Every run except a mobile run with an agent executes in the backend JVM | Execution moves out of process onto the customer-side runner | EXE-02 |

## Target: one model, one launch service, the customer-side runner

```mermaid
flowchart LR
    subgraph Doors["Entry doors"]
        T1["TMS connector"]
        T2[".feature upload"]
        T3["Specification / user story<br/>deterministic extraction when AI is off"]
        T4["Native authoring and paste"]
    end
    CAN[("One stored scenario model<br/>canonical, versioned, 7-state review")]
    subgraph Launch["One launch service"]
        REC["Recorded run<br/>plan, suite, feature, schedule"]
        DRF["Draft run<br/>from the editor, never syncs"]
    end
    subgraph Customer["Customer network"]
        RUN["Customer-side runner<br/>generalised mobile agent, /v1/runs<br/>warm browser pool per engine<br/>element inspector"]
        SUT["Application under test<br/>devices"]
    end
    PROT{"Protected environment?<br/>higher grade required"}
    SYNC["Write-back: one result per source case<br/>connector cases only, can be turned off"]

    T1 & T2 & T3 & T4 --> CAN --> REC & DRF
    REC & DRF --> PROT --> RUN --> SUT
    REC -.-> SYNC
```

The runner is one piece of infrastructure answering four commitments at once: private-network
execution, the element inspector, run durability across deploys, and the warm browser pool (alcance §3,
§9.1). Requirements: ORG-18, EXE-02, WEB-06, NFR-01, NFR-03, NFR-05, ENV-05, SYN-01, SYN-09.
