# DevOps TP – Docker, GitHub Actions & Ansible

This repository contains my work for the three DevOps labs (Takima / EFREI):

1. **Docker** – containerize a 3-tier application (database, backend API, HTTP reverse proxy)
2. **GitHub Actions** – CI pipeline (build, tests, SonarCloud quality gate) and CD to Docker Hub
3. **Ansible** – automatic deployment of the whole application (+ front-end) on a remote server

The application is deployed on: **http://mohamed.sakho.takima.school**

## Project structure

```
tp/
├── Database/               # PostgreSQL image + init scripts
│   ├── Dockerfile
│   ├── 01-CreateScheme.sql
│   └── 02-InsertData.sql
├── BackendAPI/             # Spring Boot API (Java 21, Maven) + multistage Dockerfile
├── HttpServer/             # Apache httpd used as reverse proxy
│   ├── Dockerfile
│   ├── httpd.conf
│   └── public-html/index.html
├── front/                  # Vue.js front-end, built with Node and served by Nginx
├── docker-compose.yml      # Local orchestration of the containers
├── .github/workflows/
│   └── main.yml            # CI/CD pipeline
└── my-project/ansible/     # Ansible inventory, playbook and roles
    ├── inventories/setup.yml
    ├── playbook.yml
    └── roles/{docker,network,database,app,front,proxy}
```

One folder per image, as asked in the subject.

---

# Part 1 – Docker

## Database

The database image is based on `postgres:17.2-alpine`. The SQL scripts are copied into `/docker-entrypoint-initdb.d/`. When the container starts for the first time, Postgres runs every script in that folder in alphabetical order. That's why they are prefixed `01-` (schema) and `02-` (data).

```dockerfile
FROM postgres:17.2-alpine

COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d/
COPY 02-InsertData.sql /docker-entrypoint-initdb.d/
```

The credentials are **not** in the Dockerfile. They are given at runtime with `-e`.

### 1-1 Why use `-e` for environment variables rather than putting them in the Dockerfile?

If the password is written in the Dockerfile, it's stored in the image (and visible with `docker history` or `docker inspect`) and in the Git repository. Anyone who pulls the image or reads the repo gets the credentials. With `-e` the secrets stay outside the image, and the same image can be reused in different environments (dev, prod…) with different values.

### 1-2 Why do we need a volume attached to the Postgres container?

A container is ephemeral: when it's removed, everything written inside it is lost, including the database files. Mounting a volume on `/var/lib/postgresql/data` stores the data on the host disk. So the data survives when the container is destroyed and recreated. I tested it: I removed the container and started a new one with the same volume, and the data was still there.

### 1-3 Database container essentials

```bash
# Network shared by all the containers
docker network create app-network

# Build the image
docker build -t sakho/database ./Database

# Run the database (credentials passed at runtime, data persisted in ./Database/data)
docker run -d --name my-db --network app-network \
  -e POSTGRES_DB=db -e POSTGRES_USER=sakho -e POSTGRES_PASSWORD=root \
  -v $(pwd)/Database/data:/var/lib/postgresql/data \
  sakho/database

# Adminer to check the tables and the data in a browser (http://localhost:8090)
docker run -d -p 8090:8080 --net app-network --name adminer adminer
```

In Adminer I could see the `departments` and `students` tables with the data from `02-InsertData.sql`.

## Backend API

I started with a simple `Main.java` "Hello World" compiled with `javac`. I copied it into an `eclipse-temurin:21-jre-alpine` image and it printed `Hello World!`. Then I moved to the Spring Boot application (`simple-api`), which exposes students and departments from the database.

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME

RUN apk add --no-cache maven

COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run stage
FROM eclipse-temurin:21-jre-alpine
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

The connection to the database is configured in `BackendAPI/src/main/resources/application.yml`. The host is `database`, the name of the database service on the Docker network:

```yaml
datasource:
  url: jdbc:postgresql://database:5432/db
  username: usr
  password: pwd
```

### 1-4 Why a multistage build? Explanation of each step

To build the application we need a JDK and Maven. To run it we only need a JRE. With a multistage build, the first image compiles the code and the final image only contains the JRE and the `.jar`. The final image is much smaller and has fewer tools inside (smaller attack surface). Also, we don't need Java or Maven installed on our own machine to build the project.

Step by step:

