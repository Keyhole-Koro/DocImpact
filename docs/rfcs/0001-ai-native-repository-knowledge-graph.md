# RFC 0001: AI-Native Repository Knowledge Graph

- Status: Proposed
- Authors: DocImpact contributors
- Created: 2026-08-05

## Summary

DocImpact is a repository-local knowledge graph that connects code, documentation, tests, configuration, architecture decisions, runbooks, examples, issues, and other engineering artifacts.

Its primary purpose is to answer two questions reliably:

1. When an artifact changes, what other artifacts may be affected?
2. What evidence explains each relationship?

DocImpact is designed for both humans and AI coding agents. Agents may discover and propose new relationships, but the system owns validation, persistence, lifecycle management, and auditability.

The project exposes the same core capabilities through a library, CLI, MCP server, and CI integration.

## Vision

Modern repositories contain more than source code. Their operational knowledge is spread across Markdown files, ADRs, tests, configuration, examples, schemas, generated references, issue links, and runbooks. These artifacts are related, but those relationships are usually implicit.

Humans infer them from experience. AI agents infer them repeatedly, with inconsistent results and no durable memory.

DocImpact makes those relationships explicit, queryable, reviewable, and maintainable inside the repository.

The long-term vision is:

> Every meaningful repository artifact can declare or acquire evidence-backed relationships to other artifacts, and every change can produce an explainable impact set.

DocImpact is not limited to locating documentation. It is a repository knowledge layer for software maintenance.

## Goals

DocImpact should:

- remain repository-local and work without a hosted service;
- model relationships between files, directories, globs, document sections, code symbols, URLs, commits, issues, and virtual concepts;
- allow humans and AI agents to propose, approve, reject, update, and remove relationships;
- preserve evidence, provenance, confidence, and lifecycle state for every inferred relationship;
- detect stale or broken relationships as repository contents evolve;
- calculate direct and transitive impact from working-tree, staged, commit, branch, and pull-request changes;
- expose stable interfaces through a core library, CLI, MCP server, and CI adapter;
- support multiple languages through pluggable analyzers;
- be deterministic when using explicit relationships and transparent when using inferred relationships;
- keep all writes auditable through normal version control.

## Non-goals

DocImpact is not intended to:

- replace source control;
- replace documentation authoring systems;
- require a vector database or remote indexing service;
- silently rewrite documentation without an external agent or explicit command;
- claim semantic certainty where only heuristic evidence exists;
- become a general-purpose project-management database.

## Design principles

### Repository-local by default

The graph, configuration, proposals, policies, and optional indexes live within the repository or a repository-local cache. Paths outside configured roots are rejected.

### One core, multiple adapters

Graph resolution and validation belong in a reusable core library. The CLI, MCP server, editor integrations, and CI call the same APIs.

### Evidence over assertion

An inferred relationship should include the evidence that produced it. Examples include symbol references, path mentions, matching headings, imports, links, ownership metadata, commit co-change, or agent-supplied rationale.

### Proposals before trust

AI-generated relationships are proposals unless policy explicitly permits automatic activation. Approval policy is configurable by relation type, confidence, evidence type, scope, and actor.

### Explainable impact

Every result should explain the path from the changed artifact to each impacted artifact, including relation types, confidence, and status.

### Stable identifiers

Relationships and addressable entities should survive renames where possible. File paths are locators, not necessarily permanent identity.

### Graceful degradation

DocImpact must remain useful with only explicit file and glob relationships. Symbol analysis, semantic retrieval, and history-derived inference enhance precision but are not prerequisites.

## Conceptual model

The graph consists of entities and directed relationships.

### Entity

An entity is an addressable repository artifact or concept.

Supported entity kinds should include:

- `file`
- `directory`
- `glob`
- `symbol`
- `document_section`
- `configuration_key`
- `schema_element`
- `test_case`
- `command`
- `url`
- `issue`
- `pull_request`
- `commit`
- `concept`

An entity has a stable ID and one or more locators.

