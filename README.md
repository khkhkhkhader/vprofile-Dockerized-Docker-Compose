
Dockerized Application

A complete containerized deployment of the VProfile Java web application using Docker, multi-stage builds, Docker Compose, and versioned images stored in Docker Hub.

The project runs the application as five coordinated containers:

- Two Apache Tomcat application containers
- One MariaDB database container
- One RabbitMQ message broker container
- One Memcached cache container

The backend services communicate through a private Docker bridge network. Only the two Tomcat application ports are published to the host.

---

## Architecture

```text
Client
  |
  |-- Host port 8081 --> Tomcat app1:8080
  |
  `-- Host port 8082 --> Tomcat app2:8080
                              |
                              | Private Docker network
                              |
                 +------------+------------+
                 |            |            |
              MariaDB      RabbitMQ     Memcached
                3306          5672         11211
```

MariaDB, RabbitMQ, and Memcached are not published to the host. They can be reached only by containers connected to the internal `backend` network.

---

## Technologies

- Docker Engine
- Docker Compose
- Docker Buildx
- Java 8
- Maven 3.9
- Apache Tomcat 9
- MariaDB 10.11
- RabbitMQ 4.1
- Memcached 1.6
- Spring MVC
- Spring Security

---

## Project Structure

```text
vprofile-docker/
├── compose.yaml
├── .dockerignore
├── .env.example
├── .gitignore
├── README.md
├── docker/
│   ├── app/
│   │   ├── Dockerfile
│   │   └── application.properties
│   ├── database/
│   │   └── Dockerfile
│   ├── rabbitmq/
│   │   └── Dockerfile
│   └── memcached/
│       └── Dockerfile
└── source/
    ├── pom.xml
    └── src/
```

---

## Container Images

The project builds and uses the following versioned images:

```text
maghraby777/vprofile-app:1.0.0
maghraby777/vprofile-db:1.0.0
maghraby777/vprofile-rabbitmq:1.0.0
maghraby777/vprofile-memcached:1.0.0
```

---

## Main Features

- Multi-stage Docker build for the Tomcat application
- Dedicated non-root user for the application runtime
- Version-pinned base images
- Runtime environment variables for credentials and service configuration
- Private Docker network for backend services
- Persistent Docker volumes for MariaDB and RabbitMQ
- Health checks for all services
- Two Tomcat application containers
- Memcached UDP disabled
- Container privilege escalation disabled
- Read-only Memcached filesystem
- Docker log rotation
- Restart policies for service recovery

---

## Prerequisites

Install the following tools:

- Git
- Docker Engine
- Docker Compose plugin
- OpenSSL

Verify the installation:

```bash
git --version
docker version
docker compose version
openssl version
```

On systems where Docker requires root permissions, prefix Docker commands with `sudo`:

```bash
sudo docker version
sudo docker compose version
```

---

## 1. Clone the Repository

```bash
git clone https://github.com/Mohamed-Maghraby/vprofile-dockerized.git
cd vprofile-dockerized
```

If the repository already contains the complete `source/` directory, continue to the next step.

If the application source is not included, clone it into `source/`:

```bash
git clone \
  --branch Master \
  --depth 1 \
  https://github.com/abdelrahmanonline4/sourcecodeseniorwr.git source
```

Remove the nested Git metadata if the source is being stored as part of this repository:

```bash
rm -rf source/.git
```

---

## 2. Configure Environment Variables

Copy the example file:

```bash
cp .env.example .env
```

Generate strong passwords:

```bash
openssl rand -hex 24
```

Run the command once for each password, or generate all values automatically:

```bash
umask 077

DB_PASSWORD="$(openssl rand -hex 24)"
DB_ROOT_PASSWORD="$(openssl rand -hex 24)"
RABBITMQ_PASSWORD="$(openssl rand -hex 24)"

cat > .env <<EOF_ENV
IMAGE_TAG=1.0.0
DB_NAME=accounts
DB_USER=vprofile
DB_PASSWORD=${DB_PASSWORD}
DB_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
RABBITMQ_USER=vprofile
RABBITMQ_PASSWORD=${RABBITMQ_PASSWORD}
EOF_ENV

