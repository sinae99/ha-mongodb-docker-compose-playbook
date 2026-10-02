# AGENTS.md

## Purpose

This repository is a local MongoDB replica-set test deployment managed by Docker Compose and Ansible. Future AI agents must behave as interactive infrastructure engineering assistants: inspect the repository first, gather missing requirements before editing, explain operational impact, make focused changes that follow the existing design, validate them, and clearly distinguish configuration validation from an actually executed deployment.

Do not assume this repository contains Kubernetes, Helm, Terraform, cloud provisioning, remote-host provisioning, backups, monitoring, CI, authentication, TLS, or production hardening. Those capabilities are not present in the current tree. If a user requests one of them, explain that it is not currently implemented and gather requirements before proposing additions.

## Project Overview

The current architecture is:

- Three `mongo:7` containers: `mongo1`, `mongo2`, and `mongo3`.
- One MongoDB replica set named `rs0`.
- One Docker network named `mongo-ha-net`.
- MongoDB listens on internal port `27017` in every container.
- Host ports are `27017`, `27018`, and `27019`, mapped to the corresponding members.
- Each member has its own named volume mounted at `/data/db`: `mongo1_data`, `mongo2_data`, and `mongo3_data`.
- Each member runs `mongod --replSet rs0 --bind_ip_all --port 27017 --oplogSize 2048`.
- The replica set is initialized from `mongo1` with member addresses based on Docker service names.
- Deployment waits for exactly one `PRIMARY` and two `SECONDARY` members.
- The inventory defines one logical Ansible controller, `controller`, using a local connection. This is a single-machine container topology, not a multi-host failure domain.

The Compose ports are not bound to `127.0.0.1`; the current form (`host_port:27017`) normally exposes them on all host interfaces. The Compose and Ansible configuration contains no MongoDB users, passwords, keyfile, TLS configuration, or authorization settings. Treat the current setup as an unsecured development/test environment and do not describe it as production-ready.

## Repository Structure

Important files and their responsibilities:

```text
README.md                                  Quick-start, status, and cleanup commands
ansible.cfg                                Inventory, role path, and Ansible defaults
compose.yaml                               Checked-in three-member Compose baseline
inventory/hosts.ini                        Local Ansible controller inventory
inventory/group_vars/all.yml               Replica-set, image, network, members, and ports
playbooks/deploy.yml                       Ping, deploy, initialize, and verify workflow
playbooks/clean.yml                        Stops containers and removes volumes
roles/mongo/tasks/main.yml                 Validation, rendering, startup, and replica-set checks
roles/mongo/templates/docker-compose.yml.j2
                                            Jinja template for services, volumes, and network
```

There are currently no Helm charts, `values.yaml` files, Terraform configurations, Kubernetes manifests, Dockerfiles, Ansible dependency files, test suites, CI workflows, backup jobs, or security configuration files. Inspect the tree again before relying on this statement after future changes.

## Configuration Model

The main configuration is in `inventory/group_vars/all.yml`:

- `mongo_port`: internal MongoDB port, currently `27017`.
- `mongo_replset`: replica-set name, currently `rs0`.
- `mongo_image`: container image, currently `mongo:7`.
- `mongo_network`: explicitly named Docker network, currently `mongo-ha-net`.
- `mongo_members`: ordered list of member objects, each with a Compose/container `name` and host-side `host_port`.
- `mongo_project_dir`: defaults to `{{ playbook_dir }}/..`, resolving to the repository root for the checked-in playbooks.

The Jinja template loops over `mongo_members` to generate services, volumes, and host-port mappings. However, the deployment logic is not fully generic: `roles/mongo/tasks/main.yml` currently asserts exactly three members, waits for exactly three members, and requires exactly one primary plus two secondaries. The deploy status task and initialization commands also use `mongo1` as the seed container. A request for a different member count requires coordinated changes to those assumptions; changing only `mongo_members` will fail the existing assertion or produce incorrect verification.

