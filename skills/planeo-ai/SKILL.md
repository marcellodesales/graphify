---
name: planeo-ai
description: Build, extend, query, classify, and explain Planeo DevSecOps Secure Software Supply Chain Graphs for Cloud Native systems. Use this skill whenever the user asks about planeo-graph, source-to-CI-to-CD-to-runtime traceability, Graphify source graphs, CNCF terminology, 12-factor app rules, Kubernetes resources, Helm, Kustomize, Ksonnet, GitOps, SBOMs, SLSA provenance, build attestations, GitHub Actions, Docker images, missing CI/CD links, or secure software supply-chain gaps in a repository.
---

# Planeo AI Secure Supply Chain Graph

Use this skill to turn a repository, a set of repositories, or an existing Graphify knowledge graph into a higher-level Planeo DevSecOps Secure Software Supply Chain Graph.

The core idea is that Graphify already builds the source-code and document knowledge graph. Planeo adds the Cloud Native supply-chain graph above it: components, CI workflows, build artifacts, images, SBOMs, attestations, deployment definitions, Kubernetes and CNCF runtime objects, policy results, observability signals, user graph classifications, and gaps.

## Mental model

Treat the Graphify graph as the source intelligence layer, not the whole supply chain.

Planeo should produce a higher-level graph called `planeo-graph` that references one or more Graphify graphs:

```text
Repository
  -> SourceGraph
  -> Component / Microservice
  -> Build Definition
  -> Build Run
  -> Artifact / Image
  -> SBOM
  -> Attestation / Provenance
  -> Deployment Definition
  -> CD Reconciler
  -> Cloud Native Runtime Workload
  -> Observability / Policy / Risk
  -> User Graph Classification
```

The graph should answer practical questions such as:

- Which source files implement this deployed workload?
- Which CI workflow built this image?
- Which commit produced this artifact?
- Does this artifact have an SBOM and provenance?
- Which Helm chart deploys this image?
- Is this workload reconciled by ArgoCD, Flux, Planeo, or manual operations?
- Which services have source and Dockerfiles but no CI image build?
- Which deployed images cannot be traced back to a workflow run?
- Which Cloud Native category does this component belong to in the current user graph?
- Which 12-factor principles does this service satisfy or violate?
- Which supply-chain controls are missing?

## Planeo map classification goal

The Planeo graph exists so the Planeo agent can understand a repository, classify everything it finds, and place each finding on the user's current Cloud Native map.

For every source file, service, package, CI step, artifact, deployment definition, and runtime object, try to classify:

- what it is
- where it lives in the repository
- what role its directory plays in the repository
- which Cloud Native / CNCF category it belongs to
- which application or platform capability it implements
- which deployment path it participates in
- which runtime object it becomes
- which security, reliability, cost, and compliance controls apply
- what evidence proves the classification

Use the user's graph as the organizing context when available. A repository should not merely produce isolated nodes; it should enrich the user's existing graph with classified, evidence-backed objects and relationships.

Important classification targets:
- repository region
- directory purpose

- application component
- platform component
- infrastructure component
- example, demo, sample, fixture, or test workload
- CI capability
- CD capability
- runtime workload
- network path
- storage dependency
- identity and access dependency
- configuration dependency
- secret dependency
- observability signal
- policy or compliance control
- security finding
- cost or efficiency concern
- operational ownership boundary

When classification is uncertain, create a `ClassificationCandidate` or `Gap` node rather than pretending the mapping is proven.

## Recursive directory classification

Classify every directory recursively before deciding what the repository's business logic is.

This matters because many repositories contain example apps, fixtures, generated artifacts, docs, test data, vendored dependencies, local graph output, CI scaffolding, or demos that should not be treated as production business logic. A directory like `sample-app/`, `examples/`, `fixtures/`, or `testdata/` may contain real source code, Dockerfiles, Helm charts, and Kubernetes manifests, but its role can still be "example/demo/test fixture" rather than "core product."

For every directory, create a `Directory` node and classify its purpose using evidence:

- path name and naming conventions
- README sections and nearby docs
- local README files
- package manifests
- Makefile/package scripts
- test file patterns
- Graphify source graph communities and god nodes
- CI/CD references
- deployment manifests
- import/dependency direction
- generated-output markers
- lockfiles and vendored paths