| Instruction | What it does |
|---|---|
| `FROM eclipse-temurin:21-jdk-alpine AS myapp-build` | Build stage based on a JDK 21 image, named `myapp-build` so we can reference it later |
| `ENV MYAPP_HOME=/opt/myapp` / `WORKDIR $MYAPP_HOME` | Defines the app folder and moves into it |
| `RUN apk add --no-cache maven` | Installs Maven (without keeping the apk cache) |
| `COPY pom.xml .` / `COPY src ./src` | Copies the project descriptor and the source code |
| `RUN mvn package -DskipTests` | Compiles and packages the app into a `.jar` (tests are run in the CI, not here) |
| `FROM eclipse-temurin:21-jre-alpine` | Run stage: a new, lighter image with only the JRE |
| `COPY --from=myapp-build … myapp.jar` | Takes only the built `.jar` from the build stage |
| `ENTRYPOINT ["java", "-jar", "myapp.jar"]` | Starts the application when the container starts |

Results through the API:

```bash
$ curl http://localhost:8080
{"id":1,"content":"Hello, World!"}
$ curl http://localhost:8080/departments
[{"id":1,"name":"IRC"},{"id":2,"name":"ETI"},{"id":3,"name":"CGP"}]
$ curl http://localhost:8080/departments/IRC/students
[{"id":1,"firstname":"Eli","lastname":"Copter","department":{"id":1,"name":"IRC"}}]
```

## HTTP server (reverse proxy)

The HTTP server is based on `httpd:2.4`. I first served a simple `index.html`. Then I got the default configuration out of the container (`docker cp`) to modify it and copy my own `httpd.conf` into the image:

```dockerfile
FROM httpd:2.4
COPY ./public-html /usr/local/apache2/htdocs/
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
```

At the end of `httpd.conf` I enabled the proxy modules and added a virtual host. Requests on `/api/` go to the backend, and everything else goes to the front-end (added in part 3):

```apache
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass /api/ http://backend:8080/
ProxyPassReverse /api/ http://backend:8080/
ProxyPass / http://front:80/
ProxyPassReverse / http://front:80/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

### 1-5 Why do we need a reverse proxy?

The reverse proxy is the only entry point of the application: it's the only container with a port open on the host (80). The backend and the database stay hidden inside the Docker network. It also lets us:
- route requests to different services from one address (`/api/` → backend, `/` → front)
- add HTTPS, load balancing, caching or security headers in one place, without touching the applications

## Docker Compose

```yaml
services:
  database:
    build: ./Database
    container_name: my-database
    environment:
      POSTGRES_DB: db
      POSTGRES_USER: usr
      POSTGRES_PASSWORD: pwd
    networks:
      - app-network

  backend:
    build: ./BackendAPI
    container_name: backend
    depends_on:
      - database
    networks:
      - app-network

  reverse-proxy:
    build: ./HttpServer
    container_name: reverse-proxy
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

### 1-6 Why is docker-compose so important?

Without Compose, I had to create the network, build every image and run every container by hand, with long `docker run` commands in the right order. Compose describes the whole application in one versioned file. One command then builds, starts, stops or rebuilds everything, the same way on every machine.

### 1-7 Most important docker-compose commands

| Command | Description |
|---|---|
| `docker compose up -d` | Builds (if needed) and starts all services in the background |
| `docker compose up -d --build` | Forces the images to be rebuilt before starting |
| `docker compose down` | Stops and removes the containers and the network |
| `docker compose down -v` | Same, and also removes the volumes |
| `docker compose ps` | Lists the services and their status |
| `docker compose logs -f <service>` | Follows the logs of a service |
| `docker compose build` | Builds the images without starting them |
| `docker compose restart <service>` | Restarts a service |

### 1-8 My docker-compose file

- **database**: built from `./Database`. Its credentials are passed as environment variables. No port is published: it's only reachable from the backend through `app-network`.
- **backend**: built from `./BackendAPI`, `depends_on: database` so it starts after the database. No port is published either; it's reachable only through the proxy.
- **reverse-proxy**: built from `./HttpServer`. It's the only service with a published port (`80:80`) and it starts after the backend.
- **app-network**: a `bridge` network shared by the three services. They reach each other by service name (`database`, `backend`).

Then the API is available at `http://localhost/`.

### What I could improve

**Network segmentation.** Right now every container is on the same `app-network`, so the reverse proxy can reach the database directly, even though it never needs to. It would be safer to use **2 networks**:

- `proxy-network`: reverse-proxy + backend (+ front)
- `db-network`: backend + database

Only the backend would be on both networks. If the proxy (the only container exposed to the outside) were compromised, the attacker still couldn't reach the database.

```yaml
services:
  database:
    networks: [db-network]
  backend:
    networks: [proxy-network, db-network]
  reverse-proxy:
    networks: [proxy-network]

networks:
  proxy-network:
  db-network:
```