`roles/mongo/tasks/main.yml` renders `roles/mongo/templates/docker-compose.yml.j2` over the repository-root `compose.yaml`. The checked-in Compose file is useful for direct startup, but the template is the source used by Ansible and can overwrite manual changes to `compose.yaml`. Changes to Compose behavior should update the template and intentionally keep the checked-in baseline synchronized.

## Deployment Flow

The README documents:

```bash
docker compose up -d
ansible-playbook playbooks/deploy.yml
```

The deploy playbook and `mongo` role perform this sequence:

1. Ping the local Ansible controller.
2. Assert exactly three unique member names and exactly three unique host ports.
3. Check Docker Compose with `docker compose version` and daemon access with `docker info`.
4. Create `mongo_project_dir` and render `compose.yaml` from the Jinja template.
5. Validate the rendered stack with `docker compose config --quiet`.
6. Start all Compose services and wait up to 90 seconds for every configured host port.
7. Build an `rs.initiate(...)` expression from `mongo_members`, then query `rs.status().ok` from `mongo1`. Initialization is attempted only when the replica set is not already initialized.
8. Wait until exactly three members appear in replica-set status.
9. Wait until status reports one `PRIMARY` and two `SECONDARY` members.
10. Run a final `rs.status().members.map(...)` query from `mongo1` and print member names and states.

Replica-set initialization is persisted in the member data volumes. Restarting containers is not equivalent to creating a new replica set, and changing member names, ports, or topology while retaining old volumes may require a migration or explicit reinitialization plan.

The cleanup playbook runs locally and executes `docker compose down -v --remove-orphans`, removing containers and all named MongoDB data volumes. Its task uses `failed_when: false`, so a successful Ansible exit alone is not proof that cleanup completed; inspect Docker state afterward.

## Interactive Agent Workflow

When a user asks to deploy, modify, or customize this infrastructure:

### Discover first

- Read the README, inventory, group variables, deploy and cleanup playbooks, role tasks, and Compose template before editing.
- Trace requested values from `inventory/group_vars/all.yml` through the Jinja template and Ansible commands.
- Identify whether the request affects replica-set membership, election behavior, member names, host ports, network exposure, persistent volumes, or the seed container.
- Explain that all current members run on one machine and therefore do not protect against host failure.
- Distinguish the template-driven Compose configuration from the checked-in `compose.yaml` that Ansible may overwrite.

### Gather requirements before editing

Do not immediately change files when important requirements are missing. Ask concise questions based on the existing configuration, and do not ask for facts already established by the repository. Depending on the request, clarify:

- How many MongoDB members are required, and should the replica set use an odd number of voting members?
- Is this still local development/test, or is staging/production use intended?
- Should members stay on one host, or should the design use separate hosts and failure domains?
- What host ports, bind addresses, DNS names, Docker networks, and client endpoints are required?
- Should the image remain `mongo:7`, or should it be pinned to a specific immutable version?
- What CPU, RAM, disk, and oplog requirements apply? The current repository defines no resource limits beyond `--oplogSize 2048`.
- Are authentication, authorization, TLS, keyfile, and secret-management requirements needed? None are currently configured.
- What persistence, backup, restore, failover, and recovery objectives apply? The repository has persistent volumes but no backup or restore workflow.
- Should existing volumes and replica-set identity be preserved, or is a fresh cluster explicitly authorized?
- Is the requested operation allowed to stop containers, recreate members, change replica-set membership, or delete data?

For a different member count, clarify whether the user wants only generated Compose services or a complete working replica-set workflow. The exact-three validation and health checks must be redesigned consistently.

### Plan and obtain confirmation

After requirements are known, summarize the desired member count, topology, network exposure, and persistence model; the exact files to change; how initialization and health checks will change; possible downtime, quorum, compatibility, and data-loss risks; and security implications.

Ask for explicit confirmation before deleting volumes, rebuilding a replica set, changing member identities, changing authentication on an existing deployment, or applying changes to a production-like environment.

## Implementation Rules