```yaml
id: ent_auth_login
kind: symbol
locator:
  path: src/auth/login.ts
  symbol: login
  language: typescript
fingerprint:
  strategy: syntax-tree
  value: sha256:...
```

### Relationship

A relationship connects a source entity to a target entity.

```yaml
id: rel_auth_login_docs
source: ent_auth_login
target: ent_auth_login_flow
kind: documented_by
status: active
origin: ai
confidence: 0.94
reason: The section describes inputs, outputs, errors, and session behavior of login.
evidence:
  - kind: symbol_reference
    source_locator:
      path: docs/authentication.md
      heading: Login flow
    value: AuthService.login
created_at: 2026-08-05T00:00:00Z
created_by: agent:example
last_verified_at: 2026-08-05T00:00:00Z
```

Relationships are directional. Queries may traverse either direction.

### Relation kinds

The built-in vocabulary should include:

- `related_to`
- `documented_by`
- `explained_by`
- `implemented_by`
- `tested_by`
- `configured_by`
- `generated_from`
- `exampled_by`
- `decided_by`
- `operated_by`
- `owned_by`
- `depends_on`
- `supersedes`
- `validates`

Projects may define custom relation kinds with traversal and validation semantics.

```yaml
relation_kinds:
  secured_by:
    inverse: secures
    transitive: false
    allowed_sources: [file, symbol, command]
    allowed_targets: [file, document_section, concept]
```

### Evidence

Evidence is structured and independently verifiable where possible.

Built-in evidence kinds should include:

- `explicit_declaration`
- `path_reference`
- `symbol_reference`
- `import_reference`
- `link_reference`
- `heading_match`
- `schema_reference`
- `test_target`
- `git_cochange`
- `semantic_similarity`
- `agent_reasoning`
- `human_assertion`

Evidence can become stale independently of the relationship.

### Provenance

Every mutation records its actor and origin.

Origins include:

- `human`
- `ai`
- `analyzer`
- `import`
- `generated`

Actors may be Git identities, MCP clients, CI jobs, or agent identifiers.

### Confidence

Confidence is optional for explicit human declarations and required for inferred relationships. It is represented as a value from `0.0` to `1.0`, accompanied by the method that produced it.

Confidence is not truth. Policy determines whether a proposed relationship may become active.

### Lifecycle

Relationships have the following states:

- `proposed`: discovered but not active;
- `active`: available to normal impact queries;
- `stale`: previously active, but one or more locators or evidence items no longer validate;
- `rejected`: reviewed and declined;
- `superseded`: replaced by another relationship;
- `archived`: intentionally retained but excluded from normal queries.

Rejected proposals retain a normalized rejection fingerprint so agents do not repeatedly recreate equivalent proposals.

## Storage

The canonical state is stored as text suitable for code review.

```text
.doc-impact/
├── config.yml
├── entities.yml
├── relations.yml
├── proposals.yml
├── rejections.yml
├── policies.yml
└── schemas/
```

For large repositories, canonical data may be sharded by namespace.

```text
.doc-impact/graph/
├── docs.yml
├── packages-auth.yml
└── packages-billing.yml
```

An implementation may maintain a derived SQLite or binary index under `.git/doc-impact/` or another ignored cache directory. Derived indexes are never the source of truth.

All file writes must be atomic and deterministic. Equivalent graph states should serialize identically to minimize merge conflicts.

## Configuration

```yaml
version: 1

roots:
  - .

storage:
  canonical: .doc-impact
  cache: .git/doc-impact

registration:
  default_mode: review
  auto_activate:
    minimum_confidence: 0.98
    allowed_evidence:
      - explicit_declaration
      - symbol_reference
    relation_kinds:
      - tested_by
      - documented_by

impact:
  include_stale: false
  max_depth: 4
  minimum_confidence: 0.5

security:
  allow_external_urls: true
  allow_outside_roots: false
  max_mutations_per_request: 100
```

## Discovery and registration

### Explicit declarations

Relationships may be declared centrally, through document frontmatter, or through language-specific annotations.

