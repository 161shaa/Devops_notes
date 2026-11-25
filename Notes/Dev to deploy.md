# CI/CD & Deployment — Complete Notes

> A practical reference covering GitHub, GitHub Actions, Jenkins, SonarQube, Docker, and deployment concepts.

---

## Table of Contents

1. [What is CI/CD?](#1-what-is-cicd)
2. [The Tools — What Each One Does](#2-the-tools--what-each-one-does)
3. [Docker — Packaging Your Application](#3-docker--packaging-your-application)
4. [GitHub Actions Basics](#4-github-actions-basics)
5. [Jenkins Basics](#5-jenkins-basics)
6. [SonarQube — Code Quality Gate](#6-sonarqube--code-quality-gate)
7. [Deployment — Where Does Code Go?](#7-deployment--where-does-code-go)
8. [End-to-End Flow](#8-end-to-end-flow)
9. [Self-Hosted vs GitHub-Hosted Runners](#9-self-hosted-vs-github-hosted-runners)
10. [Quick Cheat Sheet](#10-quick-cheat-sheet)

---

## 1. What is CI/CD?

| Term | Meaning | Goal |
| :--- | :--- | :--- |
| **CI** (Continuous Integration) | Automatically build & test code every time a developer pushes changes | Catch bugs early, keep the main branch healthy |
| **CD** (Continuous Delivery / Deployment) | Automatically deliver / deploy the tested code to an environment | Get changes to users quickly and safely |

**Simple flow:**

```
Developer pushes code → Build → Test → Quality Check → Package (Docker image) → Deploy → Live Server
```

**Key idea:** You never manually copy files to a server. The pipeline does it for you, stage by stage.

---

## 2. The Tools — What Each One Does

| Tool | Role | Key Point |
| :--- | :--- | :--- |
| **GitHub** | Hosts your code (Git repository) | Source of truth for code |
| **Docker** | Packages app + runtime into a portable container image | "Build once, run anywhere" — eliminates environment drift |
| **GitHub Actions** | CI/CD built into GitHub | Runs workflows defined in `.github/workflows/*.yml` |
| **Jenkins** | Self-managed automation server | Flexible, plugin-based; you host and maintain it |
| **SonarQube** | Code quality & security scanner | Checks for bugs, code smells, vulnerabilities ("quality gate") |

**How they connect:**

```
GitHub (code) ──triggers──> GitHub Actions / Jenkins (pipeline)
                                    │
                                    ├──> Build   (compile, package into Docker image)
                                    ├──> Test    (unit, integration, inside container)
                                    ├──> SonarQube (quality gate)
                                    ├──> Push image to Container Registry
                                    └──> Deploy  (pull image → run on server / environment)
```

---

## 3. Docker — Packaging Your Application

### 3.1 What is Docker?

Docker is an open platform for developing, shipping, and running applications in **containers**—isolated environments that share the host machine's OS kernel but package everything else the app needs[reference:0].

A **container** holds everything required for an application to run: the OS libraries, runtime, dependencies, configuration, and your code—all as a single standard unit[reference:1].

### 3.2 Why Docker? (The Problem It Solves)

The classic problem: **"It works on my machine but not on yours."**

Before containers, a developer's laptop, the test server, and production often had different OS versions, library versions, and configurations. Code that worked locally would break elsewhere[reference:2].

Docker solves this by **shipping the environment along with the code**:

- **Build once, run anywhere** — the same image runs identically on a laptop, a CI runner, and a production server[reference:3].
- **Lightweight** — containers do **not** include a full guest OS (unlike virtual machines), so they start in seconds and use far fewer resources[reference:4][reference:5].
- **Reproducible** — the Dockerfile is version-controlled, so the exact environment is codified and repeatable[reference:6].

### 3.3 Container vs. Virtual Machine

| | Virtual Machine | Docker Container |
| :--- | :--- | :--- |
| **OS** | Full guest OS per VM | Shares host OS kernel |
| **Startup** | Minutes | Seconds |
| **Resource usage** | Heavy (pre-allocated CPU/RAM) | Lightweight, dynamic |
| **Isolation** | Strong (hypervisor-level) | Good (kernel-level; not perfect) |

Containers are Linux-based (though they work on Windows via a Linux kernel layer), and isolation is slightly less strict than a VM because they share the underlying kernel[reference:7].

### 3.4 Core Docker Vocabulary

| Term | Meaning |
| :--- | :--- |
| **Dockerfile** | A text file describing the steps to build an image |
| **Image** | A read-only blueprint/snapshot of an environment (name + tag, e.g., `my-app:1.0`) |
| **Container** | A running (read-write) instance of an image |
| **Layer** | Each instruction in a Dockerfile creates a cached, read-only layer |
| **Registry** | A warehouse for images (Docker Hub, GitHub Container Registry, Amazon ECR, etc.) |

Source: [reference:8]

### 3.5 How a Developer Uses Docker (Daily Workflow)

The inner-loop development workflow with Docker looks like this[reference:9]:

```
1. Write code
2. Write a Dockerfile
3. Build the image        →  docker build -t my-app:1.0 .
4. (Optional) Define multi-service setup in docker-compose.yml
5. Run the container      →  docker run -p 8080:8080 my-app:1.0
6. Test the app
7. Push the image to a registry  →  docker push <registry>/my-app:1.0
8. Repeat
```

**The critical shift:** you stop shipping *code* and start shipping *images*. The server pulls the pre-built image from the registry and runs it—it never builds your code from source[reference:10].

### 3.6 A Minimal Dockerfile (Spring Boot Example)

```dockerfile
# Stage 1: Build the JAR
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run the JAR in a slim runtime image
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

This is a **multi-stage build**—the build tools stay in Stage 1, and only the final JAR and a minimal JRE go into the runtime image. The result is smaller, faster to pull, and more secure[reference:11].

### 3.7 Docker in CI/CD

Docker changes the CI/CD pipeline significantly[reference:12]:

| Without Docker | With Docker |
| :--- | :--- |
| CI runner must have every language, SDK, and tool pre-installed | CI runner only needs Docker installed |
| Builds depend on runner OS and versions | Builds happen inside the Dockerfile—fully reproducible |
| Artifact is a JAR or binary | Artifact is a **container image** |
| Deploy = copy files + configure server | Deploy = pull image + run container |

**Updated pipeline stages with Docker:**

```
Push code → Build image → Test inside container → SonarQube scan
→ Push image to registry → Deploy (pull image on server) → Running
```

---

## 4. GitHub Actions Basics

- **Workflow**: A YAML file inside `.github/workflows/` that defines the pipeline.
- **Trigger (`on`)**: What starts the workflow (e.g., `push`, `pull_request`).
- **Job**: A set of steps that run on one runner.
- **Step**: A single task (run a command or use an action).
- **Runner**: The machine that executes the job.

**Minimal Docker-based CI workflow — `.github/workflows/ci.yml`:**

```yaml
name: Docker CI Pipeline
on:
  push:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build Docker image
        run: docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .

      - name: Test inside container
        run: docker run --rm ghcr.io/${{ github.repository }}:${{ github.sha }} mvn test

      - name: Push image to registry
        run: docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

**Useful concepts:**
- **Secrets** — store registry tokens/passwords in GitHub → Settings → Secrets.
- **Artifacts** — save build outputs (test reports, etc.) between jobs.
- **Matrix** — run the same job across multiple versions.

---

## 5. Jenkins Basics

- **Controller** — The Jenkins server that schedules jobs.
- **Agent / Node** — The machine that actually runs the build.
- **Jenkinsfile** — Defines the pipeline in code (stored in your repo).
- **Plugins** — Extend functionality (Git, Docker, Maven, SonarQube, etc.).

**Jenkins vs GitHub Actions:**

| Aspect | GitHub Actions | Jenkins |
| :--- | :--- | :--- |
| Setup | Zero — built into GitHub | You install & maintain |
| Hosting | Managed by GitHub | Self-hosted (you own the server) |
| Best for | Simple → medium pipelines | Complex custom pipelines, on-prem / legacy |
| Flexibility | Very good | Extremely high (plugins) |

**Minimal `Jenkinsfile` with Docker:**

```groovy
pipeline {
    agent any
    stages {
        stage('Build Image') {
            steps { sh 'docker build -t my-app:${BUILD_NUMBER} .' }
        }
        stage('Test') {
            steps { sh 'docker run --rm my-app:${BUILD_NUMBER} mvn test' }
        }
        stage('SonarQube') {
            steps { sh 'mvn sonar:sonar' }
        }
        stage('Push Image') {
            steps { sh 'docker push my-registry/my-app:${BUILD_NUMBER}' }
        }
        stage('Deploy') {
            steps { sh './deploy.sh' }
        }
    }
}
```

---

## 6. SonarQube — Code Quality Gate

- **What it does:** Static analysis — finds bugs, code smells, security vulnerabilities, and measures test coverage.
- **Quality Gate:** A pass/fail rule (e.g., "no new bugs, coverage > 80%"). If it fails, the pipeline should stop.
- **Where it fits:** Usually runs **after build/test**, **before deploy**.

---

## 7. Deployment — Where Does Code Go?

Code is deployed to **environments** — machines / servers where the app runs:

| Environment | Purpose | Users |
| :--- | :--- | :--- |
| **Development** | Dev/QA testing | Developers, QA |
| **Staging (Pre-Prod)** | Mirror of production, final checks | QA, stakeholders |
| **Production** | Live, real users | Everyone |

**Deployment targets can be:**
- Linux VM (e.g., AWS EC2) — running Docker containers
- Kubernetes cluster
- PaaS (Azure App Service, Render)
- On-prem servers

> **Key change with Docker:** You no longer copy JARs or configure servers manually. The server pulls the pre-built image from the registry and runs it as a container[reference:13].

**Typical deployment flow:**

```
Docker image pushed to registry
        │
        ├──> Pull & run on Dev       (auto)
        ├──> Pull & run on Staging   (auto after tests pass)
        └──> Pull & run on Production (manual approval or auto)
```

---

## 8. End-to-End Flow

Putting everything together — a Docker-based pipeline:

```
1. Developer pushes code to GitHub
2. GitHub Actions (or Jenkins) triggers on push
3. Pipeline runs:
     a. Checkout code
     b. Build Docker image  (docker build)
     c. Test inside container (docker run ... mvn test)
     d. SonarQube scan → quality gate
     e. Push image to Container Registry (docker push)
     f. Deploy: server pulls image and runs container
        ├──> Dev / Staging   (auto)
        └──> Production      (manual or auto)
4. App is live on the target server
```

Source: [reference:14]

---

## 9. Self-Hosted vs GitHub-Hosted Runners

| Aspect | GitHub-Hosted Runner | Self-Hosted Runner |
| :--- | :--- | :--- |
| **Management** | Fully managed by GitHub | You manage everything (OS, patches, scaling) |
| **Environment** | Fresh, ephemeral VM per job | Persistent machine you control |
| **Setup Effort** | Zero — ready to use immediately | Requires installation, registration, maintenance |
| **Best For** | Standard workflows, quick start | Special hardware, compliance, high-volume cost, private networks |

**Docker note:** If your runner has Docker installed, self-hosted runners can build and run containers just like hosted ones. The Docker daemon needs to be available on the machine.

---

## 10. Quick Cheat Sheet

| Question | Answer |
| :--- | :--- |
| Where does code live? | GitHub (Git repo) |
| What packages the app + environment? | Docker (Dockerfile → Image) |
| What runs the pipeline? | GitHub Actions or Jenkins |
| What checks code quality? | SonarQube |
| Where is the image stored? | Container Registry (Docker Hub, ghcr.io, ECR, etc.) |
| Where does the app deploy to? | Dev → Staging → Production (pull image & run container) |
| How does deployment happen? | Pipeline pushes image; server pulls and runs it |
| What is a runner? | The machine that executes pipeline jobs |
| Hosted vs self-hosted runner? | Hosted = GitHub manages; Self-hosted = you manage |
| Where are pipeline definitions kept? | GitHub Actions: `.github/workflows/*.yml` · Jenkins: `Jenkinsfile` |
| Where do secrets go? | GitHub → Settings → Secrets; Jenkins → Credentials |

---

## References

- [Docker — What is a Container?](https://www.docker.com/resources/what-container/)
- [Docker Docs](https://docs.docker.com/)
- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [Jenkins Docs](https://www.jenkins.io/doc/)
- [SonarQube Docs](https://docs.sonarsource.com/)
- [Development workflow for Docker apps (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/docker-application-development-process/docker-app-development-workflow)

---

*Last updated: 2026-11-26*
