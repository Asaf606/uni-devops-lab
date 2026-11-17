# uni-devops-lab

A small DevOps project for the university student portal (Flask + Redis). The code lives in Git, the portal is packaged with Docker, and Ansible deploys it to a server automatically.

## What's inside

- **Git**: every stage was done on its own branch and merged into `main` through a Merge Request. Direct pushes to `main` are blocked. `.env`, `*.log` and key files are kept out of the repo with `.gitignore`.
- **Docker**: the image is built from `python:3.12-slim` and tagged `student-portal:1.0`. `requirements.txt` is copied and installed before the app code, so changing the code doesn't reinstall the packages. The container runs as a non-root user.
- **Compose**: the portal and Redis start together. The portal reaches Redis by its service name (`REDIS_HOST=redis`). Redis data is stored in a named volume called `redis-data` with AOF turned on (`--appendonly yes`), so the `/visits` counter survives restarts.
- **Ansible**: nothing is installed on the app server by hand. The playbook installs Docker from the official repo, enables the service, copies the portal files, starts everything with Compose and finally checks `/health`. The port, image name and folder path are all variables.

## Project layout

```
uni-devops-lab/
├── app/
│   ├── app.py
│   └── requirements.txt
├── ansible/
│   ├── ansible.cfg
│   ├── requirements.yml
│   ├── site.yml
│   ├── inventory/hosts.ini
│   ├── group_vars/app_servers.yml
│   └── roles/
│       ├── docker/
│       └── portal/
├── Dockerfile
├── .dockerignore
├── compose.yml
├── .gitignore
└── README.md
```

## Running it locally

Build the image and try a single container:

```bash
docker build -t student-portal:1.0 .
docker run -d --name portal-test -p 8080:5000 student-portal:1.0
curl http://localhost:8080/
curl http://localhost:8080/health
docker rm -f portal-test
```

Portal and Redis together:

```bash
docker compose up -d --build
curl http://localhost:8080/visits
docker compose restart
curl http://localhost:8080/visits
```

The counter also keeps going after `docker compose down` and `up`, because the volume isn't removed. Only `docker compose down -v` wipes the data.

## Deploying with Ansible

I used two servers on AWS EC2 (`eu-north-1`, Ubuntu 24.04, `t3.micro`):

| Role | Name | What's on it |
|---|---|---|
| Control node | `uni-devops-control` | Ansible (`pipx install ansible`) and git |
| App server | `uni-devops-app01` (`16.171.195.199`) | Plain Ubuntu. Ansible sets up Docker and everything else |

The security group allows `22/tcp` (SSH) and `8080/tcp` (portal).

1. Create a key on the control node and copy it to the app server, so SSH works without a password:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@16.171.195.199
ssh ubuntu@16.171.195.199 hostname
```

2. The server IP and user (`ubuntu`) are in `ansible/inventory/hosts.ini`. To deploy somewhere else, that's the only file you need to change.

3. Install the collection and check the connection:

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
ansible app_servers -m ping
```

4. Run the playbook:

```bash
ansible-playbook site.yml
```

If the server user needs a password for `sudo`, add `-K`.

Results:

| Run | PLAY RECAP | `/health` |
|---|---|---|
| First | `ok=13 changed=8 failed=0` (Docker installed, image built, services started) | `HTTP 200 {"status": "healthy"}` |
| Second | `ok=13 changed=0 failed=0` (nothing changed) | `HTTP 200 {"status": "healthy"}` |

The portal is available at `http://16.171.195.199:8080`. The `/visits` counter went 1, 2, 3, then 4 after `docker compose restart`, and 5 after `docker compose down` and `up`.

### Variables (`ansible/group_vars/app_servers.yml`)

| Variable | Value | What it's for |
|---|---|---|
| `portal_port` | `8080` | Port the portal is published on |
| `portal_image` | `student-portal:1.0` | Portal image name |
| `portal_dir` | `/opt/student-portal` | Project folder on the server |
| `portal_project_name` | `student-portal` | Compose project name |
| `portal_redis_image` | `redis:7.4-alpine` | Redis image |
| `portal_health_path` | `/health` | Endpoint used for the health check |

Ansible writes these values into a `.env` file on the server and `compose.yml` reads them from there. That `.env` only exists on the server, not in the repo.

## A problem I ran into

When I ran the playbook a second time, the Compose task still showed `changed`. The reason was `docker compose up --build`: it rebuilt the image and recreated the container on every run, which broke the "second run shouldn't change anything" requirement.

To fix it, I saved the result of the file copy tasks with `register` and tied the `build` option of `docker_compose_v2` to it. If the files changed, it uses `always`, otherwise `policy`. Now the image is only rebuilt when the code or the Dockerfile changes, and the second run ends with `changed=0`.

## Screenshots

Screenshots of the first and second playbook runs and of the running portal are submitted separately.
