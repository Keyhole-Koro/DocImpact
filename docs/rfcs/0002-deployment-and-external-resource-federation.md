# RFC 0002: Deployment and External Resource Federation

- Status: Proposed
- Authors: DocImpact contributors
- Created: 2026-08-05

## Summary

DocImpact should support both local and remote MCP deployments while keeping repository-managed graph data as the canonical source of truth.

The system should also treat external documentation and external MCP resources as first-class graph entities. This allows repository artifacts to relate to documents stored in systems such as Notion, Google Drive, Confluence, GitHub, issue trackers, and arbitrary MCP servers without copying those documents into the repository.

## Decision

DocImpact will use the following model:

1. Relationship metadata and repository-owned entities are canonically stored under `.doc-impact/` in the repository.
2. Local MCP servers provide access to working-tree state, staged changes, local files, and repository-local mutations.
3. Remote MCP servers provide browser-based AI clients with access to pushed repository state, Git hosting providers, external documentation systems, and federated MCP resources.
4. Databases, vector indexes, and search indexes are derived state and must be rebuildable from canonical repository data and configured external providers.
5. External document contents remain in their source systems. The repository stores stable locators, relationship metadata, evidence, revisions, and verification state.
6. MCP roots may be consumed as optional context but are never treated as an authorization boundary.

## Goals

This RFC aims to:

- support local IDE and terminal agents;
- support browser-based AI clients through a remote MCP endpoint;
- preserve offline and self-hosted operation;
- keep relationship changes reviewable through normal Git workflows;
- connect repository artifacts to external documents without duplicating them;
- allow DocImpact to federate resources exposed by other MCP servers;
- make authentication, authorization, and trust boundaries explicit;
- avoid coupling the core graph model to one Git host or documentation provider.

## Non-goals

This RFC does not require:

- a mandatory hosted DocImpact service;
- direct remote access to an uncommitted local working tree;
- copying complete external documents into the repository;
- storing provider credentials in repository files;
- treating MCP roots as a sandbox;
- making external providers part of the canonical graph store.

## Source-of-truth model

### Canonical repository state

The canonical DocImpact state is committed to the repository.

```text
.doc-impact/
├── config.yml
├── entities.yml
├── relations.yml
├── proposals.yml
├── rejections.yml
├── policies.yml
├── providers.yml
└── schemas/
```

This state contains:

- repository-local entities;
- external entity locators;
- relationships;
- evidence and provenance;
- confidence and lifecycle state;
- provider references that do not contain secrets;
- optional external revision identifiers and content hashes;
- validation and synchronization policy.

Changes to active relationships should normally be committed alongside the code or documentation changes that motivated them.

### Derived state

The following are derived and must not be authoritative:

- SQLite indexes;
- graph databases;
- vector stores;
- full-text search indexes;
- cached external document metadata;
- cached external document contents;
- remote service projections;
- generated visualizations.

Derived state may live locally, in CI, or in a hosted service. It must be possible to invalidate and rebuild it.

### External systems remain authoritative for their content

A Notion page, Google Drive document, Confluence page, issue, or MCP resource remains authoritative in its source system.

DocImpact stores the relationship to that resource, not a competing canonical copy of its content.

```yaml
id: ent_external_auth_design
kind: external_document
locator:
  scheme: mcp
  provider: company-notion
  uri: notion://page/abc123
revision:
  value: "2026-08-04T12:03:22Z"
  content_hash: sha256:...
```

Cached content must be marked as derived and governed by provider policy.

## Deployment model

DocImpact exposes one core implementation through multiple deployment adapters.

```text
                   Browser AI clients
                           |
                    HTTPS MCP transport
                           |
                 Remote DocImpact MCP
                  /        |         \
             Git host   External   Derived
              adapter     MCPs      indexes
                  |
                pushed repository state

Local AI / IDE
      |
  stdio MCP
      |
Local DocImpact MCP and CLI
      |
working tree, staged changes, local repository
```

### Local MCP server

The local server is the primary interface for:

- working-tree and staged changes;
- files not yet pushed to a Git host;
- repository-local discovery and validation;
- local filesystem analyzers;
- editing proposal and graph files;
- interactive approval and repair;
- editor integrations.

The expected transport is `stdio`.

```bash
doc-impact serve --transport stdio
```

A local HTTP mode may be supported for specialized clients, but it must bind to loopback by default and enforce origin, authentication, and repository-boundary checks.

### Remote MCP server

The remote server is the primary interface for:

- browser-based AI clients;
- pushed branches, commits, and pull requests;
- GitHub, GitLab, and other Git hosting providers;
- external documentation providers;
- external MCP resource federation;
- organization-wide indexing and search;
- cross-repository impact views where authorized.

A remote endpoint may use Streamable HTTP over HTTPS.

```text
https://mcp.example.com/docimpact
```

The remote server does not implicitly gain access to a user's local filesystem or uncommitted working tree.

### Shared core

Both deployments call the same core interfaces:

