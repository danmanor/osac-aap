# Implementation Plan — OSAC-2035

## Summary

Add `create_external_ip.yaml` and `delete_external_ip.yaml` to the `osac.templates.netris` role, plus matching playbooks, argument specs, and config-as-code entries. The create task authenticates with Netris, looks up the parent ExternalIPPool, finds an available IP from the pool's IPAM subnet (purpose=nat), allocates it as a /32 IPAM subnet to reserve it, and writes the allocated address to a CR annotation. The delete task releases the /32 IPAM subnet.

## Branch

- **Name:** feat/OSAC-2035-netris-external-ip
- **Local Base:** origin/main
- **PR Target:** main (via fork remote)

## Interface Definitions

### New Types

No new types required — this is Ansible YAML, not typed code.

### Modified Interfaces

**argument_specs.yaml** — add `create_external_ip` and `delete_external_ip` entries following the existing pattern (resource dict + optional template_parameters).

### New Functions

No new functions required.

## Test Strategy

### Unit Tests

No automated unit test framework for Ansible template roles in this project. The quality gate is `ansible-lint` and `ansible-playbook --syntax-check`.

### Integration Tests

No integration tests required — integration testing for template roles happens via E2E tests against a live Netris controller and AAP environment, which is out of scope for this task.

### Coverage Goals

All task files must pass `ansible-lint`. Playbooks must pass `--syntax-check`. The task files must follow established patterns from the existing netris role (auth, API calls, debug output, idempotency via lookup-before-create).

## Task Breakdown

### Task 1: Add create_external_ip.yaml to netris template role
- **Files:** `collections/ansible_collections/osac/templates/roles/netris/tasks/create_external_ip.yaml`
- **What:** Create the task file that:
  1. Extracts ExternalIP CR fields (name, pool name from `osac.openshift.io/externalippool-name` annotation)
  2. Looks up ExternalIPPool CR by name to get CIDRs
  3. Authenticates with Netris via `netris.controller.auth`
  4. Uses `netris.controller.ipam` read task to find 1 available IP from `nat` purpose subnets at the configured site
  5. Creates a /32 IPAM subnet named after the ExternalIP to reserve the allocated IP (using `netris.controller.ipam` create_subnet with purpose=nat)
  6. Writes the allocated address to a CR annotation (`osac.openshift.io/allocated-address`)
  7. Outputs debug information following existing patterns
- **Why:** AC-1 (create_external_ip.yaml exists and allocates from IPAM), AC-3 (address stored in annotation), AC-4 (follows existing patterns)
- **Commit message:** `OSAC-2035: add Netris ExternalIP create template role task`
- **Status:** Done

### Task 2: Add delete_external_ip.yaml to netris template role
- **Files:** `collections/ansible_collections/osac/templates/roles/netris/tasks/delete_external_ip.yaml`
- **What:** Create the task file that:
  1. Extracts ExternalIP CR name
  2. Authenticates with Netris via `netris.controller.auth`
  3. Deletes the /32 IPAM subnet by name (using `netris.controller.ipam` delete_subnet)
  4. Outputs debug information
- **Why:** AC-2 (delete_external_ip.yaml exists and releases IP), AC-4 (follows patterns)
- **Commit message:** `OSAC-2035: add Netris ExternalIP delete template role task`
- **Status:** Done

### Task 3: Update argument_specs.yaml
- **Files:** `collections/ansible_collections/osac/templates/roles/netris/meta/argument_specs.yaml`
- **What:** Add `create_external_ip` and `delete_external_ip` entries with `external_ip` dict (required) and `template_parameters` dict (optional, default {}), following the same pattern as existing entries.
- **Why:** AC-5 (argument specs updated)
- **Commit message:** `OSAC-2035: add ExternalIP argument specs to netris role`
- **Status:** Done

### Task 4: Add external_ip playbooks and config-as-code entries
- **Files:**
  - `playbook_osac_create_external_ip.yml`
  - `playbook_osac_delete_external_ip.yml`
  - `collections/ansible_collections/osac/config_as_code/roles/aap/vars/controller.yml`
- **What:**
  1. Create playbooks following the standard pattern: receive EDA payload, extract `external_ip` and `implementation_strategy`, dispatch to `osac.templates.{strategy}` with `tasks_from: create_external_ip` / `delete_external_ip`
  2. Add `{prefix}-create-external-ip` and `{prefix}-delete-external-ip` entries to the AAP config-as-code controller vars, pointing to the new playbooks
- **Why:** AC-1, AC-2 (end-to-end dispatch chain from operator → AAP → template role)
- **Commit message:** `OSAC-2035: add ExternalIP playbooks and AAP job template registration`
- **Status:** Done

## Acceptance Criteria Coverage

| AC | Description | Covered by |
|----|-------------|------------|
| AC-1 | create_external_ip.yaml allocates from Netris IPAM | Task 1, Task 4 |
| AC-2 | delete_external_ip.yaml releases IP back to pool | Task 2, Task 4 |
| AC-3 | Allocated address stored in annotation for operator | Task 1 |
| AC-4 | Tasks follow existing netris role patterns | Task 1, Task 2 |
| AC-5 | argument_specs.yaml updated | Task 3 |

## Risk Assessment

- **Netris IPAM /32 subnet as IP reservation:** Creating a /32 IPAM subnet to reserve a single IP is an inferred pattern — the existing codebase doesn't do single-IP reservation via IPAM. The `read` task finds available IPs by enumerating the subnet CIDR and subtracting hosts. A /32 IPAM subnet may or may not appear as a "host" in the parent subnet's host query. **Mitigation:** If the /32 subnet approach doesn't correctly mark the IP as unavailable in subsequent `read` calls, we can switch to direct Netris host API calls (`/api/v2/ipam/host`). The task structure remains the same either way.

- **Operator address reading:** The operator's `getExternalIPAddress` currently hardcodes MetalLB Service lookup. The annotation approach (`osac.openshift.io/allocated-address`) requires a corresponding operator update. **Mitigation:** This is a separate, well-scoped operator task. The template role is correct regardless — it allocates the IP and stores the address. The operator update can be filed as a follow-up.

- **ExternalIPPool dependency:** The sibling task OSAC-2034 (ExternalIPPool role) creates the IPAM subnet that this role allocates from. If the pool's IPAM subnet doesn't exist yet, the `read` task will return no available IPs. **Mitigation:** Tasks can proceed in parallel (per epic description). The template role will fail gracefully if no IPAM subnets with purpose=nat exist.

## Open Questions

1. **Operator address fallback:** Should the operator CR annotation update (`osac.openshift.io/allocated-address`) be done via `kubernetes.core.k8s_json_patch` or `kubernetes.core.k8s` merge patch? JSON patch is more precise but requires the annotation key to exist. Merge patch is simpler. Will use merge patch (consistent with how annotations are typically managed in Ansible).

2. **IPAM site_id resolution:** The `ipam.read` task supports `ipam_site_id` for site filtering. The ExternalIP CR doesn't carry site information — it's inherited from the pool's region. Should the site_id be extracted from the ExternalIPPool CR or from Netris group_vars? **Decision:** Use `netris_site_id` from group_vars/defaults (same as existing netris roles). The pool and the IP must be in the same Netris site.
