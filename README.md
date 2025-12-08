# Stack Overview

This project uses Docker Compose to run a small web and data-processing stack consisting of a reverse proxy (Nginx), PHP/MySQL dashboard, RDF-Connect pipelines, and supporting services.

## Architecture at a Glance

All containers are connected through a shared public Docker network. Nginx is the main entry point and proxies requests to the appropriate backend services.

Main flow:

```
Browser → Nginx → (PHP | Adminer | Solid | OIDC | Pipeline)
                    ↓
                  MySQL
```

## Services

### nginx

**Role:** Reverse proxy and public entrypoint.

* Uses: nginx:alpine
* Exposes the system on ports:
  * 8001, 8002, 8003, 8004
* Loads configuration from:
  * ./nginx.conf → /etc/nginx/conf.d/default.conf (read-only)

**Responsibilities:**

* Routes incoming HTTP requests to the correct backend container based on the Nginx config.
* Waits for php, adminer, and solid to be available before starting.


### Solid

**Role:** Runs the main RDF/Linked Data pipeline (authentication-oriented).

Built using: RDF-Connect

**Runs:**
```
./pipeline/pipeline/auth-pipeline.ttl
```
Which contains
* Community Solid Server (hosting the LDES) 
* ACL generator creating authorization files
* Solid Server Identity Provider
* Solid Server Identity creator

**Exposed/Published:**
Internally: 3000

**Mounts:**
* LDES data
* Pipeline definitions

**Environment variables:**
* groupHistory: http://php/history.php?groups
* userHistory: http://php/history.php?users

**Responsibilities:**
Uses data from the php application to create authorization files for the Solid Server.
For this it uses the defined groups and users.
Create accounts for their WebId and Client Credentials.


### pipeline

**Role:** Runs the main data-processing RDF pipeline.

Built using: RDF-Connect

**Runs:**
```
./pipeline/pipeline/pipeline.ttl
```


**Environment variables:**
* groupHistory: http://php/history.php?groups
* userHistory: http://php/history.php?users
* sensorHistory: http://php/history.php?sensors
* dataHistory: http://php/history.php?data

**Responsibilities:**

Core data pipeline processing.

Fetches historical data from the php service endpoints and transforms that data into the LDES.
Including sensor LDES and sensor data LDES.


###  db (MySQL)

**Role:** Relational database for the dashboard.
**Image:** mysql:8.0
**Internal port:** 3306

Initialized using:
```
./dashboard/database_withDeviceChannelIndex.sql.sql
```

Stores user data and sensor data.

### php

**Role:** PHP-based web backend for the dashboard.

**Built from:** ./dashboard/

**Mounts:**
* Dashboard source code
* Custom settings.php

**Responsibilities:**

Serves the web dashboard.
Dashboard is used as main dashboard for viewing data and managing users.

Provides HTTP endpoints used by the pipeline containers for:
* Data history
* Sensors history
* Groups history
* Users history


### adminer

**Role:** Web-based database management UI.

**Image:** adminer

**Responsibilities:**

Allows inspecting and managing the MySQL database through the browser.


## Component interactions

All services are connected to a shared Docker network named public.
This allows containers to communicate using service names as hostnames, e.g.:

http://php/...
http://db:3306
http://solid:3000

How Components Interact

![Alt text](./Overview.drawio.svg)

Request flow


Start the Stack
```
docker compose up --build
```


Stop the stack:
```
docker compose down
```