Directory classification should be recursive and inherited, but not blindly. A child directory inherits its parent's purpose unless stronger local evidence overrides it. For example, `sample-app/order-service/` inherits `sample-app/` as an example/demo area even though it contains valid Go service code and a Dockerfile.

Use these directory purposes:

- `core_product`
- `business_logic`
- `platform_runtime`
- `cli_entrypoint`
- `api_server`
- `web_ui`
- `agent_or_ai`
- `infrastructure`
- `deployment_config`
- `ci_cd`
- `security_supply_chain`
- `observability`
- `documentation`
- `example_demo`
- `test`
- `fixture`
- `generated`
- `build_output`
- `cache`
- `vendor_dependency`
- `local_tooling`
- `unknown`

Create directory edges:

- `Repository -> HAS_DIRECTORY -> Directory`
- `Directory -> CONTAINS_DIRECTORY -> Directory`
- `Directory -> CLASSIFIED_AS -> DirectoryPurpose`
- `Directory -> MAPS_TO_USER_GRAPH_NODE -> UserGraphNode`
- `Component -> LOCATED_IN -> Directory`
- `Component -> INHERITS_DIRECTORY_PURPOSE -> DirectoryPurpose`
- `Directory -> DESCRIBED_BY -> SourceGraph`

Add these properties to each `Directory` node:

- `path`
- `depth`
- `purpose`
- `business_logic_scope`: `core`, `supporting`, `example`, `test`, `generated`, `external`, or `unknown`
- `classification_reason`
- `inherited_from`
- `confidence`

Business logic classification should be conservative:

- Only mark a directory as `business_logic` or `core_product` when README/docs/import graph/package structure support that claim.
- Mark sample/demo/example directories as `example_demo` even when they include microservices, charts, or Dockerfiles.
- Mark `graphify-out/`, `planeo-graph/`, caches, and generated reports as `generated` or `graph_artifact`.
- Mark test fixtures and E2E harnesses as `test` or `fixture`.
- Mark CI folders and workflows as `ci_cd`.

If a directory contains deployable workloads but is classified as `example_demo`, preserve the deployable workload nodes, but attach them to the example/demo directory purpose. This lets the agent reason about the example without confusing it with the product's core business logic.

## First steps when invoked

1. Determine the repository root.
2. Read the repository README and architecture docs if present.
3. If `graphify-out/wiki/index.md` exists, use it as the source graph entry point.
4. Otherwise, if `graphify-out/GRAPH_REPORT.md` exists, read it before searching raw files.
5. If `graphify-out/graph.json` exists, treat it as the source graph to reference.
6. If the user asks to build or update graph artifacts and no source graph exists, run Graphify on the target path.
7. Inspect supply-chain artifacts:
   - `.github/workflows/*.yml` and `.github/workflows/*.yaml`
   - Dockerfiles
   - language manifests such as `go.mod`, `package.json`, `requirements.txt`, `pom.xml`, `Cargo.toml`
   - Cloud Native manifests and config: Kubernetes YAML, CRDs, Helm, Kustomize, Ksonnet, Helmfile, Jsonnet, ArgoCD, Flux, Skaffold, Tilt, Garden, Terraform, Pulumi, Crossplane, and CUE
   - service mesh, ingress, gateway, network policy, policy-as-code, observability, and secrets management config
   - SBOM, provenance, attestation, signature, and vulnerability scan outputs
8. Build the recursive directory classification tree.
9. Classify nodes against directory purpose, Cloud Native, CNCF, Kubernetes, and 12-factor concepts.
10. Preserve evidence and confidence on every node and edge.

Do not infer a control as present unless there is evidence. If a relationship is likely but not proven, mark it `INFERRED` and report what evidence would confirm it.

## Recommended output layout

When asked to create graph artifacts, write them under a dedicated Planeo graph output directory:

```text
planeo-graph/
├── graph.json              # Planeo supply-chain graph
├── cypher.txt              # Optional Neo4j import
├── GRAPH_REPORT.md         # Human-readable supply-chain graph report
├── gaps.md                 # Missing CI/CD/security evidence
├── classifications.json    # Cloud Native, CNCF, Kubernetes, and 12-factor classifications
├── evidence.json           # File/line/workflow/API evidence map
└── sources/
    └── graphify-graphs.json # References to source graphify-out/graph.json inputs
```