unset DB_PASSWORD
unset DB_ROOT_PASSWORD
unset RABBITMQ_PASSWORD

chmod 600 .env
```

Verify the file permissions without printing the credentials:

```bash
ls -l .env
```

Expected permissions:

```text
-rw-------
```

Do not commit `.env` to Git.

---

## 3. Build the MariaDB Image

```bash
docker build \
  -f docker/database/Dockerfile \
  -t maghraby777/vprofile-db:1.0.0 \
  .
```

The database Dockerfile copies the initialization SQL file into:

```text
/docker-entrypoint-initdb.d/01-vprofile.sql
```

MariaDB executes this script only when the database volume is initialized for the first time.

Verify the image:

```bash
docker image ls maghraby777/vprofile-db
```

Verify the SQL initialization file:

```bash
docker run --rm \
  --entrypoint ls \
  maghraby777/vprofile-db:1.0.0 \
  -l /docker-entrypoint-initdb.d/
```

---

## 4. Build the RabbitMQ Image

```bash
docker build \
  -f docker/rabbitmq/Dockerfile \
  -t maghraby777/vprofile-rabbitmq:1.0.0 \
  .
```

Verify the image:

```bash
docker image ls maghraby777/vprofile-rabbitmq
```

Inspect its exposed port:

```bash
docker image inspect \
  maghraby777/vprofile-rabbitmq:1.0.0 \
  --format '{{json .Config.ExposedPorts}}'
```

Expected:

```text
{"5672/tcp":{}}
```

RabbitMQ credentials are supplied at runtime through Docker Compose. They are not stored in the image.

---

## 5. Build the Memcached Image

```bash
docker build \
  -f docker/memcached/Dockerfile \
  -t maghraby777/vprofile-memcached:1.0.0 \
  .
```

Verify the image:

```bash
docker image ls maghraby777/vprofile-memcached
```

Inspect its startup command:

```bash
docker image inspect \
  maghraby777/vprofile-memcached:1.0.0 \
  --format '{{json .Config.Cmd}}'
```

The container starts Memcached with controlled memory, thread, connection, and network settings. UDP is disabled using:

```text
-U 0
```

Confirm the image runs as a non-root user:

```bash
docker run --rm \
  --entrypoint id \
  maghraby777/vprofile-memcached:1.0.0
```

---

## 6. Build the Tomcat Application Image

```bash
docker build \
  -f docker/app/Dockerfile \
  -t maghraby777/vprofile-app:1.0.0 \
  .
```

The application image uses a multi-stage build.

### Build stage

The first stage contains:

- Maven
- Java compiler
- Application source code
- Build dependencies

Maven creates:

```text
target/vprofile-v2.war
```

### Runtime stage

The second stage contains only:

- Java runtime
- Apache Tomcat
- Compiled application WAR

The WAR is copied as:

```text
/usr/local/tomcat/webapps/ROOT.war
```

This makes the application available at `/` instead of `/vprofile-v2`.

Verify the image:

```bash
docker image ls maghraby777/vprofile-app
```

Verify the runtime user and exposed port:

```bash
docker image inspect \
  maghraby777/vprofile-app:1.0.0 \
  --format 'User={{.Config.User}} Ports={{json .Config.ExposedPorts}}'
```

Expected:

```text
User=tomcatapp Ports={"8080/tcp":{}}
```

---

## 7. Verify All Images

```bash
docker image ls --format \
  'table {{.Repository}}\t{{.Tag}}\t{{.Size}}' | \
grep maghraby777
```

Expected repositories:

```text
maghraby777/vprofile-db
maghraby777/vprofile-rabbitmq
maghraby777/vprofile-memcached
maghraby777/vprofile-app
```

---

## 8. Validate Docker Compose

```bash
docker compose config --quiet
```

No output means the Compose file is valid.

Display the services:

```bash
docker compose config --services
```

Expected:

```text
database
rabbitmq
memcached
app1
app2
```

---

## 9. Start the Complete Application

```bash
docker compose up -d
```

Docker Compose will:

1. Create the private `backend` network.
2. Create persistent MariaDB and RabbitMQ volumes.
3. Start MariaDB, RabbitMQ, and Memcached.
4. Wait for the backend services to become healthy.
5. Start both Tomcat application containers.

Check container status:

```bash
docker compose ps
```

Expected services:

```text
vprofile-database
vprofile-rabbitmq
vprofile-memcached
vprofile-app1
vprofile-app2
```

All services should eventually show `Up` and `healthy`.

---

## 10. Access the Application

Application container 1:

```text
http://localhost:8081
```

Application container 2:

```text
http://localhost:8082
```

Both root paths redirect to `/login`.

Test app1:

```bash
curl -s -o /dev/null \
  -w 'app1 status: %{http_code}\n' \
  http://127.0.0.1:8081/login
