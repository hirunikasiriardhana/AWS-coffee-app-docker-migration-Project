# ☕ Coffee Suppliers App — Docker Migration Project

> Migrating a Node.js + MySQL web application from EC2 guest OS installs to fully containerized Docker workloads, then publishing the image to Amazon ECR.

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-8.0.23-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20ECR-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)

---

## 📖 Overview

A café business acquired a coffee supplier company that ran its **inventory tracking web app** directly on two EC2 instances (app server + MySQL server). This project migrates that application to **Docker containers** — making it portable, reproducible, and ready for scalable deployment — and pushes the final image to **Amazon Elastic Container Registry (ECR)**.

**Architecture shift:**

```
BEFORE                              AFTER
┌─────────────────┐                ┌────────────────────────────────┐
│  AppServerNode  │                │   Docker Host (EC2)            │
│ (Node.js on OS) │                │   ┌───────────┐                │
└─────────────────┘                │   │ node_app  │──┐             │
┌─────────────────┐    ──────►     │   │ container │  │  network    │
│ MysqlServerNode │                │   └───────────┘  │             │
│ (MySQL on OS)   │                │   ┌───────────┐  │             │
└─────────────────┘                │   │ mysql_1   │◄─┘             │
                                   │   │ container │                │
                                   │   └───────────┘                │
                                   └────────────────────────────────┘
                                                   │
                                                   ▼
                                    📦 Amazon ECR (node-app:latest)
```

---

## 🎯 Lab Objectives

- [x] Create a `Dockerfile` for a Node.js application
- [x] Build a Docker image from a `Dockerfile`
- [x] Run and manage containers from Docker images
- [x] Pass runtime configuration via environment variables
- [x] Dump, migrate, and seed a MySQL database into a container
- [x] Connect containers to each other over the Docker bridge network
- [x] Create an Amazon ECR repository
- [x] Authenticate the Docker client to ECR and push an image

---

## 🛠️ Tech Stack

| Layer            | Technology                          |
|-------------------|--------------------------------------|
| Application       | Node.js (Express framework)          |
| Database          | MySQL 8.0.23                         |
| Containerization  | Docker (`node:11-alpine`, `mysql:8.0.23`) |
| Cloud Platform    | AWS EC2, Amazon ECR                  |
| Dev Environment   | VS Code IDE (code-server on EC2)     |

---

## 📋 Task-by-Task Walkthrough

### Task 1 — Environment Setup
Provisioned the VS Code IDE, downloaded the lab codebase, and ran the setup script to configure the AWS CLI, Python SDK (`boto3`), and baseline IAM/security group permissions.

### Task 2 — Baseline Application Analysis
Verified the original app running directly on the `AppServerNode` EC2 instance's guest OS (Node.js on port 80), added and edited a supplier record to confirm CRUD functionality end-to-end.

<img width="1918" height="868" alt="01-original-app-home" src="https://github.com/user-attachments/assets/a0059377-eacd-410d-8de5-b1ee94b46d78" />

<img width="1917" height="872" alt="02-original-app-supplier-added" src="https://github.com/user-attachments/assets/8e1305bb-17fe-4eec-91e1-1a326883a369" />

### Task 3 — Containerizing the Node.js Application

**Dockerfile** (`containers/node_app/codebase_partner/Dockerfile`):

```dockerfile
FROM node:11-alpine
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm", "run", "start"]
```

Built the image and ran the container:

```bash
docker build --tag node_app .
docker run -d --name node_app_1 -p 3000:3000 node_app
```

Verified the app served HTML on port 3000 via `curl`, then exposed port 3000 in the instance's security group to reach it from the browser.

### Task 4 — Containerizing the MySQL Database

Dumped the live database, adjusted a data value for later verification, and built a MySQL image that seeds itself from the dump:

```bash
mysqldump -P 3306 -h <mysql-host-ip> -u nodeapp -p --databases COFFEE > my_sql.sql
```

**Dockerfile** (`containers/mysql/Dockerfile`):

```dockerfile
FROM mysql:8.0.23
COPY ./my_sql.sql /
EXPOSE 3306
```

```bash
docker build --tag mysql_server .
docker run --name mysql_1 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=rootpw -d mysql_server
docker exec -i mysql_1 mysql -u root -prootpw < my_sql.sql
docker exec -i mysql_1 mysql -u root -prootpw -e \
  "CREATE USER 'nodeapp' IDENTIFIED WITH mysql_native_password BY 'coffee'; GRANT all privileges on *.* to 'nodeapp'@'%';"
```

### Task 5 — Connecting the Two Containers

Discovered the MySQL container's internal bridge-network IP with `docker inspect`, then relaunched the Node app container pointing at it via an environment variable:

```bash
docker inspect -f '{{.NetworkSettings.IPAddress}}' <mysql_container_id>
# → 172.17.0.3

docker run -d --name node_app_1 -p 3000:3000 -e APP_DB_HOST=172.17.0.3 node_app
```