If multiple Graphify source graphs exist, use Graphify's cross-repo graph support first:

```bash
graphify merge-graphs repo1/graphify-out/graph.json repo2/graphify-out/graph.json --out graphify-out/cross-repo-graph.json
```

Then create Planeo supply-chain nodes that reference the merged source graph nodes by stable IDs.

## Node ontology

Use stable node types. Keep node IDs deterministic so repeated runs merge cleanly.

Core repository and source nodes:

- `Repository`
- `Directory`
- `DirectoryPurpose`
- `SourceGraph`
- `SourceDirectory`
- `SourceFile`
- `Function`
- `Class`
- `PackageManifest`
- `Dependency`
- `Component`
- `Microservice`
- `Application`
- `ServiceBoundary`
- `APIRoute`
- `ConfigProperty`
- `FeatureFlag`

Cloud Native classification nodes:

- `CloudNativeCategory`
- `CNCFProject`
- `CNCFDomain`
- `CNCFCapability`
- `TwelveFactorRule`
- `TwelveFactorAssessment`
- `ArchitectureStyle`
- `UserGraph`
- `UserGraphNode`
- `ClassificationCandidate`
- `ClassificationResult`
- `Capability`
- `ControlPlane`
- `DataPlane`
- `PlatformService`
- `DeveloperExperience`

CI and build nodes:

- `GitHubWorkflow`
- `WorkflowJob`
- `WorkflowStep`
- `WorkflowRun`
- `GitCommit`
- `GitTag`
- `PullRequest`
- `BuildArtifact`
- `TestReport`
- `LintReport`
- `PolicyGate`

Container and artifact nodes:

- `Dockerfile`
- `ContainerImage`
- `ImageDigest`
- `ImageTag`
- `Registry`
- `BaseImage`
- `SBOM`
- `SPDXDocument`
- `CycloneDXDocument`
- `SyftDocument`
- `Attestation`
- `SLSAProvenance`
- `Signature`
- `VulnerabilityFinding`

CD and runtime nodes:

- `HelmChart`
- `HelmValues`
- `Kustomization`
- `KustomizeOverlay`
- `KustomizeBase`
- `KsonnetApp`
- `KsonnetComponent`
- `JsonnetLibrary`
- `CUEPackage`
- `Helmfile`
- `KubernetesManifest`
- `KubernetesResource`
- `KubernetesDeployment`
- `KubernetesStatefulSet`
- `KubernetesDaemonSet`
- `KubernetesReplicaSet`
- `KubernetesJob`
- `KubernetesCronJob`
- `KubernetesService`
- `KubernetesIngress`
- `GatewayAPIResource`
- `KubernetesGateway`
- `HTTPRoute`
- `GRPCRoute`
- `TCPRoute`
- `TLSRoute`
- `KubernetesConfigMap`
- `KubernetesSecret`
- `KubernetesServiceAccount`
- `KubernetesRole`
- `KubernetesRoleBinding`
- `KubernetesClusterRole`
- `KubernetesClusterRoleBinding`
- `KubernetesNetworkPolicy`
- `KubernetesPersistentVolume`
- `KubernetesPersistentVolumeClaim`
- `KubernetesStorageClass`
- `KubernetesHorizontalPodAutoscaler`
- `KubernetesVerticalPodAutoscaler`
- `KubernetesPodDisruptionBudget`
- `KubernetesResourceQuota`
- `KubernetesLimitRange`
- `KubernetesCustomResourceDefinition`
- `KubernetesCustomResource`
- `KubernetesNamespace`
- `HelmRelease`
- `ArgoCDApplication`
- `ArgoCDApplicationSet`
- `FluxKustomization`
- `FluxHelmRelease`
- `FluxGitRepository`
- `FluxOCIRepository`
- `PlaneoStack`
- `ManualReconciler`
- `CDReconciler`
- `Environment`
- `Cluster`
- `RuntimeWorkload`
- `Pod`
- `Container`

Risk and operations nodes:

- `SecretReference`
- `Policy`
- `PolicyResult`
- `ComplianceControl`
- `ObservabilitySignal`
- `OpenTelemetrySignal`
- `PrometheusMetric`
- `GrafanaDashboard`
- `AlertRule`
- `ServiceMesh`
- `MeshPolicy`
- `IngressController`
- `APIGateway`
- `ExternalSecret`
- `SealedSecret`
- `SecretStore`
- `VaultSecret`
- `OPAConstraint`
- `KyvernoPolicy`
- `AdmissionPolicy`
- `PodSecurityPolicy`
- `PodSecurityAdmission`
- `TraceSpan`
- `LogEvent`
- `Metric`
- `Incident`
- `Gap`

## Edge ontology

Prefer explicit, directional relationships.

Source and component edges:

- `CONTAINS`
- `IMPLEMENTS`
- `IMPLEMENTED_IN`
- `IMPORTS`
- `CALLS`
- `DEPENDS_ON`
- `OWNED_BY`
- `CLASSIFIED_AS`
- `MAPS_TO_USER_GRAPH_NODE`
- `IMPLEMENTS_CAPABILITY`
- `BELONGS_TO_DOMAIN`
- `SATISFIES_12_FACTOR`
- `VIOLATES_12_FACTOR`

CI/build edges:

- `TRIGGERED_BY`
- `HAS_JOB`
- `HAS_STEP`
- `USES_ACTION`
- `CHECKS_OUT`
- `BUILDS`
- `BUILT_FROM`
- `BUILT_BY`
- `PRODUCES`
- `CONSUMES`
- `TESTS`
- `LINTS`

Artifact and security edges:

- `PUBLISHES_IMAGE`
- `TAGS_IMAGE`
- `RESOLVES_TO_DIGEST`
- `HAS_SBOM`
- `ATTESTED_BY`
- `SIGNED_BY`
- `DESCRIBES_SUBJECT`
- `HAS_VULNERABILITY`
- `SATISFIES_CONTROL`
- `VIOLATES_POLICY`

CD/runtime edges:

- `DECLARED_BY`
- `PACKAGED_IN`
- `REFERENCES_IMAGE`
- `RENDERS`
- `PATCHES`
- `OVERLAYS`
- `GENERATES`
- `CUSTOMIZES`
- `DEPLOYS`
- `DEPLOYS_TO`
- `RECONCILED_BY`
- `RUNS_IMAGE`
- `RUNS_CONTAINER`
- `CONFIGURED_BY`
- `READS_SECRET`
- `USES_SERVICE_ACCOUNT`
- `GRANTS_PERMISSION`
- `EXPOSED_BY`
- `ROUTES_TO`
- `MESHES_WITH`
- `SCALES_WITH`
- `STORES_DATA_IN`

Evidence and gap edges:

- `EVIDENCED_BY`
- `HAS_GAP`
- `MISSING_CONTROL`
- `NEEDS_EVIDENCE`
- `AFFECTS`

Avoid the phrase "implemented in Helm" for services. Source code implements services. Helm declares, packages, renders, or deploys them.

## Evidence model

Every node and edge should include:

- `id`
- `type`
- `label`
- `properties`
- `evidence`
- `confidence`
- `extractor`

Use confidence labels consistently:

- `EXTRACTED`: directly found in a file, API response, SBOM, provenance statement, or runtime object.
- `INFERRED`: reasonable deduction from naming, paths, chart values, or conventions.
- `AMBIGUOUS`: plausible but needs human review.

Evidence should cite file paths, line ranges, workflow names, artifact names, API URLs, or runtime object references.

Do not expose secret values. Store only secret keys, reference names, or masked values.

## Extractor guidance

### Graphify source graph connector

Read `graphify-out/graph.json` and create:

- one `SourceGraph` node per graph
- one `Repository` node per repo
- optional `SourceDirectory` bridge nodes
- bridge edges from `Microservice` or `Component` to relevant Graphify source nodes

When multiple repositories are involved, run or reuse:

```bash
graphify merge-graphs <g1.json> <g2.json> ... --out graphify-out/cross-repo-graph.json
```

Preserve Graphify node IDs and `repo` attributes so queries can traverse back into code.

### Cloud Native and user graph classifier

Classify every discovered entity into a Cloud Native map that the Planeo agent can use.

Use these classification dimensions:

- application vs platform vs infrastructure
- CI vs CD vs runtime vs observability vs security
- CNCF landscape category when identifiable
- Kubernetes resource category
- 12-factor rule relationship
- service ownership boundary
- environment or namespace boundary
- control plane vs data plane
- user-facing capability vs internal dependency

Create classification edges:

- `Component -> CLASSIFIED_AS -> CloudNativeCategory`
- `Component -> BELONGS_TO_DOMAIN -> CNCFDomain`
- `Component -> MAPS_TO_USER_GRAPH_NODE -> UserGraphNode`
- `KubernetesResource -> CLASSIFIED_AS -> KubernetesResourceCategory`
- `Component -> SATISFIES_12_FACTOR -> TwelveFactorRule`
- `Component -> VIOLATES_12_FACTOR -> TwelveFactorRule`

When the user's current graph already has categories, reuse those nodes instead of inventing duplicates. When no user graph exists, create a small bootstrap classification graph and mark it as inferred.

### 12-factor app assessment

Assess services against the 12-factor app principles when evidence is present:

- Codebase: one codebase tracked in revision control, many deploys
- Dependencies: explicitly declare and isolate dependencies
- Config: store config in environment, not code
- Backing services: treat backing services as attached resources
- Build, release, run: strictly separate build and run stages
- Processes: execute as stateless processes
- Port binding: export services via port binding
- Concurrency: scale out via process model
- Disposability: fast startup and graceful shutdown
- Dev/prod parity: keep environments similar
- Logs: treat logs as event streams
- Admin processes: run admin tasks as one-off processes

Do not give a pass/fail without evidence. Create `TwelveFactorAssessment` nodes with:

- rule
- status: `satisfied`, `violated`, `unknown`, or `not_applicable`
- evidence
- confidence
- remediation

Examples:

- Dockerfile `EXPOSE 8080` and Kubernetes `containerPort` can support Port Binding evidence.
- Hard-coded credentials in values files can violate Config.
- Multiple replicas and stateless deployment can support Concurrency and Processes, but only if persistent local state is not found.
- Lack of signal handling should be `unknown` unless code evidence proves it.

### CNCF and Cloud Native terms

Create normalized nodes for Cloud Native concepts so the Planeo agent can explain systems in common CNCF language.

Useful domains include:

- orchestration and scheduling
- app definition and development
- CI/CD
- database and storage
- streaming and messaging
- service proxy, discovery, and mesh
- API gateway and ingress
- observability and analysis
- monitoring
- logging
- tracing
- security and compliance
- policy
- secrets management
- container registry
- runtime
- image build
- artifact management
- chaos engineering
- cost management
- platform engineering

Create `CNCFProject` nodes when tools are found, such as Kubernetes, Helm, ArgoCD, Flux, Prometheus, Grafana, OpenTelemetry, Envoy, Istio, Linkerd, Cilium, OPA, Gatekeeper, Kyverno, Harbor, Falco, SPIFFE, SPIRE, Crossplane, Backstage, Buildpacks, Tekton, and Knative.

Do not claim a project is used just because its name appears in documentation. Mark documentation-only mentions as `MENTIONS`; mark config/runtime evidence as `USES`.

### GitHub Actions extractor

Parse `.github/workflows/*.yml` and `.github/workflows/*.yaml`.

Extract:

- workflow name
- triggers
- permissions
- concurrency
- jobs
- job dependencies
- runner type
- steps
- `uses:` actions
- `run:` commands
- artifact upload/download
- Docker build/push commands
- SBOM generation
- attestation/provenance generation
- release publication

Create gaps for:

- unpinned third-party actions
- broad permissions
- `id-token: write` without attestation usage
- build artifacts without SBOM
- build artifacts without provenance
- Dockerfiles or images with no detected build job

### Dockerfile extractor

For each Dockerfile, extract:

- component directory
- stages
- base images
- base image tags or digests
- build commands
- exposed ports
- environment variables by key only
- entrypoint/cmd
- copied artifacts

Create gaps for:

- base image not digest-pinned
- `latest` tags
- secret-like environment variables or build args
- package install without version pinning where clear

### Kubernetes, Helm, Kustomize, and manifest extractor

Support full Kubernetes application definition, not just Helm.

