Docker & DevOps Implementation — MERN Chat App

This document describes the Dockerization and DevOps implementation performed for the MERN Chat App, covering Docker fundamentals, multi-stage builds, Docker Compose, networking, persistent storage, Nginx reverse proxy, container troubleshooting, and Docker Hub image management.

1. Application Architecture

The application consists of three main services:

                    ┌──────────────────────┐
                    │       Browser        │
                    └──────────┬───────────┘
                               │
                               │ :3000
                               ▼
                    ┌──────────────────────┐
                    │   Frontend / Nginx  │
                    │     Container        │
                    └──────────┬───────────┘
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
              /api/                    /socket.io/
                 │                           │
                 ▼                           ▼
        ┌──────────────────────┐
        │ Backend / Node.js    │
        │      :5000           │
        └──────────┬───────────┘
                   │
                   │ MongoDB
                   ▼
        ┌──────────────────────┐
        │     MongoDB :27017   │
        │                      │
        │  mongo-data volume   │
        └──────────────────────┘
Docker Services
Service	Container	Port	Purpose
Frontend	mern-chat-app-frontend	3000	React/Vite application served through Nginx
Backend	mern-chat-app-backend	5000	Node.js/Express API and Socket.IO
MongoDB	mern-chat-mongo	27017	Application database
2. Dockerization
Frontend Dockerization

Created a dedicated Dockerfile for the React/Vite frontend.

The frontend uses a multi-stage Docker build:

Build Stage
    ↓
Install dependencies
    ↓
Build React/Vite application
    ↓
Production Stage
    ↓
Nginx
    ↓
Serve built frontend

This keeps build dependencies out of the final runtime image.

Backend Dockerization

Created a dedicated Dockerfile for the Node.js/Express backend.

The backend Dockerfile also uses a multi-stage build:

Build Stage
    ↓
Install dependencies
    ↓
Prepare application
    ↓
Runtime Stage
    ↓
Run Node.js backend

The backend exposes port 5000.

3. Multi-Stage Docker Builds

Multi-stage builds were used to separate the application build environment from the production runtime environment.

Benefits
Keeps build tools out of the final image.
Reduces unnecessary image content.
Produces cleaner production-oriented images.
Separates build and runtime responsibilities.
4. Image Optimization

The Dockerfiles use lightweight Node.js Alpine-based images where applicable and install only the dependencies required by the application runtime.

This helps reduce unnecessary image size and resource usage.

5. Docker Images

Separate Docker images were created for the frontend and backend.

bhagyajeet/mern-chat-app-frontend:latest
bhagyajeet/mern-chat-app-backend:latest

The images were tested locally before being pushed to Docker Hub.

6. Docker Hub

The frontend and backend images were tagged and pushed to Docker Hub.

Dockerfile
    ↓
docker build
    ↓
Docker image
    ↓
docker tag
    ↓
Docker Hub

Docker Compose then uses the published images instead of building the application locally.

7. Docker Compose

A docker-compose.yml file was created to manage the complete application stack.

The Compose file defines:

Frontend service
Backend service
MongoDB service
Custom Docker network
Persistent MongoDB volume
Environment variables
Service dependencies
Port mappings

The main services are:

services:
  frontend:
  backend:
  Db:
8. Docker Network

A custom Docker network was created:

networks:
  mern-chat-network:

All three services are connected to this network.

frontend
    │
    ├──────────────┐
    │              │
backend           MongoDB
    │              │
    └──────────────┘
          │
  mern-chat-network

This allows containers to communicate using Docker DNS/service names instead of hard-coded container IP addresses.

For example:

backend → Db
frontend → backend
9. Service Dependencies

Docker Compose was configured with depends_on:

frontend:
  depends_on:
    - backend

backend:
  depends_on:
    - Db

This defines the startup dependency relationship between the services.

10. Environment Variables

Application configuration was passed through environment variables rather than hard-coding sensitive configuration into the Compose file.

Backend configuration includes:

environment:
  JWT_SECRET: ${JWT_SECRET}
  MONGO_DB_URI: ${MONGO_DB_URI}

MongoDB initialization variables include:

environment:
  MONGO_INITDB_ROOT_USERNAME: ${MONGO_INITDB_ROOT_USERNAME}
  MONGO_INITDB_ROOT_PASSWORD: ${MONGO_INITDB_ROOT_PASSWORD}
  MONGO_INITDB_DATABASE: ${MONGO_INITDB_DATABASE}

This keeps environment-specific configuration outside the application image.

11. Persistent MongoDB Storage

A named Docker volume was configured for MongoDB:

volumes:
  - mongo-data:/data/db

The volume is declared at the bottom of the Compose file:

volumes:
  mongo-data:
Purpose
MongoDB container
       │
       ▼
   /data/db
       │
       ▼
  mongo-data

The database data remains available when the MongoDB container is recreated.

