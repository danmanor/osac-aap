# Story Context — OSAC-2035

## Story Summary

- **Title:** Add Netris ExternalIP template role
- **Type:** Task
- **Jira:** OSAC-2035
- **Epic:** OSAC-2044 — Netris Unified Networking Template Roles
- **Feature:** OSAC-2043 — Fabric Manager - Netris

### User Story

As a fabric manager implementation, I want to allocate and release individual ExternalIPs from Netris IPAM so that tenants can get external IP addresses for inbound access (DNAT) and outbound NAT (SNAT).

### Acceptance Criteria

1. `create_external_ip.yaml` exists in `osac.templates.netris` and allocates a single IP from the Netris IPAM pool (purpose=nat)
2. `delete_external_ip.yaml` exists in `osac.templates.netris` and releases the allocated IP back to the pool
3. The allocated IP address is stored in an annotation on the ExternalIP CR for the operator to read
4. The tasks follow existing netris template role patterns (auth, API calls, debug output)
5. `argument_specs.yaml` is updated with `create_external_ip` and `delete_external_ip` entries

### Implementation Guidance

From the Jira description: "allocate/release single IP from Netris IPAM via netris.controller.ipam. Proto contract defined in OSAC-1473 (closed)."

### Testing Approach

No specific testing approach prescribed — follow project conventions. ansible-lint must pass.

### Dependencies

| Story | Status | Merged | Risk |
|-------|--------|--------|------|
| OSAC-1473 (ExternalIP proto) | Closed | Yes | None |
| OSAC-2034 (ExternalIPPool role) | In Progress | No | Low — pool role creates the IPAM subnet that this role allocates from. Tasks can proceed in parallel per epic description. |

## Design Context

### Relevant Design Sections

From the unified networking EP (`enhancements/unified-networking/README.md`):

- **ExternalIPPool** [EP: Resource Hierarchy]: ExternalIPPools are provider-managed and region-scoped. The fabric manager handles IP allocation — one pool serves all resource types.
- **ExternalIP alloc/release** [EP: Dispatcher]: The dispatcher calls only the `fabricManager` for ExternalIP operations.
- **End-to-End Flow** [EP: External Access]: `osac create externalip --pool external-pool-1 --name my-ip` → fabric manager allocates IP from IPAM.

### PRD Requirements Covered

- **R4:** ExternalIP is external to the VirtualNetwork — API is identical for all deployment topologies
- **R5:** Clear ingress/egress separation — ExternalIP allocation is independent of attachment

## Codebase Context

### Affected Components

#### osac.templates.netris (template role)
- **Location:** `osac-aap/collections/ansible_collections/osac/templates/roles/netris/`
- **Purpose:** Netris fabric manager template role — provisions networking resources via Netris Controller API
- **Current patterns:** Task files named `{action}_{resource}.yaml`. Each task: extract CR fields → auth with Netris → call Netris API (via controller roles or direct `ansible.builtin.uri`) → debug output. Uses `netris.controller.auth` for session cookie, `netris.controller.ipam` for IPAM operations, `netris.controller.vnet` for V-Nets, `netris.controller.acl` for ACLs.
- **What changes:** Add `create_external_ip.yaml` and `delete_external_ip.yaml`
- **Existing tests:** No automated tests for template roles (ansible-lint only)

#### netris.controller.ipam (controller role)
- **Location:** `osac-aap/collections/ansible_collections/netris/controller/roles/ipam/`
- **Purpose:** Low-level IPAM operations against Netris Controller API
- **Current patterns:** Tasks: `create_allocation`, `create_subnet`, `delete_allocation`, `delete_subnet`, `read` (find available IPs by purpose), `collect_available_ips`, `query_subnet_hosts`
- **What changes:** No changes — use existing tasks (`read` to find available IPs). Host creation/deletion will be done via direct API calls since no `create_host`/`delete_host` tasks exist.

#### Playbooks
- **Location:** `osac-aap/playbook_osac_*.yml`
- **Purpose:** Top-level playbooks dispatched by AAP job templates, receive EDA event payload
- **Current patterns:** Extract `implementation_strategy` from CR annotation, dynamically include role with `tasks_from`
- **What changes:** Need `playbook_osac_create_external_ip.yml` and `playbook_osac_delete_external_ip.yml`

