# Docker & DevOps Implementation — MERN Chat App

This document describes the Dockerization and DevOps implementation performed for the MERN Chat App, covering Docker fundamentals, multi-stage builds, Docker Compose, networking, persistent storage, Nginx reverse proxy, container troubleshooting, and Docker Hub image management.

## 1. Application Architecture

The application consists of three main services:

```text
Browser
   |
   | :3000
   v
Frontend / Nginx
   |
   +---------- /api/ ----------> Backend :5000
   |
   +------ /socket.io/ --------> Backend :5000
                                    |
                                    | MongoDB
                                    v
                              MongoDB :27017
                                    |
                                    v
                              mongo-data
```

### Docker Services

| Service  | Container                |    Port | Purpose                                     |
| -------- | ------------------------ | ------: | ------------------------------------------- |
| Frontend | `mern-chat-app-frontend` |  `3000` | React/Vite application served through Nginx |
| Backend  | `mern-chat-app-backend`  |  `5000` | Node.js/Express API and Socket.IO           |
| MongoDB  | `mern-chat-mongo`        | `27017` | Application database                        |

---

## 2. Dockerization

### Frontend Dockerization

Created a dedicated Dockerfile for the React/Vite frontend and used a multi-stage Docker build.

```text
Build Stage
    |
    +-- Install dependencies
    |
    +-- Build React/Vite application
    |
    v
Production Stage
    |
    +-- Nginx
    |
    +-- Serve frontend
```

### Backend Dockerization

Created a dedicated Dockerfile for the Node.js/Express backend and used a multi-stage Docker build.

```text
Build Stage
    |
    +-- Install dependencies
    |
    +-- Prepare application
    |
    v
Runtime Stage
    |
    +-- Run Node.js backend
```

---

## 3. Multi-Stage Docker Builds

Multi-stage builds were used to separate the application build environment from the production runtime environment.

### Benefits

* Keeps build dependencies out of the final image.
* Reduces unnecessary image content.
* Produces cleaner production-oriented images.
* Separates build and runtime responsibilities.

---

## 4. Image Optimization

Used lightweight Node.js Alpine-based images where applicable and installed only the dependencies required by the application runtime.

This helps reduce unnecessary image size and resource usage.

---

## 5. Docker Images

Separate Docker images were created for the frontend and backend:

```text
bhagyajeet/mern-chat-app-frontend:latest
bhagyajeet/mern-chat-app-backend:latest
```

The images were tested locally before being pushed to Docker Hub.

---

## 6. Docker Hub

The frontend and backend images were tagged and pushed to Docker Hub.

```text
Dockerfile
    |
    v
docker build
    |
    v
Docker Image
    |
    v
docker tag
    |
    v
Docker Hub
```

Docker Compose uses the published images for deployment.

---

## 7. Docker Compose

Created a `docker-compose.yml` file to manage the complete application stack.

The Compose file defines:

* Frontend service
* Backend service
* MongoDB service
* Custom Docker network
* Persistent MongoDB volume
* Environment variables
* Service dependencies
* Port mappings

The main services are:

```yaml
services:
  frontend:
  backend:
  Db:
```

---

## 8. Docker Network

Created a custom Docker network:

```yaml
networks:
  mern-chat-network:
```

All three services are connected to this network.

```text
frontend
    |
    +----------------+
    |                |
backend           MongoDB
    |                |
    +----------------+
            |
    mern-chat-network
```

This allows containers to communicate using Docker DNS/service names instead of hard-coded container IP addresses.

---

## 9. Service Dependencies

Docker Compose was configured with `depends_on`:

```yaml
frontend:
  depends_on:
    - backend

backend:
  depends_on:
    - Db
```

This defines the startup dependency relationship between the services.

---

## 10. Environment Variables

Application configuration was passed through environment variables instead of hard-coding sensitive configuration into the Compose file.

Backend configuration:

```yaml
environment:
  JWT_SECRET: ${JWT_SECRET}
  MONGO_DB_URI: ${MONGO_DB_URI}
```

MongoDB initialization configuration:

```yaml
environment:
  MONGO_INITDB_ROOT_USERNAME: ${MONGO_INITDB_ROOT_USERNAME}
  MONGO_INITDB_ROOT_PASSWORD: ${MONGO_INITDB_ROOT_PASSWORD}
  MONGO_INITDB_DATABASE: ${MONGO_INITDB_DATABASE}
```

This keeps environment-specific configuration outside the application image.

---

## 11. Persistent MongoDB Storage

A named Docker volume was configured for MongoDB:

```yaml
volumes:
  - mongo-data:/data/db
```

The volume is declared at the bottom of the Compose file:

```yaml
volumes:
  mongo-data:
```

### Purpose

```text
MongoDB Container
       |
       v
   /data/db
       |
       v
  mongo-data
```

The database data remains available when the MongoDB container is recreated.

> `docker compose down` removes the containers but normally keeps the named volume.