```

Test app2:

```bash
curl -s -o /dev/null \
  -w 'app2 status: %{http_code}\n' \
  http://127.0.0.1:8082/login
```

Expected:

```text
app1 status: 200
app2 status: 200
```

Do not use `curl -I` against `/login`. `curl -I` sends an HTTP `HEAD` request, while this login endpoint accepts `GET`, which may return `405 Method Not Allowed` for `HEAD`.

---

## 11. Default Demonstration Account

The database initialization script includes the following demonstration credentials:

```text
Username: admin_vp
Password: admin_vp
```

These credentials are for testing only and must be changed for a real environment.

---

## 12. View Logs

View logs for all services:

```bash
docker compose logs
```

Follow logs in real time:

```bash
docker compose logs -f
```

Application logs:

```bash
docker compose logs --tail=150 app1 app2
```

Database logs:

```bash
docker compose logs --tail=100 database
```

RabbitMQ logs:

```bash
docker compose logs --tail=100 rabbitmq
```

Memcached logs:

```bash
docker compose logs --tail=100 memcached
```

---

## 13. Verify Published Ports

```bash
docker ps --format \
  'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Only the application containers should publish host ports:

```text
vprofile-app1   0.0.0.0:8081->8080/tcp
vprofile-app2   0.0.0.0:8082->8080/tcp
```

MariaDB, RabbitMQ, and Memcached should not have host port mappings.

Verify operating-system listeners:

```bash
sudo ss -lntp
```

The backend ports should not appear as host listeners:

```text
3306
5672
11211
```

---

## 14. Verify the Database

List the database tables using the runtime credentials already available inside the container:

```bash
docker exec vprofile-database sh -c \
'mariadb \
  -u"$MARIADB_USER" \
  -p"$MARIADB_PASSWORD" \
  "$MARIADB_DATABASE" \
  -e "SHOW TABLES;"'
```

Check container health:

```bash
docker inspect \
  --format '{{json .State.Health}}' \
  vprofile-database
```

---

## 15. Push Images to Docker Hub

Authenticate using a Docker Hub personal access token with read/write permission:

```bash
docker login --username maghraby777
```

Push the database image:

```bash
docker push maghraby777/vprofile-db:1.0.0
```

Push the RabbitMQ image:

```bash
docker push maghraby777/vprofile-rabbitmq:1.0.0
```

Push the Memcached image:

```bash
docker push maghraby777/vprofile-memcached:1.0.0
```

Push the application image:

```bash
docker push maghraby777/vprofile-app:1.0.0
```

Or push all images:

```bash
for image in \
  vprofile-db \
  vprofile-rabbitmq \
  vprofile-memcached \
  vprofile-app
do
  docker push "maghraby777/${image}:1.0.0"
done
```

Log out after pushing:

```bash
docker logout
```

---

## 16. Deploy Using Prebuilt Images

A runtime host does not need Maven, source code, or Dockerfiles. It only needs:

```text
compose.yaml
.env
Docker Engine
Docker Compose
```

Pull the images:

```bash
docker compose pull
```

Start the stack:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

---

## 17. Stop or Restart the Stack

Stop and remove containers while preserving volumes:

```bash
docker compose down
```

Restart the stack:

```bash
docker compose up -d
```

Restart a specific service:

```bash
docker compose restart app1
```

Recreate only the application containers:

```bash
docker compose up -d --force-recreate app1 app2
```

---

## 18. Remove Persistent Data

To remove containers, networks, and persistent volumes:

```bash
docker compose down -v
```

> Warning: `-v` permanently deletes MariaDB and RabbitMQ data.

---

## 19. Update the Application

