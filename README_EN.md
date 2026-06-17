# 06 — Enterprise Stack with Docker Compose and Portainer

![Docker](https://img.shields.io/badge/Docker-Engine-2496ED?logo=docker)
![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)
![Portainer](https://img.shields.io/badge/Portainer-CE-13BEF9?logo=portainer)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-orange?logo=ubuntu)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen)

Deployment of a complete web infrastructure with Docker and Docker Compose on Ubuntu Server 22.04. The stack includes a web server with a custom image built from a Dockerfile, a MariaDB database with persistence via volumes, phpMyAdmin for database management, and Portainer as a visual Docker administration interface.

---

## Table of Contents

- [Key concepts](#key-concepts)
- [Stack architecture](#stack-architecture)
- [Docker installation](#docker-installation)
- [Project structure](#project-structure)
- [Deployment](#deployment)
- [Verification](#verification)
- [Data persistence](#data-persistence)
- [Troubleshooting](#troubleshooting)

---

## Key concepts

| Concept | Description |
|---|---|
| Image | Read-only template that defines a container's content |
| Container | A running instance of an image, isolated but sharing the host's kernel |
| Dockerfile | Set of instructions to build a custom image |
| Volume | Storage that persists outside the container's lifecycle |
| Docker network | Lets containers resolve each other by name (internal DNS) |
| Docker Compose | Orchestrates multiple containers defined in a single YAML file |

Unlike a virtual machine, a container does not include a full operating system or its own kernel: it shares the host's Linux kernel, making it much lighter (hundreds of MB vs tens of GB) and faster to start (seconds vs minutes).

---

## Stack architecture

```
SRV-DOCKER (Ubuntu Server 22.04)
Docker Engine + Docker Compose
        │
        └── red-empresa (custom Docker network)
            │
            ├── db-empresa          (mariadb:latest)
            │   └── volume: datos-empresa → /var/lib/mysql
            │
            ├── phpmyadmin-empresa  (phpmyadmin:latest)
            │   └── connects to db-empresa by name (PMA_HOST)
            │   └── port 8092:80
            │
            ├── web-empresa         (custom image: empresa-stack-web)
            │   └── built from ./web/Dockerfile
            │   └── port 8090:80
            │
            └── portainer-empresa   (portainer/portainer-ce:latest)
                └── volume: datos-portainer → /data
                └── access to /var/run/docker.sock
                └── port 9443:9443 (HTTPS)
```

---

## Docker installation

### Official repository and installation

```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Verification with a test image

![Docker pull nginx](screenshots/01-docker-pull-nginx.png)

### User permissions (avoid sudo on every command)

![Docker usermod group](screenshots/02-docker-usermod-group.png)

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## Project structure

```
empresa-stack/
├── docker-compose.yml
├── .env
└── web/
    ├── Dockerfile
    └── html/
        └── index.html
```

### Environment variables (.env)

Credentials are kept separate from `docker-compose.yml` as a good practice — `.env` is excluded from the repository:

```env
MYSQL_ROOT_PASSWORD=admin
MYSQL_DATABASE=empresa_multisede
MYSQL_USER=user
MYSQL_PASSWORD=user
```

> In a real production environment these credentials would be stronger and managed with a secrets system (Vault, AWS Secrets Manager, etc).

### Web server Dockerfile

```dockerfile
FROM nginx:alpine

LABEL maintainer="Daniel Loyo"
LABEL description="Multi-site company web server"

COPY html/ /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### docker-compose.yml

![Docker Compose YML](screenshots/03-docker-compose-yml.png)
![Compose YML content](screenshots/04-compose-yml-content.png)

```yaml
services:
  db:
    image: mariadb:latest
    container_name: db-empresa
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - datos-empresa:/var/lib/mysql
    networks:
      - red-empresa

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpmyadmin-empresa
    restart: unless-stopped
    environment:
      PMA_HOST: db-empresa
    ports:
      - "8092:80"
    networks:
      - red-empresa
    depends_on:
      - db

  web:
    build: ./web
    container_name: web-empresa
    restart: unless-stopped
    ports:
      - "8090:80"
    networks:
      - red-empresa

  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer-empresa
    restart: unless-stopped
    ports:
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - datos-portainer:/data
    networks:
      - red-empresa

volumes:
  datos-empresa:
  datos-portainer:

networks:
  red-empresa:
```

---

## Deployment

### Start the whole stack with a single command

![Docker Compose up](screenshots/05-docker-compose-up.png)

```bash
docker compose up -d
```

Docker Compose pulls the official images (`mariadb`, `phpmyadmin`, `portainer-ce`), builds the custom `empresa-stack-web` image from the Dockerfile, creates the `red-empresa` network and volumes, and starts all 4 containers connected to each other.

### Resulting images

![Docker images](screenshots/06-docker-images.png)

```
REPOSITORY                  TAG       SIZE
empresa-stack-web           latest    92.7MB
mariadb                     latest    464MB
phpmyadmin                  latest    821MB
portainer/portainer-ce      latest    242MB
```

---

## Verification

### Custom web server

![Web app](screenshots/07-web-app.png)

```
http://IP-SRV-DOCKER:8090
```

### phpMyAdmin — Login

![phpMyAdmin login](screenshots/08-phpmyadmin-login.png)

```
http://IP-SRV-DOCKER:8092
Server:   db-empresa
Username: user
```

### phpMyAdmin — Test table

![phpMyAdmin DB](screenshots/09-phpmyadmin-db.png)

```sql
CREATE TABLE prueba_docker (
    id INT AUTO_INCREMENT PRIMARY KEY,
    mensaje VARCHAR(100)
);

INSERT INTO prueba_docker (mensaje) VALUES ('Stack funcionando correctamente');
```

### Portainer — Dashboard

![Portainer dashboard](screenshots/10-portainer-dashboard.png)

```
https://IP-SRV-DOCKER:9443
```

> Portainer requires HTTPS on port 9443. The certificate is self-signed, so the browser will show a warning that needs to be accepted.

### Portainer — Containers

![Portainer containers](screenshots/11-portainer-containers.png)

| Container | State | Image | Ports |
|---|---|---|---|
| db-empresa | running | mariadb:latest | — |
| phpmyadmin-empresa | running | phpmyadmin:latest | 8092:80 |
| portainer-empresa | running | portainer/portainer-ce:latest | 9443:9443 |
| web-empresa | running | empresa-stack-web | 8090:80 |

### Portainer — Volumes

![Portainer volumes](screenshots/12-portainer-volumes.png)

### Portainer — Network and connected containers

![Portainer network](screenshots/13-portainer-network.png)

All 4 containers share the `empresa-stack_red-empresa` network and resolve each other by name thanks to Docker's internal DNS (for example, phpMyAdmin connects to `db-empresa` without needing to know its IP).

---

## Data persistence

The following test was performed to verify that data survives the container lifecycle:

```bash
# 1. Insert data into the database via phpMyAdmin
INSERT INTO prueba_docker (mensaje) VALUES ('Stack funcionando correctamente');

# 2. Tear down the whole stack
docker compose down

# 3. Verify nothing is running
docker ps -a

# 4. Bring the stack back up
docker compose up -d

# 5. Check the data is still there
SELECT * FROM empresa_multisede.prueba_docker;
```

The record `'Stack funcionando correctamente'` persists after the full `down`/`up` cycle, because it lives in the `datos-empresa` volume, not in the container itself.

---

## Troubleshooting

**Issue:** `docker build` failed with `failed to read dockerfile: open Dockerfile: no such file or directory`.
**Root cause:** The command was run from inside the `html/` subfolder instead of the project root where the Dockerfile lives.
**Fix:** Always run `docker build` from the folder containing the Dockerfile, verifying with `ls` before building.

---

**Issue:** `docker compose up -d` failed with `Conflict. The container name "/db-empresa" is already in use`.
**Root cause:** Leftover containers from earlier practice exercises were using the same names as the Compose stack.
**Fix:** Cleaned the environment before deploying the final project with `docker stop $(docker ps -aq)` and `docker rm $(docker ps -aq)`.

---

**Issue:** `Bind for 0.0.0.0:8090 failed: port is already allocated`.
**Root cause:** A previous test container was still using port 8090.
**Fix:** Identified the container with `docker ps` and removed it before starting the stack.

---

**Issue:** Portainer's web interface would not load.
**Root cause:** Accessed via `http://` instead of `https://`; Portainer requires HTTPS on port 9443.
**Fix:** Accessed with `https://IP-SRV-DOCKER:9443`, accepted the self-signed certificate warning, and restarted the container with `docker restart portainer-empresa` after an initial timeout.

---

*Lab built with Docker Engine and Docker Compose on Ubuntu Server 22.04 LTS — Daniel Moisés Loyo Vásquez*
