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