```yaml
---
doc_impact:
  relates_to:
    - path: src/auth/login.ts
      symbol: login
      kind: documented_by
---
```

Declarations are parsed into the canonical graph; the source declaration is retained as evidence.

### Analyzer discovery

Language and artifact analyzers may produce candidates from:

- imports and references;
- symbols and call graphs;
- tests and covered targets;
- schemas and generated outputs;
- Markdown links and code references;
- ownership and package metadata;
- Git co-change history;
- semantic similarity.

Analyzers only produce candidate relationships and evidence. Policy controls activation.

### AI discovery

AI agents use MCP tools to inspect the graph, search candidates, and submit proposals. Agents should not edit canonical graph files directly.

A proposal must include:

- source and target locators or IDs;
- relation kind;
- rationale;
- evidence;
- confidence and method;
- optional scope and expiration.

The server normalizes locators, resolves entities, checks policy, detects duplicates, enforces repository roots, and records provenance.

### Registration modes

- `manual`: mutations require a human command;
- `review`: agents may create proposals, humans or authorized agents approve them;
- `policy`: proposals satisfying policy activate automatically;
- `auto`: authorized actors may activate valid relationships directly.

Modes can be configured globally and overridden by relation kind or path scope.

## Validation

Validation operates at multiple levels.

### Structural validation

- schema validity;
- unique IDs;
- valid relation kinds;
- valid state transitions;
- duplicate and contradictory edges;
- repository-root confinement.

### Locator validation

- files and directories exist;
- globs match or are intentionally prospective;
- document headings resolve;
- symbols resolve through an analyzer;
- issue and URL references meet policy.

### Evidence validation

- referenced text or symbol still exists;
- generated fingerprints still match;
- evidence paths are readable;
- semantic or history-based evidence records its model and parameters.

### Semantic validation

Optional analyzers or AI agents may check whether the target still describes the source. Semantic validation produces evidence and confidence, not an unconditional truth value.

### Staleness

A relationship becomes stale when its entities or required evidence no longer validate. Staleness includes a machine-readable reason.

```yaml
status: stale
stale_reason:
  code: heading_not_found
  locator:
    path: docs/authentication.md
    heading: Login flow
```

Validation may suggest locator repair after a file rename, symbol rename, or heading change.

## Impact analysis

Impact analysis starts from one or more changed entities and traverses the graph according to policy.

Sources of change include:

- working tree;
- staged changes;
- commit range;
- branch comparison;
- pull request;
- explicit paths or symbols.

The result contains:

- changed artifacts;
- directly related artifacts;
- transitively related artifacts;
- traversal paths;
- confidence aggregation;
- stale warnings;
- recommended actions.

```json
{
  "changed": [
    {"id": "ent_auth_login", "locator": {"path": "src/auth/login.ts", "symbol": "login"}}
  ],
  "impacted": [
    {
      "id": "ent_auth_login_flow",
      "locator": {"path": "docs/authentication.md", "heading": "Login flow"},
      "distance": 1,
      "confidence": 0.94,
      "path": ["rel_auth_login_docs"],
      "recommended_action": "review"
    }
  ]
}
```

Relation kinds define default impact behavior. For example, changing a generator should impact its generated output, while changing generated output may point back to the generator with a different recommended action.

## MCP interface

The MCP server is a thin adapter over the core library.

### Roots

The server uses MCP roots when available and intersects them with configured repository roots. It must reject path traversal, symlink escape, and writes outside allowed roots.

### Tools

#### `docimpact.resolve`

Resolve paths, globs, headings, symbols, or IDs into graph entities.

#### `docimpact.get_relations`

Return incoming, outgoing, or bidirectional relationships with filters for kind, status, confidence, and depth.

#### `docimpact.analyze_impact`

Calculate impact for explicit artifacts or a Git change source.

#### `docimpact.search_candidates`

Run configured analyzers to find plausible related artifacts. This tool does not mutate state.

#### `docimpact.propose_relation`

