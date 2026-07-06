## OSAC-2035: Add Netris ExternalIP template role

**Jira:** https://redhat.atlassian.net/browse/OSAC-2035
**Epic:** OSAC-2044 — Netris Unified Networking Template Roles
**Feature:** OSAC-2043 — Fabric Manager - Netris

### Summary

Adds `create_external_ip.yaml` and `delete_external_ip.yaml` to the `osac.templates.netris` role for allocating and releasing individual ExternalIP addresses from Netris IPAM. Includes matching playbooks, argument specs, and AAP job template registrations for the end-to-end operator→AAP→template dispatch chain.

### Changes

**Template role tasks:**
- `create_external_ip.yaml` — Authenticates with Netris, finds an available IP from nat-purpose IPAM subnets, creates a named /32 IPAM allocation to reserve it, writes the allocated address to a CR annotation (`osac.openshift.io/allocated-address`)
- `delete_external_ip.yaml` — Authenticates with Netris, deletes the named /32 IPAM allocation

**Role metadata:**
- `argument_specs.yaml` — Added `create_external_ip` and `delete_external_ip` entries

**Playbooks:**
- `playbook_osac_create_external_ip.yml` — Standard dispatch playbook for ExternalIP creation
- `playbook_osac_delete_external_ip.yml` — Standard dispatch playbook for ExternalIP deletion

**Config-as-Code:**
- Registered `{prefix}-create-external-ip` and `{prefix}-delete-external-ip` AAP job templates

### Testing

- **ansible-lint:** All new and modified files pass
- **Syntax check:** Both playbooks pass `ansible-playbook --syntax-check`
- **E2E:** Requires live Netris controller and AAP — out of scope for this PR

### Cross-repo dependencies

- **osac-operator:** The `ExternalIPReconciler` dispatches to `osac-create-external-ip` / `osac-delete-external-ip` AAP templates — this PR provides those templates. The operator's `getExternalIPAddress()` currently reads from MetalLB Services; a follow-up operator update is needed to read from the `osac.openshift.io/allocated-address` annotation for the Netris implementation strategy.
- **OSAC-2034 (ExternalIPPool role):** Sibling task that creates the IPAM nat-purpose subnet this role allocates from. Can proceed in parallel.

### Acceptance Criteria

- [x] `create_external_ip.yaml` exists and allocates a single IP from Netris IPAM
- [x] `delete_external_ip.yaml` exists and releases the allocated IP
- [x] Allocated address stored in CR annotation for the operator
- [x] Tasks follow existing netris template role patterns
- [x] `argument_specs.yaml` updated with ExternalIP entries