> `docker compose down -v` removes the named volume and therefore the stored database data.

---

## 12. Nginx Reverse Proxy

Nginx was configured as the frontend web server and reverse proxy.

It performs two main responsibilities:

1. Serves the React/Vite frontend.
2. Routes API and Socket.IO traffic to the backend container.

```text
Browser
   |
   v
 Nginx
   |
   +-- /              -> Frontend files
   |
   +-- /api/          -> backend:5000
   |
   +-- /socket.io/    -> backend:5000
```

---

## 13. SPA Routing

React is a Single Page Application (SPA), so Nginx was configured with:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

This allows Nginx to return `index.html` when a React client-side route is requested.

---

## 14. API Reverse Proxy

Nginx was configured to forward API requests to the backend:

```nginx
location /api/ {
    proxy_pass http://backend:5000/;
}
```

Request flow:

```text
Browser
   |
   | /api/...
   v
 Nginx
   |
   v
backend:5000
```

---

## 15. Socket.IO / WebSocket Proxy

The application uses Socket.IO for real-time communication.

Nginx was configured with:

```nginx
location /socket.io/ {
    proxy_pass http://backend:5000/socket.io/;

    proxy_http_version 1.1;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

The WebSocket upgrade headers allow Socket.IO connections to be handled correctly through Nginx.

---

## 16. HTTP Proxy Headers

The Nginx proxy configuration includes:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

These headers preserve useful information about the original client request when traffic is forwarded from Nginx to the backend.

---

## 17. JWT Cookie Handling

JWT authentication uses a cookie named `jwt`.

During troubleshooting, Nginx cookie handling was investigated using:

```nginx
proxy_cookie_flags jwt nosecure;
```

This was related to the application's authentication cookie behavior while accessing the application over HTTP during the Docker deployment.

---

## 18. Container Troubleshooting

Several Docker and application-level troubleshooting techniques were used during the implementation.

### Check running containers

```bash
docker ps
```

Used to verify whether the frontend, backend, and MongoDB containers were running.

### Check container logs

```bash
docker logs <container>
```

Used to identify application startup and runtime errors.

### Execute commands inside containers

```bash
docker exec <container> <command>
```

Used to inspect container configuration, Nginx configuration, environment variables, and connectivity.

### Verify Nginx configuration

```bash
docker exec mern-chat-app-frontend \
cat /etc/nginx/conf.d/default.conf
```

Used to verify the actual Nginx configuration inside the running container.

### Test application connectivity

`curl` was used to test backend APIs and external service connectivity.

---

## 19. MongoDB Connectivity Troubleshooting

Backend-to-MongoDB connectivity was tested during deployment.

The backend uses:

```text
MONGO_DB_URI
```

and connects to MongoDB through the Docker network.

MongoDB is reached using the Docker service/container name rather than a changing container IP address.

---

## 20. External Avatar Service Troubleshooting

The application uses an external avatar service.

During troubleshooting, the service was tested from the VM and the frontend container.

The request returned:

```text
HTTP/1.1 502 Bad Gateway
```

The VM also reported:

```text
No route to host
```

This helped identify that the avatar issue was related to connectivity to the external service/network path rather than the application's Nginx `/assets/` configuration.

---

## 21. Docker Compose Validation

The Compose configuration was validated using:

```bash
docker compose config
```

This was used to verify:

* YAML structure
* Environment variable substitution
* Service configuration
* Networks
* Volumes
* Port mappings

---

## 22. Docker Concepts Covered

### Docker Fundamentals

* Docker images
* Docker containers
* Dockerfiles
* Docker build
* Docker run
* Port mapping
* Environment variables
* Docker networks
* Docker volumes
* Docker Compose

### Advanced / Production Concepts

* Multi-stage Docker builds
* Image optimization
* Service-to-service communication
* Persistent database storage
* Nginx reverse proxy
* SPA routing
* API reverse proxy
* Socket.IO/WebSocket proxying
* HTTP proxy headers
* JWT cookie handling
* Container troubleshooting
* Docker Hub image management

---

## 23. Final Deployment Flow

```text
MERN Application
       |
       v
Create Dockerfiles
       |
       v
Multi-stage Docker Builds
       |
       v
Frontend + Backend Images
       |
       v
Push Images to Docker Hub
       |
       v
Docker Compose
       |
       +----------------+
       |                |
       v                v
   Frontend          Backend
   + Nginx              |
       |                |
       |                v
       |             MongoDB
       |                |
       |                v
       |           mongo-data
       |
       +-- mern-chat-network
```

## Result

The MERN Chat App was containerized as a multi-container application with:

* Separate frontend and backend Docker images
* Multi-stage Docker builds
* Docker Compose orchestration
* Custom Docker networking
* Persistent MongoDB storage
* Nginx reverse proxy
* React SPA routing
* API routing
* Socket.IO/WebSocket support
* Environment-based configuration
* Docker Hub image management
* Practical container and network troubleshooting
