This document describes the design, architecture, and development workflow of a Docker-based Node.js and MongoDB application.

# INDEX
- [1. What this application does](#what-this-application-does)
- [2. Why Docker is used](#why-docker-is-used)
- [3. What Docker Compose does](#what-docker-compose-does)
- [4. Containers in this project](#containers-in-this-project)
- [5. Docker Compose file explained](#docker-compose-file-explained)
- [6. Break down by services](#break-down-by-services)
- [7. Networking (Key Learning Point)](#networking)
- [8. How Containers Communicate](#how-containers-communicate)
- [9. Dockerfile Explanation](#dockerfile-explanation)
- [10. Startup Process](#startup-process)
- [11. Accessing the Application](#accessing-the-application)
- [12. Architecture Diagram](#architecture-diagram)
- [13. EXTRA STEP — Live updates to index.ejs](#live-updates)
- [14. Why node_modules must be preserved](#node-modules-preserved)
- [15. Tips & Troubleshooting](#tips)
- [16. Development vs Production Configuration](#dev-vs-prod-config)
- [17. Production vs Development Comparison](#prod-vs-dev-comparison)


# Docker Compose (sparta app - mongodb )

<a id="what-this-application-does"></a>
# <span style="color:red">1. What this application does</span>


This project is a **Node.js** web application that uses **MongoDB** as a database.

The app provides three main features:

1. Homepage `(/)`

- Displays a simple webpage

- Confirms the app is running correctly

2. Blog posts `(/posts)`

- Displays blog posts stored in MongoDB

- The posts are generated automatically using a seed script

- Requires a working database connection

3. Fibonacci calculator `(/fibonacci/:number)`

- Calculates a Fibonacci number

- Intentionally slow to simulate heavy CPU usage

- Does not use the database

<a id="why-docker-is-used"></a>
# <span style="color:red">2. Why Docker is used</span>


Docker is used to:

- Package the app so it runs the same everywhere

- Avoid installing Node and Mongo manually

- Isolate the database from the internet

- Make setup reproducible with one command

Instead of running everything on one machine (like EC2), we split responsibilities into containers.

<a id="what-docker-compose-does"></a>
# <span style="color:red">3. What Docker Compose does</span>


**Docker Compose** allows us to:

- Define multiple containers in one file

- Control how they connect

- Control startup order

- Run everything with one command

Command used:

```bash
docker-compose up
```
<a id="containers-in-this-project"></a>
# <span style="color:red">4. Containers in this project</span>


We use **three services (container):**

| Service | Purpose                           |
| ------- | --------------------------------- |
| `mongo` | Runs MongoDB                      |
| `seed`  | Inserts initial data into MongoDB |
| `app`   | Runs the Node.js web server       |

Each container has one responsibility only.

<a id="docker-compose-file-explained"></a>
# <span style="color:red">5. Docker Compose file explained (docker-compose.yml)</span>


```yaml
version: "3.8"

services:
  mongo:
    image: mongo:6
    container_name: sparta-mongo
    volumes:
      - mongo-data:/data/db
    networks:
      - backend

  seed:
    build: .
    container_name: sparta-seed
    command: ["node", "seeds/seed.js"]
    environment:
      DB_HOST: mongodb://mongo:27017/posts
    depends_on:
      - mongo
    networks:
      - backend
    restart: "no"

  app:
    build: .
    container_name: sparta-app
    ports:
      - "3000:3000"
    environment:
      DB_HOST: mongodb://mongo:27017/posts
    depends_on:
      - seed
    networks:
      - frontend
      - backend

volumes:
  mongo-data:

networks:
  frontend:
    driver: bridge

  backend:
    driver: bridge
    internal: true

```

<a id="break-down-by-services"></a>
# <span style="color:red">6. Break down by services</span>


### Mongo service (database)

```yaml
mongo:
  image: mongo:6
  volumes:
    - mongo-data:/data/db
  networks:
    - backend
```
What it does:

- Runs MongoDB using the official image

- Stores data in a Docker volume so data is not lost

- Only connects to the internal backend network

Why <u>no ports</u> are exposed:

- The database should not be accessible from the internet

- Only the app and seed containers can access it

### Seed service (data steup)

```yaml
seed:
  build: .
  command: ["node", "seeds/seed.js"]
  environment:
    DB_HOST: mongodb://mongo:27017/posts
  depends_on:
    - mongo
  restart: "no"
```
What it does:

- Runs the seed.js script one time

- Connects to MongoDB using DB_HOST

- Inserts example blog posts

- Stops after finishing

Why this is a separate container:

- Seeding is a one-time task

- Keeps the app startup clean

- Matches real production workflows

- Easy to re-run if needed (problem will seed with diferent text, solve in progress)

### App services (web service)

```yaml
app:
  build: .
  ports:
    - "3000:3000"
  environment:
    DB_HOST: mongodb://mongo:27017/posts
  depends_on:
    - seed
```

What it does:

- Runs the Node.js application

- Exposes port 3000 to the host

- Starts only after seeding is finished

Why ports are exposed here:

- This is the only service users should access

- Allows browser access via localhost:3000

<a id="networking"></a>
# <span style="color:red">7. Networking (Key Learning Point)</span>


Two Docker networks are used to separate public and private traffic.

### Frontend Network

- Used for browser access

- Allows localhost:3000 to reach the app

- Only the app container is connected

```yaml
frontend:
  driver: bridge
```

### Backend Network

- Private internal network

- Used for app ↔ database communication

- Blocks internet and host access

```yaml
backend:
  driver: bridge
  internal: true
```

### Why this matters

- The app must be accessible to users

- The database must remain private

- Using two networks achieves both safely

<a id="how-containers-communicate"></a>
# <span style="color:red">8. How Containers Communicate</span>


Docker provides built-in DNS.

Containers communicate using service names, not IP addresses.

The app connects to MongoDB using:

```bash
mongodb://mongo:27017/posts
```

`mongo`is automatically resolved by Docker on the backend network.

<a id="dockerfile-explanation"></a>
# <span style="color:red">9. Dockerfile Explanation</span>

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```
### Explanation

| Step              | Purpose                     |
| ----------------- | --------------------------- |
| Base image        | Lightweight Node.js runtime |
| Work directory    | Keeps paths consistent      |
| Copy dependencies | Enables caching             |
| Install packages  | Installs required libraries |
| Copy source       | Adds app code               |
| Expose port       | Documents port usage        |
| CMD               | Starts the application      |

<a id="startup-process"></a>
# <span style="color:red">10. Startup Process</span>


When `docker-compose up` is run:

1. Docker creates the networks

2. MongoDB starts

3. Seed container runs once and inserts posts

4. Seed container exits

5. App container starts

6. App listens on port 3000

<a id="accessing-the-application"></a>
# <span style="color:red">11. Accessing the Application</span>


```arduino
http://localhost:3000
```

Available routes:

- /

- /posts

- /fibonacci/10
##
### Why This Architecture Works Well

- Clear separation of responsibilities

- Database is protected by network isolation

- No manual setup after startup

- Easy to reproduce and debug

- Matches real-world Docker Compose patterns

11. Summary

- This Docker Compose setup:

- Uses multiple containers correctly

- Separates public and private networking

- Seeds data automatically

- Keeps the database secure

- Is suitable for beginners while following best practices


<a id="architecture-diagram"></a>
# <span style="color:red">12. Architecture Diagram</span>

```bash

                    ┌─────────────────────┐
                    │     Web Browser     │
                    │  http://localhost   │
                    │        :3000        │
                    └──────────┬──────────┘
                               │
                               │  (published port)
                               ▼
                     ┌─────────────────────┐
                     │     APP CONTAINER   │
                     │  Node.js + Express  │
                     │  Routes: / /posts  │
                     │  /fibonacci/:n     │
                     └───────┬────────────┘
                             │
              frontend network│backend (internal) network
                             │
                             ▼
               ┌──────────────────────────┐
               │      MONGO CONTAINER     │
               │        MongoDB           │
               │   Data stored in volume  │
               └──────────────────────────┘

               ┌──────────────────────────┐
               │      SEED CONTAINER      │
               │  Runs seeds/seed.js once │
               │  Inserts initial posts   │
               └──────────────────────────┘
```
##


<a id="live-updates"></a>
# <span style="color:red">13. EXTRA STEP — Live updates to index.ejs (NO rebuild, NO restart)</span>


### Goal

You want to:

- Edit views/index.ejs on your machine

- See the change instantly in the browser

- Without:

    - rebuilding the image

    - restarting containers

    - committing code

##

Solution: Bind Mount (Volume Mount)

Docker can **mirror** a local folder into a container.

When you edit a file locally, the container sees it immediately.

## Change to `docker-compose.yml` (APP ONLY)

Add a volume mount to the `app` service:
```yaml
app:
  build: .
  ports:
    - "3000:3000"
  environment:
    DB_HOST: mongodb://mongo:27017/posts
  depends_on:
    - seed
  networks:
    - frontend
    - backend
  volumes:
    - .:/app
```
### <span style="color: red;"> IMPORTANT:</span>

- This is for **development only**

- Not used in production

- Production containers should run from built images only

##
### Why this works

- Your local project folder (`.`) is mounted into `/app `inside the container

- Express reads `views/index.ejs` from `/app/views`

- When you save the file locally:

    - Docker syncs it instantly

    - Express renders the new version

- Refresh browser → changes appear

No Docker commands needed.

<a id="node-modules-preserved"></a>
# <span style="color:red">14. Why node_modules must be preserved</span>


Bind mounts **replace** directories inside the container.

When the entire project directory is mounted, it can overwrite the `node_modules` folder that was created during the image build.

This line prevents that issue:

```yaml
- /usr/src/app/node_modules
```

It ensures:

- Dependencies remain inside the container

- Node.js can still find modules like express

- Live code updates do not break the application
## TEST

1. Run:
```bash
docker-compose up
```
2. Opn:
```arduino
http://localhost:3000
```
3. Edit locally:
```bash
views/index.ejs
```
Example change:

```bash
<h1>Hello from Docker!</h1>
```
4. Refresh vrowser
    Change appears inmediately


Final mental model 
| Environment | Behaviour                 |
| ----------- | ------------------------- |
| Development | Bind mounts, live changes |
| Production  | Built image, no mounts    |
| Dockerfile  | Defines runtime           |
| Compose     | Defines wiring            |

<a id="tips"></a>
# <span style="color:red">15. Tips & Troubleshooting</span>

for troubleshooting
```bash
docker-compose down -v
docker-compose build --no-cache
docker-compose up
```

### Why this fixes most problems
`docker-compose down -v`

This command:

- Stops all running containers

- Removes containers

- Removes volumes (`-v`)

Why this matters:

- MongoDB data is stored in a volume

- Old data can hide problems (for example, bad seeds)

- Removing volumes guarantees a clean database

Use this when:

- Seed data looks wrong

- Posts are empty or duplicated

- You want a fresh start

`docker-compose build --no-cache`

This command:

- Rebuilds images from scratch

- Ignores Docker’s build cache

Why this matters:

- Docker normally reuses old layers

- Code changes may not be picked up

- Dependency changes can be skipped

Use this when:

- Code changes don’t seem to apply

- The app behaves like “nothing changed”

- You changed Dockerfile or dependencies

`docker-compose up`

This command:

- Starts everything again

- Uses the freshly built images

- Recreates containers and networks

Simple terms

| Problem          | Cause        | Fix                |
| ---------------- | ------------ | ------------------ |
| App not updating | Cached image | `build --no-cache` |
| Data looks wrong | Old volume   | `down -v`          |
| Weird behaviour  | Mixed state  | Full reset         |


<a id="dev-vs-prod-config"></a>
# <span style="color:red">16. Development vs Production Configuration</span>


As the project evolved, it became necessary to separate development behaviour from production behaviour.

Development and production have different goals, and Docker allows us to support both cleanly using separate configuration files.
##
### Why this separation is important

During development, we want:

- Live updates when editing files

- Fast feedback

- No need to rebuild images constantly

During production, we want:

- Stable, immutable containers

- No live file changes

- Maximum security and predictability

Mixing these concerns in one configuration can lead to:

- Accidentally deploying development features

- Security risks

- Hard-to-debug behaviour

To avoid this, the configuration is split into base (production) and development override files.
##
# File Structure Overview
```java
project/
├── Dockerfile
├── Dockerfile.dev
│
├── docker-compose.yml
├── docker-compose.dev.yml
│
├── app.js
├── views/
├── public/
├── models/
├── seeds/
└── package.json
```

##
### Base Configuration (Production)
`Dockerfile`

- Builds a complete, self-contained application image

- Copies all source code into the image

- Installs dependencies once

- Used for production and deployment

`docker-compose.yml`

- Defines the full system architecture

- Includes:

    - app

    - database

    - seed process

    - networks

    - volumes

- Contains no bind mounts

- Represents how the system should run in production

This file is safe to deploy.
##
###Development Configuration (Overrides)
`Dockerfile.dev`

- Installs dependencies only

- Does not copy source code

- Expects code to come from a bind mount

- Optimised for fast iteration

`docker-compose.dev.yml`

- Overrides only what is needed for development

- Adds bind mounts for live updates

- Preserves node_modules inside the container

- Does not affect database or networking configuration

This file is never used in production.
##
### How Docker Compose merges configurations

Docker Compose allows multiple files to be combined.

When running:

```bash
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up
```
Docker:

1. Loads the base configuration

2. Applies development overrides

3. Starts the merged setup

The base file remains unchanged and production-ready.
##

### How to run each environment
### Development (live updates enabled)
```bash
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up
```

### Production (stable configuration)
```bash
docker-compose up
```
##
<a id="prod-vs-dev-comparison"></a>
# <span style="color:red">17. Production vs Development Comparison</span>

| Aspect             | Development                                     | Production           |
| ------------------ | ----------------------------------------------- | -------------------- |
| Dockerfile         | `Dockerfile.dev`                                | `Dockerfile`         |
| Compose file       | `docker-compose.yml` + `docker-compose.dev.yml` | `docker-compose.yml` |
| Code updates       | Live via bind mounts                            | Requires rebuild     |
| Bind mounts        | Yes                                             | No                   |
| `node_modules`     | Stored in container                             | Stored in container  |
| Image immutability | No                                              | Yes                  |
| Security level     | Lower (dev only)                                | Higher               |
| Intended use       | Local development                               | Deployment           |

### Why this approach works well

This pattern:

- Prevents development features leaking into production

- Makes behaviour predictable

- Matches real-world Docker workflows

- Prepares the project for CI/CD and Kubernetes

- Makes the system easier to explain and maintain