#### Config-as-Code
- **Location:** `osac-aap/collections/ansible_collections/osac/config_as_code/roles/aap/vars/controller.yml`
- **Purpose:** Registers AAP job templates
- **What changes:** Need to add `{prefix}-create-external-ip` and `{prefix}-delete-external-ip` template entries

### Relevant Types and Interfaces

**ExternalIP CRD** (osac-operator `api/v1alpha1/externalip_types.go`):
```go
type ExternalIPSpec struct {
    Pool string `json:"pool"` // ExternalIPPool UUID, immutable
}
type ExternalIPStatus struct {
    Phase   ExternalIPPhaseType `json:"phase,omitempty"`
    Address string              `json:"address,omitempty"` // Allocated IP
    State   ExternalIPStateType `json:"state,omitempty"`
}
```

**ExternalIPPool CRD** (osac-operator `api/v1alpha1/externalippool_types.go`):
```go
type ExternalIPPoolSpec struct {
    CIDRs                  []string `json:"cidrs"`
    IPFamily               string   `json:"ipFamily"` // IPv4 or IPv6
    ImplementationStrategy string   `json:"implementationStrategy,omitempty"` // metallb-l2 or netris
}
```

**CR Annotations set by operator** (from `externalip_controller.go`):
- `osac.openshift.io/implementation-strategy` — inherited from parent pool
- `osac.openshift.io/externalippool-name` — K8s name of parent ExternalIPPool CR

**Netris IPAM API endpoints** (from controller role patterns):
- GET `/api/v2/ipam/subnets` — list IPAM subnets (filter by purpose)
- GET `/api/v2/ipam/hosts/{subnet_id}` — list allocated hosts in a subnet
- POST `/api/v2/ipam/host` — create host (allocate IP) — inferred from query pattern
- DELETE `/api/v2/ipam/host/{host_id}` — delete host — inferred

**IPAM read task interface** (`netris.controller.ipam` read.yaml):
- Input: `ipam_purpose` (e.g., "nat"), `ipam_count` (number of IPs needed), `ipam_site_id` (optional)
- Output: `ipam_allocated_ips` (list of available IP addresses)

### Relevant APIs

- Netris Controller API v2 — IPAM endpoints for IP allocation
- AAP template dispatch — `osac-create-external-ip` / `osac-delete-external-ip`

## Repository Topology

- **Origin:** osac-project/osac-aap
- **Type:** Direct
- **Fork:** danmanor/osac-aap

## Validation Profile

### Commit Format
- **Pattern:** `OSAC-XXXXX: description of change`
- **Discovered from:** git log, CLAUDE.md

### Pre-PR Checks (ordered)
1. `ansible-lint` — Lint all playbooks and roles
2. `ansible-playbook --syntax-check playbook_osac_create_external_ip.yml` — Verify playbook syntax

### PR Conventions
- **Title format:** `OSAC-2035: description`
- **PR template:** None — use default template
- **Description guidance:** Include cross-repo dependencies, ansible-lint passes, meta/osac.yaml updated

### Coverage Tooling
- **Command:** N/A — no automated test suite for template roles
- **Minimum new-code coverage:** N/A — ansible-lint is the primary quality gate

### Discovered from
- `osac-aap/CLAUDE.md`
- `osac-aap/.claude/rules/playbook-patterns.md`
- `osac-aap/.claude/rules/networking-cudn.md`
- `osac-aap/.ansible-lint.yml`
- Git log

## Open Questions

1. What is the exact Netris IPAM API endpoint for creating a host (individual IP allocation)? The existing controller role has `query_subnet_hosts` (GET `/api/v2/ipam/hosts/{id}`) but no create/delete host tasks. Is it POST `/api/v2/ipam/host` with `{address, subnetId, ...}`?

2. How should the allocated address be reported back to the operator? The MetalLB implementation stores it in a LoadBalancer Service; for Netris, writing to a CR annotation (e.g., `osac.openshift.io/allocated-address`) seems cleanest. The operator's `maybePopulateAddress` currently hardcodes MetalLB Service lookup — this will need a separate operator update.

3. Should the playbooks and config-as-code entries be part of this task or a separate task? The template role task files are the core deliverable, but without playbooks the operator can't dispatch to them.