Confirmed success when the **modified address ("Container Street")** appeared in the supplier list — proof the app was now reading from the *containerized* database, not the original EC2 MySQL instance.

### Task 6 — Pushing the Image to Amazon ECR

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

aws ecr create-repository --repository-name node-app

docker tag node_app:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/node-app:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/node-app:latest
```

Verified with:

```bash
aws ecr list-images --repository-name node-app
```

---

## 🐛 Bugs Encountered & Fixes

| # | Bug | Root Cause | Fix |
|---|-----|-----------|-----|
| 1 | Browser couldn't reach `http://<IP>:3000` (`ERR_CONNECTION_TIMED_OUT`) | Security group had **no inbound rule for port 3000** | Added a Custom TCP inbound rule for port 3000, source = My IP |
| 2 | "There was a problem retrieving the list of suppliers" | Node container was using the **hardcoded default DB host** from `config.js` — no `APP_DB_HOST` env var was passed at `docker run` | Recreated the container with `-e APP_DB_HOST=<mysql-ip>` |
| 3 | `mysqldump` → `Access denied for user 'nodeapp'@'...' (using password: NO)` | Command was executed without letting the interactive `Enter password:` prompt appear, so an empty password was sent | Ran the command, waited for the prompt, then typed the password |
| 4 | `docker build` → `ERROR: failed to solve: the Dockerfile cannot be empty` | Pasted Dockerfile content into the **wrong file** (a stray `Dockerfile` was auto-created one directory level up) | Wrote the Dockerfile directly from the terminal with a heredoc (`cat > Dockerfile << 'EOF' ... EOF`) inside the correct `mysql/` directory |
| 5 | `git push` → `GH007: Your push would publish a private email address` | Local Git `user.email` was set to a real address not authorized for public pushes under GitHub's privacy protection | Set `git config --global user.email` to the GitHub-provided `@users.noreply.github.com` address and amended the commit |

<img width="713" height="652" alt="04-bug-port3000-connection-timeout" src="https://github.com/user-attachments/assets/7ac836f2-c8f4-45f7-85cf-a2931d3bd991" />

<img width="1918" height="863" alt="06-bug-db-connection-error" src="https://github.com/user-attachments/assets/04d3190a-f9f8-4aa1-8097-649695e8f78d" />

---

## ✅ Verification Screenshots

**Security group opened for port 3000:**

<img width="1918" height="861" alt="03-security-group-before-fix" src="https://github.com/user-attachments/assets/610829a1-77bb-45f5-8fd6-f1174489b7af" />

**Containerized app reachable on port 3000:**

<img width="1918" height="966" alt="05-containerized-app-home-working" src="https://github.com/user-attachments/assets/f06b0a18-305d-43fe-af03-69172abd1528" />

**Node container successfully querying the original EC2 MySQL instance:**

<img width="1918" height="982" alt="07-container-connected-to-ec2-mysql" src="https://github.com/user-attachments/assets/97e8f455-82bf-4690-8609-3b038cab5cf1" />

**Final proof: Node container ↔ MySQL container (data shows "Container Street"):**

<img width="1918" height="870" alt="08-final-container-to-container-success" src="https://github.com/user-attachments/assets/1e591ced-5c67-4484-ae27-504ccde7d737" />

**All EC2 instances healthy and running:**

<img width="1918" height="815" alt="09-ec2-instances-running" src="https://github.com/user-attachments/assets/be3ccf21-c1d0-400e-af21-bcfc7a432d1e" />

---

## 📂 Project Structure

```
containers/
├── node_app/
│   └── codebase_partner/
│       ├── Dockerfile
│       ├── app/
│       ├── views/
│       ├── public/
│       └── package.json
├── mysql/
│   ├── Dockerfile
│   └── my_sql.sql
screenshots/
└── *.png
README.md
```

---

## 🚀 Key Commands Reference

```bash
# Build images
docker build --tag node_app .
docker build --tag mysql_server .

# Run containers
docker run -d --name mysql_1 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=rootpw -d mysql_server
docker run -d --name node_app_1 -p 3000:3000 -e APP_DB_HOST=<mysql-ip> node_app

# Inspect / debug
docker ps
docker inspect -f '{{.NetworkSettings.IPAddress}}' <container_id>
docker exec -ti <container_id> sh

# ECR
aws ecr create-repository --repository-name node-app
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/node-app:latest
```

---

## 📌 What I Learned

- How to translate a manually-provisioned app into a repeatable `Dockerfile`
- Why environment variables (not hardcoded config) are essential for portable containers
- How Docker's default bridge network assigns internal IPs and enables container-to-container communication
- The full lifecycle of pushing an image to a private registry (ECR) for future deployment
- Practical debugging of security groups, container networking, and Git authentication issues

---

## 🔮 Next Steps

Per the lab scenario, the next phase deploys this ECR-hosted image using **AWS Elastic Beanstalk**, with the MySQL layer migrating to **Amazon RDS** instead of a self-managed container — for built-in backups, patching, and high availability.

---

<p align="center">Built as part of an AWS Docker containerization lab 🐳</p>