**Database credentials.** The database login info (`usr` / `pwd`) is written in plain text in `docker-compose.yml` and in `BackendAPI/src/main/resources/application.yml`, so it's committed to Git. This contradicts what I explained in 1-1. A cleaner way is to put them in a `.env` file that isn't committed (listed in `.gitignore`), and reference them as variables:

```yaml
# docker-compose.yml
database:
  environment:
    POSTGRES_DB: ${POSTGRES_DB}
    POSTGRES_USER: ${POSTGRES_USER}
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
backend:
  environment:
    SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
    SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
```

and in `application.yml`: `password: ${SPRING_DATASOURCE_PASSWORD}`. A stronger password would also be better than `pwd`.

## Publication on Docker Hub

```bash
docker login

# Tag the local image with my Docker Hub username and a version
docker tag sakho/database waraba/tp-database:1.0

# Push it
docker push waraba/tp-database:1.0
```

### 1-9 Published images

These images are published on my Docker Hub account `waraba`. Since part 2 they are pushed automatically by the CI:

| Image | Content |
|---|---|
| `waraba/tp-database` | PostgreSQL with schema and data |
| `waraba/tp-devops-simple-api` | Spring Boot backend |
| `waraba/tp-reverse-proxy` | Apache httpd reverse proxy |
| `waraba/tp-front` | Vue.js front-end served by Nginx |

### 1-10 Why put our images in an online repository?

The images then aren't stored only on my computer. Teammates, the CI pipeline and the production server can all pull exactly the same version of the application, without rebuilding it. With tags we also keep a history of versions and can go back to a previous one.

---

# Part 2 – GitHub Actions

The pipeline is in [`.github/workflows/main.yml`](.github/workflows/main.yml). It runs on every push on `main` and `develop`, and on pull requests. It has 3 jobs, chained with `needs`:

```
test-backend  ──►  build-and-push-docker-image  ──►  deploy
```

### Job 1 – `test-backend`: build, tests and quality gate

```yaml
test-backend:
  runs-on: ubuntu-24.04
  steps:
    - uses: actions/checkout@v4
    - name: Set up JDK 21
      uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'

    - name: Build and test with Maven
      run: mvn clean verify --file BackendAPI/pom.xml

    - name: SonarCloud Scan
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      run: >
        mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar
        -Dsonar.projectKey=sakho22_Docker_labs
        -Dsonar.organization=sakho22
        -Dsonar.host.url=https://sonarcloud.io
        --file BackendAPI/pom.xml
```

`mvn clean verify` cleans the previous build, compiles the application and runs the unit tests and the integration tests.

Problems I had along the way:
- `actions/setup-java` didn't work without a version: it must be `actions/setup-java@v4`.
- The path to the pom was wrong because of case sensitivity on Linux: `backendAPI/pom.xml` → `BackendAPI/pom.xml`.
- `sonar:sonar` from the subject failed ("couldn't find sonar"), because the plugin prefix couldn't be resolved. I used the full plugin name instead: `org.sonarsource.scanner.maven:sonar-maven-plugin:sonar`.

### 2-1 What are testcontainers?

Testcontainers are Java libraries that start real Docker containers during the tests. In this project the integration tests start a PostgreSQL container, so the tests run against a real database instead of a mock. The container is created at the start of the tests and removed at the end. It works the same way locally and in the CI, since GitHub runners have Docker.

### 2-2 Why do we need secured variables?

The pipeline needs credentials: the Docker Hub username and token, the SonarCloud token, and the SSH private key for the deployment. If they were written in `main.yml`, anyone who can read the repository could use them. GitHub secrets are encrypted, they are masked in the logs, and they are only injected when the workflow runs.

Secrets used: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `SONAR_TOKEN`, `SSH_PRIVATE_KEY`.

### Job 2 – `build-and-push-docker-image`

```yaml
build-and-push-docker-image:
  needs: test-backend
  runs-on: ubuntu-24.04
  steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Login to DockerHub
      run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login --username ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin

    - name: Build image and push backend
      uses: docker/build-push-action@v6
      with:
        context: ./BackendAPI
        tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-api:latest
        push: ${{ github.ref == 'refs/heads/main' }}
    # same step for ./Database, ./HttpServer and ./front
```

The images are built on every branch (to check that the Dockerfiles still work). They are **pushed only on `main`**, thanks to `push: ${{ github.ref == 'refs/heads/main' }}`.

### 2-3 Why `needs: test-backend` on this job?