Submit one or more evidence-backed relationship proposals.

#### `docimpact.approve_relation`

Activate a proposal after authorization and validation.

#### `docimpact.reject_relation`

Reject a proposal with a reason and reusable rejection fingerprint.

#### `docimpact.upsert_relation`

Create or update an active relationship when policy permits direct mutation.

#### `docimpact.remove_relation`

Archive or delete a relationship according to retention policy.

#### `docimpact.validate`

Validate all or part of the graph and return repair suggestions.

#### `docimpact.repair`

Apply explicitly selected safe repairs, such as locator updates after a rename.

#### `docimpact.explain`

Explain why artifacts are related or impacted, including traversal and evidence.

#### `docimpact.batch`

Perform validated atomic batches of graph mutations.

### Resources

Resources provide read-only views suitable for agents:

- `docimpact://graph`
- `docimpact://entity/{id}`
- `docimpact://relations/{id}`
- `docimpact://related/{encoded-locator}`
- `docimpact://impact/{change-id}`
- `docimpact://proposals`
- `docimpact://validation`

Resource payloads should include stable structured data and an optional human-readable rendering.

### Prompts

The server may expose prompts for common workflows:

- review documentation impact for current changes;
- discover missing repository relationships;
- validate and repair stale relationships;
- explain an architecture area from connected artifacts.

Prompts are optional convenience features and must not contain core business logic.

## CLI

The CLI mirrors the core and MCP vocabulary.

```bash
doc-impact init
doc-impact resolve src/auth/login.ts#login
doc-impact related src/auth/login.ts --direction both
doc-impact impact --base main
doc-impact discover src/auth/login.ts
doc-impact propose --source ... --target ... --kind documented_by
doc-impact approve <proposal-id>
doc-impact reject <proposal-id> --reason "..."
doc-impact validate
doc-impact repair --interactive
doc-impact graph export --format json
doc-impact serve --transport stdio
```

Every mutating command supports `--dry-run`, machine-readable output, and deterministic exit codes.

## Core API

The core library should separate interfaces from adapters.

```text
core/
├── entities
├── relations
├── graph
├── policies
├── discovery
├── impact
├── validation
├── storage
└── git

adapters/
├── cli
├── mcp
├── ci
└── editor

analyzers/
├── markdown
├── git
├── treesitter
├── schemas
└── semantic
```

Key interfaces include:

- `GraphStore`
- `EntityResolver`
- `RelationRepository`
- `DiscoveryAnalyzer`
- `PolicyEngine`
- `ImpactAnalyzer`
- `Validator`
- `GitChangeProvider`

The core should not depend on MCP, a specific Git host, or an AI model.

## Transactions and concurrency

Graph mutations must be atomic. A request may carry an expected graph revision to prevent lost updates.

```json
{
  "expectedRevision": "sha256:...",
  "operations": []
}
```

On conflict, DocImpact returns the current revision and a structured conflict description. It must not silently overwrite concurrent changes.

Canonical files should use stable sorting and sharding to reduce merge conflicts. A validation command should detect unresolved semantic conflicts even when Git merges successfully.

## Security and trust

The MCP server operates on potentially sensitive repositories and accepts agent-generated writes.

Required protections include:

- root confinement and canonical-path checks;
- symlink escape prevention;
- no arbitrary shell execution in the core;
- explicit Git command allowlists;
- bounded traversal and result sizes;
- mutation rate and batch limits;
- actor authorization by operation and path scope;
- secret redaction in diagnostics;
- audit records for every mutation;
- optional read-only server mode.

External URLs and remote issue lookups are disabled unless configured. Local operation must not require telemetry.

## CI integration

CI should support both advisory and enforcing modes.

Example checks:

- graph schema is valid;
- active relationships are not stale;
- changed code has reviewed documentation impact;
- generated artifacts and their sources remain connected;
- new public symbols have documentation or an explicit exemption;
- rejected proposals are not recreated unchanged;
- canonical graph serialization is clean.