- `GraphStore`;
- `EntityResolver`;
- `RelationRepository`;
- `ProviderRegistry`;
- `ExternalResourceResolver`;
- `ImpactAnalyzer`;
- `PolicyEngine`;
- `Validator`.

Transport and hosting concerns must not be embedded in graph logic.

## Local and remote consistency

The repository is the synchronization boundary between local and remote deployments.

A typical flow is:

1. a local agent changes code;
2. the local MCP server analyzes working-tree impact;
3. the agent proposes or updates relationships under `.doc-impact/`;
4. the relationship changes are reviewed in the same branch;
5. the branch is pushed;
6. the remote service indexes the new commit;
7. browser clients observe the pushed state.

The remote service may create a branch or pull request containing graph proposals, but it must not maintain a hidden canonical graph that diverges from the repository.

## External entity model

The graph model must not assume that every entity is a local file.

Supported locator schemes should include:

- `repo`;
- `git`;
- `github`;
- `gitlab`;
- `url`;
- `mcp`;
- provider-specific schemes such as `notion`, `gdrive`, and `confluence`.

Examples:

```yaml
- id: ent_login_symbol
  kind: symbol
  locator:
    scheme: repo
    path: src/auth/login.ts
    symbol: login

- id: ent_auth_doc
  kind: external_document
  locator:
    scheme: notion
    provider: company-notion
    page_id: abc123

- id: ent_auth_architecture_resource
  kind: external_resource
  locator:
    scheme: mcp
    provider: architecture-server
    uri: architecture://systems/authentication
```

Locators should be structured rather than stored only as opaque URLs. Implementations may retain a normalized URI representation for interchange.

## External MCP federation

A DocImpact MCP server may also act as an MCP client to other servers.

```text
AI client
   |
DocImpact MCP gateway
   |-- repository provider
   |-- GitHub provider
   |-- Notion MCP server
   |-- Google Drive MCP server
   |-- Confluence MCP server
   `-- custom MCP server
```

The federation layer should:

- enumerate configured providers;
- resolve external resource URIs;
- retrieve metadata and content when authorized;
- normalize external resources into DocImpact entities;
- preserve the originating provider and URI;
- record revision and content-hash information where available;
- expose provider failures without corrupting canonical graph state;
- prevent credential or data leakage between providers.

DocImpact should not flatten every external tool into its own native API when a provider already exposes a suitable MCP resource interface.

## Provider configuration

Repository configuration declares logical providers without storing credentials.

```yaml
providers:
  company-notion:
    type: mcp
    expected_resource_schemes:
      - notion
    trust: organization

  architecture-server:
    type: mcp
    expected_resource_schemes:
      - architecture
    trust: project
```

Runtime configuration resolves logical provider IDs to actual endpoints and credentials.

```text
Repository configuration:
  provider identity, allowed schemes, policy

Runtime secret store:
  endpoint, OAuth tokens, API keys, refresh tokens
```

Provider aliases allow the same repository to work in local, CI, and hosted environments without committing environment-specific secrets.

## Relationship example

```yaml
id: rel_login_documented_by_external_design
source: ent_login_symbol
target: ent_auth_doc
kind: documented_by
status: active
origin: ai
confidence: 0.93
reason: The external design document specifies the login flow and failure behavior.
evidence:
  - kind: symbol_reference
    provider: company-notion
    resource_uri: notion://page/abc123
    value: AuthService.login
  - kind: semantic_similarity
    provider: company-notion
    resource_uri: notion://page/abc123
    content_hash: sha256:...