After changing the source code, create a new image version instead of overwriting the existing tag.

Example:

```bash
docker build \
  -f docker/app/Dockerfile \
  -t maghraby777/vprofile-app:1.1.0 \
  .
```

Push it:

```bash
docker push maghraby777/vprofile-app:1.1.0
```

Update `.env`:

```text
IMAGE_TAG=1.1.0
```

Pull and recreate the containers:

```bash
docker compose pull
docker compose up -d
```

Verify:

```bash
docker compose ps
```

---

## 20. Security Notes

The project implements the following controls:

- Secrets are supplied at runtime instead of being embedded in Docker images.
- `.env` is excluded from Git.
- Backend service ports are not published to the host.
- Application containers run as a non-root user.
- Memcached runs as a non-root user.
- Memcached UDP is disabled.
- Container privilege escalation is disabled using `no-new-privileges`.
- The Memcached container uses a read-only root filesystem.
- Base image versions are pinned.
- The application runtime image excludes Maven and source code.
- MariaDB and RabbitMQ data use persistent volumes.
- Health checks control service startup order.
- Docker logs are rotated to reduce disk exhaustion risk.

Do not commit any of the following:

```text
.env
*.pem
*.key
secrets/
source/target/
Docker credentials
Personal access tokens
Real database passwords
```

Verify that `.env` is ignored:

```bash
git check-ignore -v .env
```

---

## 21. Load Balancing Note

The application stores HTTP sessions locally inside each Tomcat container.

If both application containers are placed behind a load balancer, enable sticky sessions so a browser continues to use the same Tomcat container during authentication.

Without session stickiness, a typical failure sequence is:

```text
GET /login  --> app1 creates the session and CSRF token
POST /login --> app2 receives the request without app1's session
Result      --> HTTP 403: expected CSRF token not found
```

A more scalable alternative is storing sessions in a shared session store.

---

## 22. Troubleshooting

### Docker permission denied

Error:

```text
permission denied while trying to connect to the Docker API
```

Use:

```bash
sudo docker ...
```

or configure Docker access according to your host security requirements.

### Build context error

Run Docker builds from the project root:

```bash
cd vprofile-dockerized
```

The final dot in the build command is required:

```bash
docker build -f docker/database/Dockerfile -t image-name .
```

### MariaDB initialization did not run again

Initialization scripts run only when the MariaDB volume is empty.

For a clean test environment:

```bash
docker compose down -v
docker compose up -d
```

This deletes all stored database data.

### Application container cannot connect to MariaDB

Check logs:

```bash
docker compose logs --tail=150 app1 app2 database
```

Check service names and environment variables:

```bash
docker compose config
```

The application should use:

```text
DB_HOST=database
DB_PORT=3306
```

### RabbitMQ authentication failure

Check:

```bash
docker compose logs --tail=100 rabbitmq app1 app2
```

Ensure the same values are used for:

```text
RABBITMQ_DEFAULT_USER
RABBITMQ_DEFAULT_PASS
RABBITMQ_USER
RABBITMQ_PASSWORD
```

### Login returns HTTP 403

If a load balancer distributes requests between both application containers, enable sticky sessions.

The error may contain:

```text
Expected CSRF token not found. Has your session expired?
```

Do not disable CSRF protection to work around this issue.

### Check resource usage

```bash
docker stats
```

Check host memory:

```bash
free -h
```

Check disk space:

```bash
df -h
```

Check Docker storage usage:

```bash
docker system df
```

---

## 23. GitHub Preparation

Confirm sensitive files are ignored:

```bash
git check-ignore -v .env
```

Review files before committing:

```bash
git status
git diff --cached --stat
git diff --cached --check
```

Configure Git identity:

```bash
git config user.name "Mohamed-Maghraby"
git config user.email "mohamedali6115@gmail.com"
```

Commit:

```bash
git add .
git commit -m "Containerize VProfile services with Docker Compose"
```

Add the GitHub repository:

```bash
git remote add origin \
  https://github.com/Mohamed-Maghraby/vprofile-dockerized.git
```

Push:

```bash
git branch -M main
git push -u origin main
```

---

## License

This project is intended for learning, demonstration, and portfolio purposes. Review the original application source and dependency licenses before using it commercially.