CI output should use annotations and optionally emit SARIF.

```bash
doc-impact ci --base "$BASE_SHA" --head "$HEAD_SHA" --policy .doc-impact/ci.yml
```

DocImpact should not require AI execution in CI. Deterministic checks remain available without model access.

## Agent workflow

A typical coding-agent workflow is:

1. inspect the requested change;
2. call `analyze_impact` for the working tree or target files;
3. read related artifacts;
4. update code and relevant documentation;
5. call `search_candidates` for newly discovered relationships;
6. submit evidence-backed proposals;
7. validate the graph;
8. report changed artifacts, updated relationships, unresolved proposals, and stale edges.

Agents should use graph results as context, not as unquestionable truth. Stale state, confidence, and provenance must be visible.

## Rename and refactor handling

DocImpact should recognize repository moves and semantic renames.

For files, Git rename detection and content fingerprints can update locators while preserving entity IDs.

For symbols and document sections, language analyzers and structural fingerprints may propose repairs. Ambiguous matches remain proposals requiring review.

A refactor may therefore update locators without deleting and recreating the conceptual relationship.

## Query language

The initial interfaces may use structured filters, but the long-term system should support a small query language.

```text
FROM changed(base="main")
TRAVERSE documented_by|tested_by|operated_by DEPTH 3
WHERE status = active AND confidence >= 0.7
RETURN paths, evidence, recommended_action
```

The query language must compile to the same safe core traversal APIs and must not permit arbitrary code execution.

## Extensibility

Plugins may contribute:

- entity resolvers;
- relation kinds;
- discovery analyzers;
- validators;
- storage adapters;
- import and export formats;
- UI renderers.

Plugins declare capabilities and required permissions. Untrusted plugins run out of process where practical.

## Interchange

DocImpact should support JSON export and import using a versioned schema. Exported graphs may omit private evidence and repository-specific metadata according to policy.

Potential integrations include:

- editor extensions;
- GitHub and GitLab checks;
- documentation sites;
- code search systems;
- static analysis tools;
- architecture visualization;
- agent frameworks.

The repository-local text format remains canonical even when external systems consume synchronized views.

## Observability

Commands and MCP calls should expose optional diagnostics:

- resolution duration;
- analyzer duration;
- cache hit rate;
- traversal size;
- proposal acceptance and rejection counts;
- stale-edge counts.

Diagnostics must avoid sending repository content externally unless explicitly configured.

## Versioning and migration

All canonical documents carry a schema version. Migrations are explicit, reversible when practical, and available through:

```bash
doc-impact migrate --to <version>
```

The CLI and MCP server should refuse unsafe writes against unsupported future schema versions while preserving read-only inspection where possible.

## Success criteria

DocImpact succeeds when:

- a developer can identify relevant documentation and operational artifacts from a code change without prior repository familiarity;
- an AI agent can register useful relationships through controlled APIs instead of editing an opaque mapping file;
- every inferred edge can be explained and reviewed;
- refactors do not routinely destroy the graph;
- CI can detect stale knowledge without requiring an AI model;
- the system remains useful in small repositories while scaling to monorepos.

## Open questions

- Which implementation language best serves the CLI, MCP ecosystem, and analyzer performance?
- Should canonical graph storage use YAML, JSON, JSON Lines, or a purpose-built textual format?
- Which relation kinds should be standardized versus project-defined?
- How should confidence aggregate across transitive paths?
- What evidence is sufficient for policy-based automatic activation?
- How should semantic analyzers record model identity and reproducibility metadata?
- How should graph shards minimize merge conflicts in very large monorepos?
- Which entity identifiers can remain stable across repository history without requiring a central database?

## Decision

Adopt DocImpact as an AI-native, repository-local knowledge graph with evidence-backed relationships, governed mutation workflows, explainable impact analysis, and shared core interfaces for CLI, MCP, CI, and future integrations.

Implementation may begin incrementally, but architectural decisions should preserve this ideal model rather than defining the project as only a file-to-document mapping utility.