docker compose down removes the containers but normally keeps the named volume.

docker compose down -v removes the named volume and therefore the stored database data.

12. Nginx Reverse Proxy

Nginx was configured as the frontend web server and reverse proxy.

It performs two main responsibilities:

Serves the React/Vite frontend.
Routes API and Socket.IO traffic to the backend container.
Browser
   │
   ▼
Nginx
   │
   ├── /              → Frontend files
   │
   ├── /api/          → backend:5000
   │
   └── /socket.io/    → backend:5000
13. SPA Routing

React is a Single Page Application (SPA), so Nginx was configured with:

location / {
    try_files $uri $uri/ /index.html;
}

This allows Nginx to return index.html when a React client-side route is requested.

This prevents valid frontend routes from returning an Nginx 404 after a browser refresh.

14. API Reverse Proxy

Nginx was configured to forward API requests to the backend:

location /api/ {
    proxy_pass http://backend:5000/;
}

Therefore:

Browser
   │
   │ /api/...
   ▼
Nginx
   │
   ▼
backend:5000

The frontend does not need to communicate directly with the backend container's IP address.

15. Socket.IO / WebSocket Proxy

The application uses Socket.IO for real-time communication.

Nginx was configured with:

location /socket.io/ {
    proxy_pass http://backend:5000/socket.io/;

    proxy_http_version 1.1;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}

The WebSocket upgrade headers allow the connection to be upgraded and maintained for real-time communication.

16. HTTP Proxy Headers

The Nginx proxy configuration includes:

proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

These headers preserve useful information about the original client request when traffic is forwarded from Nginx to the backend.

17. JWT Cookie Handling

JWT authentication uses a cookie named jwt.

During troubleshooting, Nginx cookie handling was investigated using:

proxy_cookie_flags jwt nosecure;

This was related to the application's authentication cookie behavior while accessing the application over HTTP during the Docker deployment.

18. Container Troubleshooting

Several Docker and application-level troubleshooting techniques were used during the implementation.

Container status
docker ps

Used to verify whether the frontend, backend, and MongoDB containers were running.

Container logs
docker logs <container>

Used to identify application startup and runtime errors.

Execute commands inside containers
docker exec <container> <command>

Used to inspect container configuration, Nginx configuration, environment variables, and network connectivity.

Nginx configuration verification
docker exec mern-chat-app-frontend \
cat /etc/nginx/conf.d/default.conf

Used to verify the actual Nginx configuration inside the running container.

Application connectivity testing

curl was used to test backend APIs and external service connectivity.

19. MongoDB Connectivity Troubleshooting

Backend-to-MongoDB connectivity was tested during the deployment.

The backend uses:

MONGO_DB_URI

and connects to MongoDB through the Docker network.

The MongoDB service is reachable using its Docker service/container name rather than its changing container IP.

20. External Avatar Service Troubleshooting

The application uses an external avatar service:

avatar.iran.liara.run

During troubleshooting, the service was tested from both the VM and the frontend container.

The request returned:

502 Bad Gateway

and the VM reported:

No route to host

This helped identify that the avatar issue was related to connectivity to the external service/network path rather than the application's Nginx /assets/ configuration.

21. Docker Compose Validation

The Compose configuration was validated using:

docker compose config

This helped verify:

YAML structure
Environment variable substitution
Service configuration
Networks
Volumes
Port mappings
22. Technologies and Concepts Covered
Docker Fundamentals
Docker images
Docker containers
Dockerfiles
Docker build
Docker run
Port mapping
Environment variables
Docker networks
Docker volumes
Docker Compose
Advanced / Production Concepts
Multi-stage Docker builds
Image optimization
Service-to-service communication
Persistent database storage
Nginx reverse proxy
SPA routing
API reverse proxy
Socket.IO/WebSocket proxying
HTTP proxy headers
JWT cookie handling
Container troubleshooting
Docker Hub image management
23. Final Deployment Flow

The final workflow can be summarized as:

MERN Application
       │
       ▼
Create Dockerfiles
       │
       ▼
Multi-stage Docker Builds
       │
       ▼
Frontend + Backend Images
       │
       ▼
Push Images to Docker Hub
       │
       ▼
Docker Compose
       │
       ├───────────────┐
       ▼               ▼
   Frontend          Backend
   + Nginx              │
       │                │
       │                ▼
       │             MongoDB
       │                │
       │                ▼
       │          mongo-data
       │
       └── mern-chat-network
Result

The MERN Chat App was successfully containerized as a multi-container application with:

Separate frontend and backend Docker images
Multi-stage Docker builds
Docker Compose orchestration
Custom Docker networking
Persistent MongoDB storage
Nginx reverse proxy
React SPA routing
API routing
Socket.IO/WebSocket support
Environment-based configuration
Docker Hub image management
Practical container and network troubleshooting
