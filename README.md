# 🐳 Student Registration App — Three-Tier Architecture with Docker

<div align="center">

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Docker Compose](https://img.shields.io/badge/Docker_Compose-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Apache Tomcat](https://img.shields.io/badge/Apache_Tomcat-F8DC75?style=for-the-badge&logo=apachetomcat&logoColor=black)
![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=for-the-badge&logo=mariadb&logoColor=white)
![Java](https://img.shields.io/badge/Java-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)
![Apache](https://img.shields.io/badge/Apache_httpd-D22128?style=for-the-badge&logo=apache&logoColor=white)

*Three-tier Student Registration app containerized with Docker — HTML/JS frontend (Apache), Java WAR backend (Tomcat 9), and MariaDB database — deployable via Docker Compose in one command*

</div>

---

## 📌 Overview

A fully containerized **Student Registration CRUD Application** built on a classic three-tier architecture. Each tier runs in its own Docker container and communicates over a shared Docker network. The project supports **two deployment approaches** — manual Docker commands for learning individual container concepts, and **Docker Compose** for a single-command full-stack startup.

---

## 🏗️ Architecture

```
  Browser
     │
     ▼
┌─────────────────────────────────────────────────────┐
│                  Docker Network                      │
│                                                      │
│  ┌───────────────────────────────────────────────┐  │
│  │  Tier 1 — Frontend (frontend-service)         │  │
│  │  Amazon Linux + Apache httpd                  │  │
│  │  Static HTML/CSS/JS   Port: 80                │  │
│  └──────────────────────────┬────────────────────┘  │
│                             │ Links to backend URL   │
│  ┌──────────────────────────▼────────────────────┐  │
│  │  Tier 2 — Backend (backend-service)           │  │
│  │  Ubuntu 24.04 + JDK + Apache Tomcat 9         │  │
│  │  student.war deployed on Tomcat  Port: 8080   │  │
│  └──────────────────────────┬────────────────────┘  │
│                             │ JDBC via jdbc/TestDB   │
│  ┌──────────────────────────▼────────────────────┐  │
│  │  Tier 3 — Database (database-service)         │  │
│  │  MariaDB — auto-initialised via student-rds.sql│  │
│  │  Database: studentapp       Port: 3306        │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## 📁 Repository Structure

```
student-app-docker-compose/
│
├── Frontend/
│   ├── Dockerfile              # Amazon Linux + Apache httpd + index.html
│   ├── index.html              # Animated welcome page — links to backend
│   └── readme.md
│
├── Backend/
│   ├── Dockerfile              # Ubuntu 24.04 + JDK + Tomcat 9 + student.war
│   ├── context.xml             # JDBC DataSource config (DB connection)
│   └── readme.md
│
├── Database/
│   ├── Dockerfile              # MariaDB image + auto-init SQL
│   ├── student-rds.sql         # Creates studentapp DB + students table
│   └── readme.md
│
├── docker-compose.yaml         # Full-stack single-command deployment
├── compose-steps.md            # Docker Compose setup guide
└── README.md
```

---

## 🛠️ Tech Stack

| Tier | Technology | Base Image |
|---|---|---|
| **Frontend** | Static HTML + CSS + JS served via Apache httpd | `amazonlinux` |
| **Backend** | Java WAR on Apache Tomcat 9.0.111 + MySQL Connector | `ubuntu:24.04` |
| **Database** | MariaDB — auto-initialised with SQL schema | `mariadb` |
| **Orchestration** | Docker Compose v3.8 | — |

---

## 🗄️ Database Schema

```sql
-- Database: studentapp
CREATE TABLE IF NOT EXISTS students (
    student_id           INT           NOT NULL AUTO_INCREMENT,
    student_name         VARCHAR(100)  NOT NULL,
    student_addr         VARCHAR(100)  NOT NULL,
    student_age          INT           NOT NULL,
    student_qual         VARCHAR(20)   NOT NULL,
    student_percent      DECIMAL(5,2)  NOT NULL,
    student_year_passed  YEAR          NOT NULL,
    PRIMARY KEY (student_id)
);
```

The `student-rds.sql` file is automatically loaded at container startup via the `/docker-entrypoint-initdb.d/` entrypoint — no manual database setup needed.

---

## 🚀 Option A — Deploy with Docker Compose (Recommended)

The fastest way to run the entire three-tier stack.

### Prerequisites
- Docker (v20+)
- Docker Compose (v2+)

### Steps

**1. Clone the repo**
```bash
git clone https://github.com/Sanket006/student-app-docker-compose.git
cd student-app-docker-compose
```

**2. Update the backend URL in the frontend**

Edit `Frontend/index.html` and replace the hardcoded IP with your server's public IP:
```html
<!-- Find this line and update the IP -->
<a href="http://<YOUR-SERVER-IP>:8080/student/">Register Here</a>
```

**3. Start all three services**
```bash
docker compose build
docker compose up -d
```

**4. Verify all containers are running**
```bash
docker compose ps
```

**5. Access the application**
```
Frontend:  http://<YOUR-SERVER-IP>:80
Backend:   http://<YOUR-SERVER-IP>:8080/student
```

**6. Stop everything**
```bash
docker compose down
```

---

## 🔧 Option B — Manual Docker (Step by Step)

Use this approach to understand how each tier connects individually.

### Step 1 — Build and Run the Database Container

```bash
cd Database/

docker build -t student-db .
docker run -itd --name database-service -p 3306:3306 student-db
```

**Get the database container's IP** — needed for the backend config:
```bash
docker inspect database-service | grep '"IPAddress"'
```
Note this IP (e.g. `172.17.0.2`).

**Verify the database initialised correctly:**
```bash
docker exec -it database-service mariadb --user root -p1234

# Inside MariaDB shell
SHOW DATABASES;
USE studentapp;
SHOW TABLES;
EXIT;
```

---

### Step 2 — Configure and Run the Backend Container

Edit `Backend/context.xml` — replace the IP with the database container IP from Step 1:

```xml
<Resource name="jdbc/TestDB" auth="Container" type="javax.sql.DataSource"
          maxTotal="100" maxIdle="30" maxWaitMillis="10000"
          username="root" password="1234"
          driverClassName="com.mysql.jdbc.Driver"
          url="jdbc:mysql://<DATABASE-CONTAINER-IP>:3306/studentapp"/>
```

```bash
cd ../Backend/

docker build -t student-backend .
docker run -itd --name backend-service -p 8080:8080 student-backend
```

**Verify the backend is serving:**
```bash
curl http://localhost:8080/student
```

---

### Step 3 — Configure and Run the Frontend Container

Edit `Frontend/index.html` — update the backend URL:
```html
<a href="http://<YOUR-SERVER-IP>:8080/student/">Register Here</a>
```

```bash
cd ../Frontend/

docker build -t student-frontend .
docker run -itd --name frontend-service -p 80:80 student-frontend
```

**Access the application:**
```
http://<YOUR-SERVER-IP>:80
```

---

## 📋 docker-compose.yaml

```yaml
version: '3.8'

services:
  db:
    build:
      context: ./Database
      dockerfile: Dockerfile
    container_name: database-service
    environment:
      MYSQL_ROOT_PASSWORD: 1234
      MYSQL_DATABASE: studentapp
    ports:
      - "3306:3306"

  backend:
    build:
      context: ./Backend
      dockerfile: Dockerfile
    container_name: backend-service
    environment:
      DB_HOST: db
      DB_PORT: 3306
      DB_NAME: studentapp
      DB_USER: root
      DB_PASSWORD: 1234
    ports:
      - "8080:8080"
    depends_on:
      - db

  frontend:
    build:
      context: ./Frontend
      dockerfile: Dockerfile
    container_name: frontend-service
    ports:
      - "80:80"
    depends_on:
      - backend
```

---

## 🔍 Useful Commands

```bash
# Check all running containers
docker ps

# View logs for a specific service
docker compose logs -f backend
docker compose logs -f db

# Restart a single service
docker compose restart backend

# Exec into the database container
docker exec -it database-service mariadb --user root -p1234

# Stop and remove all containers + networks
docker compose down

# Stop and also remove volumes (wipes DB data)
docker compose down -v
```

---

## ⚠️ Security Notes for Production

| Issue | Location | Fix |
|---|---|---|
| Hardcoded DB password `1234` | `context.xml`, `docker-compose.yaml` | Use Docker secrets or environment variables from a `.env` file — never commit real passwords |
| Hardcoded container IP in `context.xml` | `Backend/context.xml` | Use the Docker Compose **service name** `db` as hostname instead of a static IP |
| Hardcoded public IP in frontend | `Frontend/index.html` | Use an environment variable or a reverse proxy (Nginx) to inject the backend URL at build time |

---

## 🔗 Related Projects

| Repo | Description |
|---|---|
| [student-app-kubernetes](https://github.com/Sanket006/student-app-kubernetes) | Same app scaled up on Kubernetes with HPA and Secrets |
| [crud-app-aws-ec2-rds](https://github.com/Sanket006/crud-app-aws-ec2-rds) | Same app deployed on AWS EC2 with managed RDS |
| [jenkins-cicd-pipelines](https://github.com/Sanket006/jenkins-cicd-pipelines) | CI/CD pipeline to automate Docker builds and deployments |

---

## 👨‍💻 Author

**Sanket Ajay Chopade** — DevOps Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/sanketchopade07)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white)](https://github.com/Sanket006)
