# MongoDB HA with Docker Compose

HA uses `mongo:7` containers in replica set `rs0` on one Docker network.

```bash
docker compose up -d
ansible-playbook playbooks/deploy.yml
```

initialize the replica set and waits for one primary and
two secondaries.


Check or clean it with:

```bash
docker compose ps
ansible-playbook playbooks/clean.yml
```

## Customize with an AI Agent

To deploy a different number of MongoDB instances or customize the
infrastructure:

1. Open this repository in Cursor, Codex, or another agent-enabled IDE.
2. Tell the agent to read `AGENTS.md` and inspect the repository first.
3. Describe the deployment you want, including the number of instances and any
   environment-specific requirements.

The agent will ask for missing requirements, explain the required changes, and
guide you through validation and deployment.