By default, the jobs of a workflow run in parallel. Without `needs`, the images would be built and pushed even if the build or the tests failed, so a broken version could end up on Docker Hub. With `needs: test-backend`, this job only starts if the tests passed.

### 2-4 Why push the Docker images?

Pushing makes each validated version available in a registry. The server can then pull and run it, which is exactly what the Ansible deployment does in part 3. It's the "delivery" part of CI/CD: every commit on `main` that passes the tests produces a ready-to-deploy image.

### Quality gate – SonarCloud

I created an organization (`sakho22`) and a project (`sakho22_Docker_labs`) on SonarCloud, and added the `SONAR_TOKEN` secret on GitHub. Each run of the pipeline analyses the code: bugs, vulnerabilities, code smells, duplication and test coverage (with JaCoCo, configured in the `pom.xml`). The report is visible on SonarCloud.

---

# Part 3 – Ansible

Everything is in [`my-project/ansible/`](my-project/ansible/).

## 3-1 Inventory and base commands

`inventories/setup.yml`:

```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: /path/to/id_rsa
  children:
    prod:
      hosts:
        mohamed.sakho.takima.school:
```

- `all.vars`: variables used for every host: the SSH user and the private key provided for the TP
- `children.prod`: a `prod` group containing my server

Base commands:

```bash
# Check that Ansible can connect to the server
ansible all -i inventories/setup.yml -m ping

# Get facts about the server (here the OS distribution)
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"

# Remove apache2 from the server (desired state = absent)
ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become
```

Ansible is declarative: I describe the state I want, not the actions. If I run the last command a second time, nothing changes (`ok` instead of `changed`) because apache2 is already removed.

## 3-2 Playbook

```yaml
- hosts: all
  gather_facts: true
  become: true

  roles:
    - docker
    - role: network
      vars:
        ansible_python_interpreter: /opt/docker_venv/bin/python
    - role: database
      vars:
        ansible_python_interpreter: /opt/docker_venv/bin/python
    - role: app
      vars:
        ansible_python_interpreter: /opt/docker_venv/bin/python
    - role: front
      vars:
        ansible_python_interpreter: /opt/docker_venv/bin/python
    - role: proxy
      vars:
        ansible_python_interpreter: /opt/docker_venv/bin/python
```

- `hosts: all`: runs on every host of the inventory
- `gather_facts: true`: collects the facts, which the docker role needs (`distribution_release` for the APT repository)
- `become: true`: runs the tasks with sudo
- The roles run in order: install Docker → create the network → database → backend → front → proxy
- `ansible_python_interpreter: /opt/docker_venv/bin/python`: the `community.docker` modules need the Docker SDK for Python. The docker role installs it in a virtualenv, so the other roles must use that Python interpreter.

The roles were created with `ansible-galaxy init roles/<name>`. I only use the `tasks` (and `handlers`) folders.

**Role `docker`**: installs the prerequisites, adds the Docker GPG key and APT repository, installs `docker-ce` and Python 3, creates the virtualenv `/opt/docker_venv` with the `docker` SDK inside, and makes sure the Docker service is started.

Run the playbook:

```bash
cd my-project/ansible
ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check   # check first
ansible-playbook -i inventories/setup.yml playbook.yml
```

## 3-3 docker_container tasks configuration

All the application roles use the `community.docker` modules.

**network**: creates the network shared by all the containers:
```yaml
- name: Create the application network
  community.docker.docker_network:
    name: app-network
```

**database**:
```yaml
- name: Run the database
  community.docker.docker_container:
    name: my-database
    image: waraba/tp-database
    pull: true
    networks:
      - name: app-network
        aliases:
          - database
    env:
      POSTGRES_DB: db
      POSTGRES_USER: usr
      POSTGRES_PASSWORD: pwd
```
The `database` alias is needed because the backend connects to `database:5432`, but the container is named `my-database`.

**app** (backend):
```yaml
- name: Run the backend API
  community.docker.docker_container:
    name: backend
    image: waraba/tp-devops-simple-api
    pull: true
    networks:
      - name: app-network
    env:
      SPRING_DATASOURCE_URL: jdbc:postgresql://database:5432/db
      SPRING_DATASOURCE_USERNAME: usr
      SPRING_DATASOURCE_PASSWORD: pwd
```
The `SPRING_DATASOURCE_*` environment variables override the values in `application.yml`, so the database configuration is controlled from Ansible.

**front**:
```yaml
- name: Run the front
  community.docker.docker_container:
    name: front
    image: waraba/tp-front
    pull: true
    restart_policy: unless-stopped
    networks:
      - name: app-network
```

