# Jenkins (Blue Ocean) — VPS Setup

Self-hosted Jenkins with Blue Ocean UI and Docker CLI support, deployed via Docker Compose behind Traefik on `jenkins.sanattudu.tech`.

## Stack

- **Base image:** `jenkins/jenkins:lts-jdk17`
- **Plugins:** Blue Ocean, Docker Workflow
- **Docker access:** mounted host socket (Jenkins controls the VPS's real Docker daemon directly — not an isolated DinD setup)
- **Reverse proxy:** Traefik (host network mode), automatic HTTPS via Let's Encrypt
- **Routing:** `https://jenkins.sanattudu.tech`

## Directory structure

```
~/apps/jenkins-server/
├── Dockerfile
├── docker-compose.yaml
└── README.md
```

## Prerequisites

- DNS: an A record for `jenkins.sanattudu.tech` pointing to the VPS's public IP
- Traefik already running on the VPS (handles TLS + routing)
- Docker and Docker Compose installed on the VPS

## 1. Dockerfile

```dockerfile
FROM jenkins/jenkins:lts-jdk17
USER root
RUN apt-get update && apt-get install -y lsb-release python3-pip
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow"
```

This installs the Docker CLI inside the Jenkins image (so pipeline steps can run `docker`/`docker compose` commands) and pre-installs Blue Ocean + the Docker Workflow plugin so they don't need to be added manually through the UI.

## 2. Build the image

```bash
cd ~/apps/jenkins-server
docker build -t myjenkins-blueocean:lts .
```

## 3. docker-compose.yaml

```yaml
services:
  jenkins:
    image: myjenkins-blueocean:lts
    container_name: jenkins
    user: root
    restart: unless-stopped
    ports:
      - "${JENKINS_AGENT_PORT:-50000}:50000"
    volumes:
      - jenkins_data:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
      - /root/apps:/root/apps
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jenkins.rule=Host(`jenkins.sanattudu.tech`)"
      - "traefik.http.routers.jenkins.entrypoints=websecure"
      - "traefik.http.routers.jenkins.tls=true"
      - "traefik.http.routers.jenkins.tls.certresolver=letsencrypt"
      - "traefik.http.services.jenkins.loadbalancer.server.port=8080"

volumes:
  jenkins_data:
```

**Why each part is there:**

| Line | Purpose |
|---|---|
| `user: root` | Lets Jenkins' process use the mounted Docker socket without permission errors |
| `/var/run/docker.sock` mount | Gives Jenkins direct control of the VPS's real Docker daemon, so it can rebuild/redeploy the actual running containers (portfolio, n8n, etc.) |
| `/usr/bin/docker` mount | Provides the Docker CLI binary inside the container to match the mounted socket |
| `/root/apps:/root/apps` | Lets Jenkins `cd` into the real project folders on the VPS and run `docker compose` there |
| `50000` port | Agent connection port — only needed if remote build agents are added later; safe to leave mapped even if unused |
| No explicit `networks:` block | Traefik runs in **host network mode** on this VPS, so it can already reach any container on any bridge network — no shared network is required for routing to work |
| Traefik labels | Route `jenkins.sanattudu.tech` → this container's port `8080`, over HTTPS with an auto-issued Let's Encrypt certificate |

## 4. Run it

```bash
docker compose up -d
```

## 5. Get the initial admin password

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

## 6. First-time setup

1. Visit `https://jenkins.sanattudu.tech`
2. Paste the initial admin password
3. Click **Install suggested plugins** and wait for it to finish
4. Create the first admin user (username, password, email)
5. Confirm the Jenkins URL (should default correctly to `https://jenkins.sanattudu.tech`)

## Useful commands

```bash
docker compose logs -f jenkins     # live logs
docker compose restart jenkins     # restart container
docker compose down                # stop and remove container (data volume persists)
docker volume rm jenkins-server_jenkins_data   # wipe all Jenkins data (fresh start)
```

## Notes

- Rebuilding the image (`docker build ...`) again later will pull the latest LTS release since the base tag `lts-jdk17` is a floating tag, not pinned to a specific version. Pin to an explicit tag (e.g. `jenkins/jenkins:2.541.3-lts-jdk17`) if reproducible builds matter more than always having the newest LTS.
- Jenkins has full control of the VPS's Docker daemon through the socket mount — treat Jenkins credentials/access with the same care as root access to the server.