- Prefer modifying the existing variables, template, role, and playbooks over introducing a separate deployment model.
- Preserve existing member volumes and replica-set identity during ordinary changes unless a fresh cluster is explicitly requested.
- Keep member names and host ports parameterized through `mongo_members`; do not duplicate environment-specific values across files.
- When changing member count, update the exact-three assertion, member-count wait, primary/secondary health condition, playbook descriptions, seed assumptions, and generated Compose services and volumes together.
- When changing the replica-set name, update the variable, rendered `mongod --replSet` arguments, and initialization expression; assess existing-volume impact first.
- When changing host ports, verify availability and decide whether they should remain exposed on all interfaces or bind only to localhost.
- Do not silently add authentication, TLS, exposed ports, data deletion, or unrelated services.
- Do not add secrets to Git, Compose files, command output, or Ansible debug output. Ask for secure secret-injection requirements instead of requesting secret values unnecessarily.
- Do not claim multi-host HA, automated backups, disaster recovery, or production readiness unless implemented and validated.

## Validation

Before declaring a configuration change complete, run the narrowest applicable checks and report the results:

```bash
docker compose config --quiet
ansible-playbook playbooks/deploy.yml --syntax-check
ansible-playbook playbooks/clean.yml --syntax-check
```

The repository's `ansible.cfg` sets `local_tmp = /tmp/ansible-local`; ensure that directory is writable. Runtime checks also require:

```bash
docker compose version
docker info
```

For actual operational verification, inspect the result instead of inferring success:

```bash
docker compose ps
docker compose exec -T mongo1 mongosh --quiet --port 27017 \
  --eval "rs.status().members.map(m => ({name:m.name, state:m.stateStr}))"
ansible-playbook playbooks/deploy.yml
```

Only claim deployment success after the deployment command has actually run and member states show the expected primary/secondary topology. If Docker is unavailable, the image cannot be pulled, the command was not authorized, or cleanup masked an error, report the result as unverified.

Check consistency after edits:

- Every `mongo_members` entry produces one service and one named volume.
- Every host port is unique, available, and bound to the intended address.
- Every service uses the same internal `mongo_port` and `mongo_replset`.
- Replica-set member addresses resolve on the configured Docker network.
- Initialization, member-count checks, and election health checks agree with the requested member count.
- The checked-in `compose.yaml` and rendered template are intentionally aligned.
- Existing volumes are not removed or silently reassigned.

There is no repository test suite, linter configuration, or CI workflow currently present. Do not claim those checks were run unless future inspection finds and executes them.

## Operations and Cleanup

Inspect the running stack without changing data:

```bash
docker compose ps
docker compose config --quiet
docker compose logs --tail=100 mongo1 mongo2 mongo3
docker compose exec -T mongo1 mongosh --quiet --port 27017 \
  --eval "rs.status().members.map(m => ({name:m.name, state:m.stateStr}))"
```

The documented cleanup command is:

```bash
# Stops containers and removes all MongoDB data volumes.
ansible-playbook playbooks/clean.yml
```

This is destructive. There is no separate volume-preserving cleanup task in the current repository. Never run `ansible-playbook playbooks/clean.yml`, `docker compose down -v`, `docker volume rm`, `kubectl delete pvc`, `terraform destroy`, or an equivalent destructive command without explicit user authorization and a clear explanation of the data-loss consequences.

## Safety Boundaries

- Treat `mongo1_data`, `mongo2_data`, and `mongo3_data` as sensitive persistent state.
- Treat the current no-auth, no-TLS, `--bind_ip_all` configuration and all-interface host port bindings as test-only.
- Do not expose this deployment to untrusted networks without a security design and explicit user approval.
- Be cautious with `mongo:7` and any future mutable image tag; ask whether the user wants an immutable version.
- Do not alter or delete existing replica-set volumes without understanding member identity, quorum, rollback, and recovery implications.
- State clearly whether you only rendered or validated configuration, or actually started, stopped, or reconfigured containers.
- If the requested behavior cannot be determined from the repository, say what must be inspected or ask the user instead of guessing.
