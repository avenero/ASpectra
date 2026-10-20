# The overlay bounded context — `testcase`

```mermaid
flowchart TB
    subgraph api["testcase/api"]
        C1["CanonicalTestCaseController"]
        C2["ConnectorController"]
        C3["ExecutionPlanController"]
        C4["IntegrationSyncJobController"]
        C5["IntegrationDetectController"]
    end
    subgraph app["testcase/application"]
        A1["ConnectorImportApplicationService"]
        A2["CanonicalTestCaseApplicationService"]
        A3["ExecutionPlanApplicationService"]
        A4["IntegrationSyncJobApplicationService"]
        A5["TestCaseAiNormalizationService"]
        A5b["TestCaseAiStepMappingService"]
        A6["SourceCaseNormalizer<br/>CatalogStepMatcher<br/>CanonicalExecutability / DerivedQuality"]
        A7["TestCaseAccessGuard"]
        A8["ConnectorRegistry / ConnectorRateLimiter"]
    end
    subgraph dom["testcase/domain - PROVIDER NEUTRAL (G27)"]
        D1["CanonicalTestCase aggregate"]
        D2["TestManagementConnectorPort"]
        D3["ImportScope / ScopeType / ResultTarget<br/>RunResultSet / ConnectorCapabilities"]
        D4["One port per AI capability<br/>AiNormalizationPort, AiStepMappingPort, ..."]
    end
    subgraph infra["testcase/infrastructure"]
        I1["connector/jira - JiraXrayServerConnectorAdapter"]
        I2["connector/xraycloud - XrayCloudConnectorAdapter"]
        I3["connector/testrail - TestRailConnectorAdapter<br/>built, hidden from 1.0 picker (FE allowlist)"]
        I4["persistence / projection / sync / retention"]
        I5["acl/ six Intelligence*Adapter classes<br/>(Normalization, StepMapping, DuplicateDetection,<br/>FieldMapping, ReviewExplanation, Proposal)"]
        I6["acl/ CatalogStepCatalogAdapter, ProjectPermission..."]
    end
    api --> app --> dom
    infra -.->|"implements ports"| dom
```

`TestCaseAiNormalizationService` (`testcase.normalize`) and `TestCaseAiStepMappingService`
(`step.map.propose`) each follow the same propose/accept shape as every AI capability in the product:
`propose()` writes only an `AiProposal`; `accept()` is the only call that mutates the canonical case, and
re-checks `baseVersion` against the case's current `rowVersion` before applying anything.
`TestCaseAiStepMappingService` proposes mappings only for step positions `CatalogStepMatcher` left
unmapped — the deterministic matcher's own mappings are never overwritten, whatever the proposal says.

TestRail's connector adapter (`TestRailConnectorAdapter`, `PROVIDER_TYPE="TESTRAIL"`,
`CONNECTOR_ID="testrail"`) is a `TestManagementConnectorPort` Spring bean, auto-discovered by
`ConnectorRegistry` exactly like Jira and Xray Cloud — there is no special-case wiring to bypass it. The
only gate withholding it from a new project is an FE-side allowlist (see
[../architectureFE/01-rings-and-modules.md](../architectureFE/01-rings-and-modules.md)), not a missing
backend implementation.