**proxy**:
```yaml
- name: Run the reverse proxy
  community.docker.docker_container:
    name: reverse-proxy
    image: waraba/tp-reverse-proxy
    pull: true
    networks:
      - name: app-network
    ports:
      - "80:80"
```

Common points:
- `image`: the images published on Docker Hub by the CI
- `pull: true`: always pulls the latest version of the image, so a redeploy picks up the new build
- `networks`: every container is on `app-network`, so they reach each other by name
- Only the proxy publishes a port (`80:80`). The database, backend and front are not reachable from outside.

### What I could improve

- **Two networks instead of one**, like for Docker Compose: the `network` role would create `proxy-network` and `db-network`. The database would only be attached to `db-network`, the proxy and front only to `proxy-network`, and the backend to both.
- **No plain-text database credentials in the roles.** `POSTGRES_PASSWORD: pwd` is written directly in the `database` and `app` roles. I could put the credentials in a variables file encrypted with **Ansible Vault** (`ansible-vault create group_vars/prod/vault.yml`). The roles would use `{{ db_password }}`, and the vault password would be given to the CI as a GitHub secret.

## Continuous Deployment

The `deploy` job runs the Ansible playbook from GitHub Actions after the images are pushed. It only runs on a **push on `main`**:

```yaml
deploy:
  needs: build-and-push-docker-image
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  runs-on: ubuntu-24.04
  steps:
    - uses: actions/checkout@v4

    - name: Install Ansible and the Docker collection
      run: |
        ansible --version || pipx install ansible-core
        ansible-galaxy collection install community.docker

    - name: Set up SSH key and known host
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/deploy_key
        chmod 600 ~/.ssh/deploy_key
        ssh-keyscan -H mohamed.sakho.takima.school >> ~/.ssh/known_hosts

    - name: Deploy with Ansible
      working-directory: my-project/ansible
      run: >
        ansible-playbook -i inventories/setup.yml playbook.yml
        -e ansible_ssh_private_key_file=~/.ssh/deploy_key
```

The private key comes from the `SSH_PRIVATE_KEY` secret and is written to a file on the runner. With `-e`, it overrides the local key path that is in the inventory.

### Is it really safe to deploy every new image automatically? How to make it more secure?

Not completely. Every push on `main` goes straight to production. If a bug isn't caught by the tests, or if someone pushes bad code (or the Docker Hub account is compromised), it is deployed immediately. Also, I use the `latest` tag, so it's hard to know exactly which version is running and to roll back.

To make it safer:
- Protect the `main` branch: changes only through pull requests, with review and a green CI required
- Block the deployment if the SonarCloud quality gate fails
- Use versioned tags (commit SHA or semantic version) instead of `latest`, so it's easy to roll back
- Deploy to a staging environment first, and add a manual approval before production (GitHub environments)
- Sign or scan the images (e.g. Trivy) before deploying them
- Keep the secrets in an Ansible Vault (the database credentials are still in plain text in the roles) and give the deployment key only the permissions it needs
- Separate the containers into two networks, so the exposed proxy can't reach the database directly

## Front-end

The front is a Vue.js application, built with a multistage Dockerfile: `node` builds the app, then `nginx` serves the static files.

```dockerfile
FROM node:12.16.3-stretch-slim AS build
WORKDIR /opt/front
COPY src/ src/
COPY public/ public/
COPY package.json .
COPY yarn.lock .
COPY default.conf .
COPY .env.production .
RUN npm install
RUN npm run build

FROM nginx:1.15.7-alpine
COPY --from=build /opt/front/dist /usr/share/nginx/html/
COPY --from=build /opt/front/default.conf /etc/nginx/conf.d/default.conf
```

- `.env.production` sets the API URL used by the front: `VUE_APP_API_URL=mohamed.sakho.takima.school/api`
- The reverse proxy sends `/api/` to the backend and everything else to the front (see `httpd.conf` above)
- I also added a `front` step in the CI (build and push of `tp-front`) and a `front` role in Ansible

The front is reachable at **http://mohamed.sakho.takima.school** and the API at **http://mohamed.sakho.takima.school/api/**.

---

## How to run the project

**Locally with Docker Compose:**
```bash
docker compose up -d --build
curl http://localhost/api/departments
```

**On the server with Ansible:**
```bash
cd my-project/ansible
ansible-galaxy collection install community.docker
ansible-playbook -i inventories/setup.yml playbook.yml
```

**Automatically:** push on `main` → tests + SonarCloud → images pushed to Docker Hub → Ansible deployment.