last_verified_revision: "2026-08-04T12:03:22Z"
```

When the provider reports a new revision or content hash, DocImpact may mark the evidence as requiring revalidation without immediately deleting the relationship.

## MCP roots

MCP roots are optional client-provided context describing filesystem locations relevant to a session.

DocImpact may use roots to discover or prioritize repositories, but roots are not an authorization mechanism and must not be required for correct operation.

The server must independently enforce:

- canonical repository boundaries;
- path normalization;
- traversal rejection;
- symlink escape prevention;
- allowed repository and provider scopes;
- operation-specific authorization.

The effective local scope is determined from explicit server configuration, repository discovery, and authorization policy. Any MCP roots are intersected with that scope rather than replacing it.

Remote deployments normally operate on repository identities, Git refs, and provider resource identifiers instead of local filesystem roots.

## Authorization model

Capabilities should be separated by resource type and operation.

```text
repo:read
repo:propose
repo:write
repo:approve
external:read
external:propose
external:write
provider:configure
index:read
index:rebuild
```

A normal coding agent should usually receive:

```text
repo:read
repo:propose
external:read
```

Activating relationships, writing external documents, or changing provider configuration should require stronger authorization or explicit approval.

Authorization must be enforced by the DocImpact server even when the upstream AI client presents a narrower interface.

## Authentication

Remote deployments should use an established authorization flow appropriate for HTTP MCP clients and provider access.

Authentication should identify:

- the human or service principal;
- the MCP client;
- the organization or tenant;
- allowed repositories;
- allowed external providers;
- permitted mutation classes.

Provider credentials should be delegated or stored in a managed secret store. They must not be returned to AI clients or written into graph evidence.

## External writes

External document writes are distinct from graph writes.

DocImpact may expose a proposal workflow for external edits, but should not silently change external documents during impact analysis.

A safe flow is:

1. detect that an external document may be stale;
2. produce a structured update proposal;
3. require explicit authorization;
4. invoke the external provider's write operation;
5. record the resulting revision and audit event;
6. revalidate the relationship.

External providers may be configured as read-only.

## Availability and degradation

The canonical graph remains readable when external providers are unavailable.

Queries should distinguish:

- relationship exists and target is verified;
- relationship exists but provider is unavailable;
- relationship exists but target revision changed;
- relationship locator no longer resolves;
- relationship is stale for semantic reasons.

Provider outages must not cause destructive graph rewrites.

## Caching and privacy

External content caching is optional and policy-controlled.

Policies may define:

- whether content may be cached;
- maximum retention;
- encryption requirements;
- allowed cache locations;
- whether embeddings may be generated;
- whether content may leave the provider's region;
- redaction and deletion behavior.

A repository may store external content hashes without storing the content itself.

## Cross-repository relationships

Remote deployments may support cross-repository relationships where authorized.

The canonical ownership of a relationship must be explicit. Recommended options are:

- store the relationship in the repository whose artifact is the source;
- mirror a read-only inverse view in the target repository;
- store organization-wide relationships in a dedicated graph repository.

A remote database may index all three cases but is not canonical.

## MCP interface additions

The MCP adapter should expose provider-aware operations.

### `docimpact.list_providers`

List configured providers and available capabilities without exposing secrets.

### `docimpact.resolve_external`

Resolve an external locator or MCP resource URI into a normalized entity.

### `docimpact.fetch_external`

Retrieve authorized external metadata or content with revision information.

### `docimpact.verify_external`

Check whether an external entity and its evidence still resolve at the recorded revision.

### `docimpact.search_external_candidates`

Search configured external providers for artifacts related to one or more entities. This tool does not activate relationships.

### `docimpact.propose_external_update`

Create a reviewable proposal to update an external document.

### `docimpact.sync_index`

Refresh derived provider and repository indexes without changing canonical graph state.

## Browser AI workflow

A browser-based agent may:

1. authenticate to the remote DocImpact MCP server;
2. select an authorized repository and Git ref;
3. analyze impact for a commit or pull request;
4. fetch related repository and external resources;
5. propose relationship additions or documentation updates;
6. create a branch or pull request containing canonical graph changes;
7. leave external edits as separately authorized proposals.

It cannot inspect uncommitted local state unless a separately authorized local bridge is introduced.

## Optional local bridge

A future local bridge may allow a remote session to delegate operations to a user's machine.

```text
Browser AI
   |
Remote DocImpact MCP
   |
authenticated, user-approved channel
   |
Local DocImpact agent
   |
working tree
```

This is an optional extension and requires:

- explicit per-session user approval;
- end-to-end authentication;
- operation allowlists;
- repository-bound scopes;
- visible local audit logs;
- immediate revocation;
- no unsolicited inbound filesystem access.

The main architecture must not depend on this bridge.

## Package architecture

```text
packages/
├── core/
├── graph-store/
├── provider-api/
├── providers/
│   ├── repository/
│   ├── github/
│   ├── generic-mcp/
│   ├── notion/
│   ├── google-drive/
│   └── confluence/
├── mcp-local/
├── mcp-remote/
├── cli/
└── indexer/
```

Provider-specific packages implement shared interfaces and may delegate to third-party MCP servers.

## Security requirements

Implementations must:

- treat repository files as untrusted input;
- prevent path and symlink escape;
- prevent server-side request forgery through external locators;
- allowlist provider endpoints and resource schemes;
- isolate tenant and repository indexes;
- redact credentials and sensitive content from logs;
- audit graph and external mutations;
- enforce capability checks on every operation;
- validate external content before passing it to downstream agents;
- defend against prompt injection in external documents by preserving source labels and trust metadata.

External content is evidence, not executable instruction.

## Open questions

- Should remote graph proposals be committed directly to branches or submitted through a dedicated proposal service before Git materialization?
- Which provider metadata is safe and useful to commit by default?
- How should external revision identifiers be normalized across providers?
- Should cross-repository relationships live in source repositories or dedicated graph repositories?
- What is the minimum generic MCP client capability required for useful federation?
- How should provider trust levels influence agent context and automatic activation policy?
- Which external caching modes should be standardized?

## Consequences

This design keeps DocImpact useful as a local tool while allowing it to become a browser-accessible knowledge gateway.

It also expands DocImpact from a repository-only mapping system into a federated engineering knowledge graph without surrendering version-controlled relationship ownership.

The cost is additional complexity in provider identity, authorization, caching, and synchronization. That complexity is isolated behind adapters and does not change the canonical graph principles defined in RFC 0001.