For each Helm chart:

- parse `Chart.yaml`
- parse dependencies
- parse `values.yaml`
- scan templates for Kubernetes resources
- extract `Deployment` containers
- connect image repository/tag values to container images
- connect service ports to workloads
- connect umbrella charts to child charts

For Kustomize:

- parse `kustomization.yaml` and `kustomization.yml`
- extract bases, resources, components, patches, images, name prefixes, name suffixes, namespaces, configMapGenerator, secretGenerator, replacements, and transformers
- create `Kustomization`, `KustomizeBase`, and `KustomizeOverlay` nodes
- connect overlays to bases with `OVERLAYS`
- connect patches with `PATCHES`
- connect generated resources with `GENERATES`

For Ksonnet and Jsonnet:

- detect `app.yaml`, `components/`, `environments/`, `lib/`, `.jsonnet`, and `.libsonnet`
- create `KsonnetApp`, `KsonnetComponent`, and `JsonnetLibrary` nodes
- connect generated Kubernetes resources with `GENERATES`
- mark Ksonnet as legacy/deprecated context when relevant, but still preserve the graph evidence

For raw Kubernetes and CRDs:

- parse every object with `apiVersion`, `kind`, `metadata.name`, and `metadata.namespace`
- create generic `KubernetesResource` nodes for unknown kinds
- create specific nodes for built-in kinds and common CNCF CRDs
- connect owner references, selectors, labels, annotations, service accounts, RBAC, storage, scaling, network policy, and ingress/gateway routes
- connect custom resources to their CRDs when both are present

Create edges:

- `HelmChart -> DEPENDS_ON -> HelmChart`
- `HelmChart -> REFERENCES_IMAGE -> ContainerImage`
- `HelmChart -> RENDERS -> KubernetesDeployment`
- `Kustomization -> OVERLAYS -> KustomizeBase`
- `Kustomization -> PATCHES -> KubernetesResource`
- `KsonnetApp -> GENERATES -> KubernetesResource`
- `KubernetesDeployment -> RUNS_IMAGE -> ContainerImage`
- `KubernetesService -> ROUTES_TO -> KubernetesDeployment`
- `KubernetesIngress/GatewayAPIResource -> EXPOSED_BY -> KubernetesService`
- `KubernetesResource -> USES_SERVICE_ACCOUNT -> KubernetesServiceAccount`
- `KubernetesRoleBinding -> GRANTS_PERMISSION -> KubernetesRole`
- `Microservice -> DECLARED_BY -> HelmChart`
- `Microservice -> PACKAGED_IN -> UmbrellaHelmChart`

Create gaps for:

- image tag `latest`
- image reference without digest
- plaintext secret-looking values
- chart with no detected CD reconciler
- deployment with no source component mapping
- resource with no owner/application classification
- workload with no probes, limits, requests, or security context
- service exposed without network policy when policy is expected
- RBAC binding with broad privileges
- custom resource with unknown controller ownership

### GitOps and CD extractor

Detect:

- ArgoCD `Application`
- ArgoCD `ApplicationSet`
- Flux `Kustomization`
- Flux `HelmRelease`
- Flux `GitRepository`
- Flux `OCIRepository`
- Flux `Bucket`
- Helmfile
- Kustomize overlays
- Ksonnet environments
- Skaffold pipelines
- Tilt dev environments
- Garden projects
- raw Kubernetes apply scripts
- Planeo stack/template definitions if present

If no reconciler exists, create a `ManualReconciler` node only with `INFERRED` confidence and report the gap.

### SBOM, provenance, and attestation extractor

Detect:

- CycloneDX JSON/XML
- SPDX JSON/tag-value
- Syft JSON
- GitHub attestations
- SLSA provenance
- cosign signatures
- checksums

Connect:

- `BuildArtifact -> HAS_SBOM -> SBOM`
- `ContainerImage/ImageDigest -> HAS_SBOM -> SBOM`
- `BuildArtifact -> ATTESTED_BY -> Attestation`
- `Attestation -> DESCRIBES_SUBJECT -> Artifact/ImageDigest`
- `SBOM -> CONTAINS -> Dependency`

If the repo has workflows that generate SBOMs or attestations but the current artifact set is unavailable, create workflow capability nodes and mark concrete artifact coverage as missing or pending evidence.

### Runtime extractor

Only use runtime extraction when the user asks for cluster-aware analysis or provides access.

Extract:

- cluster
- namespace
- workloads
- pods
- containers
- image digests
- labels/annotations
- owner references
- Helm release metadata
- ArgoCD/Flux ownership metadata

Connect runtime back to CD, images, CI, and source. If a runtime image digest cannot be traced to a build run, create a high-priority gap.

## Gap rules

Run gap detection after extraction. Report gaps with severity, evidence, and remediation.

Important gaps:

- source component has Dockerfile but no CI image build
- Helm chart references image but no producing CI workflow is found
- image uses mutable tag such as `latest`
- image is not digest-pinned in CD
- artifact has no SBOM
- artifact has no provenance/attestation
- attestation exists but does not cover all shipped subjects
- workflow uses unpinned third-party actions
- workflow has excessive permissions
- chart exists but no ArgoCD, Flux, Planeo, or other CD reconciler is detected
- runtime workload cannot be traced to source code
- runtime image digest cannot be traced to commit and workflow run
- SBOM dependency has vulnerability but no owning service is known
- secret-like values are stored in plaintext config

When answering, distinguish:

- proven missing control
- missing evidence
- ambiguous mapping
- inferred relationship

## Sample-app pattern

For a sample application with three microservices and one umbrella Helm chart, create the high-level graph:

```text
Microservice:order-service   -> PACKAGED_IN -> HelmChart:all-services
Microservice:product-service -> PACKAGED_IN -> HelmChart:all-services
Microservice:user-service    -> PACKAGED_IN -> HelmChart:all-services
```

Also create the richer trace:

```text
Microservice:order-service
  -> IMPLEMENTED_IN -> SourceDirectory:sample-app/order-service
  -> BUILT_FROM -> Dockerfile:sample-app/order-service/Dockerfile
  -> PRODUCES -> ContainerImage:order-service
  -> DECLARED_BY -> HelmChart:order-service
  -> PACKAGED_IN -> HelmChart:all-services
```

Repeat for each service. Then connect the umbrella chart to its deployment mechanism:

```text
HelmChart:all-services
  -> DEPENDS_ON -> HelmChart:order-service
  -> DEPENDS_ON -> HelmChart:product-service
  -> DEPENDS_ON -> HelmChart:user-service
  -> RECONCILED_BY -> CDReconciler:ArgoCD | Flux | Planeo | Manual
```

If the repo has GitHub Actions for CI but no image build for these microservices, report:

```text
Gap: service has Dockerfile and Helm image reference, but no CI job was found that builds and publishes the image.
```

## Answering questions from the graph

When asked a question:

1. Prefer `planeo-graph/graph.json` if it exists.
2. Use `graphify-out/GRAPH_REPORT.md` and `graphify query` for source-level details.
3. Traverse from supply-chain nodes to source graph nodes for code questions.
4. Cite file paths and line ranges when possible.
5. If a node or edge is missing, say which evidence would create it.
6. Do not make compliance claims without evidence.
7. Summarize gaps separately from confirmed relationships.

Good answer shape:

```text
Confirmed:
- service A is declared by chart B because ...
- chart B references image C because ...

Missing evidence:
- no workflow builds image C
- no SBOM found for image C

Recommended graph additions:
- add WorkflowJob node ...
- add ImageDigest node after registry lookup ...
```

## Security posture

Be conservative with secrets and credentials:

- Never print secret values from config, CI, Kubernetes, or Helm.
- Store secret references by name/key only.
- Treat plaintext secret-like strings as findings.
- Do not run destructive CI/CD or Kubernetes commands unless the user explicitly asks.
- Runtime enrichment should be read-only by default.
- Prefer local files and existing graph artifacts before external APIs.

## Quality bar

A useful Planeo graph is not just a larger graph. It must provide traceability.

Every important runtime or deployment node should eventually trace backward:

```text
Runtime workload -> image digest -> build artifact -> workflow run -> commit -> source files
```

Every important source component should eventually trace forward:

```text
source files -> component -> image/build artifact -> SBOM/provenance -> deployment definition -> runtime workload
```

If either direction breaks, create a gap node and explain what evidence is missing.
