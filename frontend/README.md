# Application Setup Guide

This guide provides step-by-step instructions to set up and run both the Backend (Java/Maven) and Frontend (Python/Streamlit) applications.

---

## Backend Setup

### 1. Create RDS Instance
Create an RDS database instance before proceeding with backend setup.


### 1. Install Dependencies
```bash
sudo yum install git -y
sudo yum install maven -y
```
### 3. Clone Repository
```bash
git clone https://github.com/CloudTechDevOps/Java-springboot-project.git
cd Java-springboot-project/backend
```
### 4. Configure RDS Details
Update the RDS connection details in:
```
/src/main/resources/application.properties
```
Ensure the database URL, username, and password match your RDS instance configuration.

### 5. Build Application
Run Maven build command from the backend directory:
```bash
mvn clean package -DskipTests=true
```

### 6. Navigate to Target Directory
```bash
cd target
```

### 7. Run JAR File
```bash
nohup java -jar datastore-0.0.7.jar > ~/datastore-nohup.out 2>&1 &  #if hard coded db crend
          or
nohup env \
  MYSQL_HOST=database-1.cxeakuo04ry1.us-east-1.rds.amazonaws.com \
  MYSQL_PORT=3306 \
  MYSQL_USERNAME=admin \
  MYSQL_PASSWORD=YOUR_PASSWORD_HERE \
  LOG_FILE_PATH=$LOG_DIR/datastore.log \
  java -jar target/datastore-0.0.7.jar \
  --server.port=8084 \
  > $LOG_DIR/nohup.out 2>&1 &
```
The backend application will now run on port **8084** (default configuration).
ps -ef | grep java #check process 

 ##run through systemd process-
 2. Create the application log directory

Your application.properties has:

logging.file.name=${LOG_FILE_PATH:/var/log/app/datastore.log}

So create the default directory:

sudo mkdir -p /var/log/app

Because your systemd service is currently going to run as root, it can write there.

3. Check your Spring Boot application.properties

Keep:

logging.file.name=${LOG_FILE_PATH:/var/log/app/datastore.log}

And database configuration should use your environment variables:

spring.datasource.url=jdbc:mysql://${MYSQL_HOST}:${MYSQL_PORT}/datastore
spring.datasource.username=${MYSQL_USERNAME}
spring.datasource.password=${MYSQL_PASSWORD}

server.port=8084
4. Create the systemd service
sudo vi /etc/systemd/system/datastore.service

Put:

[Unit]
Description=Student Management Spring Boot Backend
After=network.target

[Service]
User=root

WorkingDirectory=/root/Java-springboot-project/backend

Environment="MYSQL_HOST=database-1.cxeakuo04ry1.us-east-1.rds.amazonaws.com"
Environment="MYSQL_PORT=3306"
Environment="MYSQL_USERNAME=admin"
Environment="MYSQL_PASSWORD=YOUR_PASSWORD_HERE"

ExecStart=/usr/bin/java -jar /root/Java-springboot-project/backend/target/datastore-0.0.7.jar --server.port=8084

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

Notice: no nohup, no &, no > nohup.out.

systemd handles the process.

5. Reload systemd
sudo systemctl daemon-reload
6. Enable it for EC2 reboot
sudo systemctl enable datastore

This means:

EC2 reboot
    ↓
systemd
    ↓
datastore.service
    ↓
Spring Boot starts automatically
7. Start the application
sudo systemctl start datastore
8. Check whether Spring Boot started
sudo systemctl status datastore

You want:

Active: active (running)
9. Check the Spring Boot application log

Because you configured:

logging.file.name=${LOG_FILE_PATH:/var/log/app/datastore.log}

and did not provide LOG_FILE_PATH, Spring Boot uses the default:

/var/log/app/datastore.log

Check:

sudo tail -f /var/log/app/datastore.log

You should see Spring Boot startup messages.

10. Check systemd logs too

Run:

sudo journalctl -u datastore -f

This is different from your Spring Boot log file.

Think:

Spring Boot log
    ↓
/var/log/app/datastore.log

while:

systemd service output
    ↓
journalctl -u datastore
11. Test Spring Boot locally

On the backend EC2:

curl http://localhost:8084/student/all

If it returns something like:

[]

your application is responding.

---

## Frontend Setup

### 1. Install Dependencies
```bash
sudo yum install git -y
sudo yum install python3-pip -y
```
### 2. Clone Repository
```bash
git clone https://github.com/CloudTechDevOps/Java-springboot-project.git
cd Java-springboot-project/frontend
```
### 3. Configure Backend API URL
Update the backend server IP in `app.py`:
```python
API_URL = os.environ.get("API_URL", "http://<BACKEND_PUBLIC_IP>:8084")
```
Replace `<BACKEND_PUBLIC_IP>` with the public IP address of your backend server. Keep the port as **8084**.

### 4. Install Python Dependencies
```bash

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 5. Run Application
```bash
streamlit run app.py
     or run using pm2
pm2 start /root/myapp/venv/bin/streamlit \
  --name streamlit-app \
  --interpreter none \
  -- run /root/myapp/app.py --server.address=0.0.0.0 --server.port=8501
```
The frontend application will start on port **8501**.

streamlit is an open-source Python framework that lets you quickly build and share interactive web apps—especially for data science, machine learning, and analytics—without needing front-end skills like HTML, CSS, or JavaScript.

---

## Access the Application

Once both services are running fine

2. **Frontend Application**: `http://<FRONTEND_PUBLIC_IP>:8501`

Open your browser and navigate to the frontend URL using the public IP and port 8501.

---
Yes. For **your current project**, keep the explanation simple and think of each tool as having one job.

## Suggested project name

A good name for your project now, which will still make sense after you add networking, Terraform and CI/CD:

### **AWS 3-Tier Application Deployment & DevOps Automation**

You currently have:

```text
Frontend → Backend → RDS
```

and later you will add:

```text
Networking → Terraform → CI/CD → Monitoring/Security
```

---

# 1. Your current project

Right now you have:

```text
                    AWS
                     |
              Public EC2
              /          \
             /            \
      Frontend EC2      Backend EC2
       Streamlit        Spring Boot
         :8501            :8084
                            |
                            ↓
                         RDS MySQL
```

You have already achieved:

* Frontend running
* Spring Boot backend running
* RDS MySQL
* Frontend → Backend connection
* Backend → RDS connection
* Environment variables
* systemd for backend
* systemd for frontend

---

# 2. What is Maven?

**Maven = Java application build tool.**

Your Spring Boot source code is not directly deployed.

You have:

```text
Java source code
      ↓
    Maven
      ↓
   JAR file
      ↓
datastore-0.0.7.jar
```

You run:

```bash
mvn clean package
```

Maven compiles your Java code, downloads dependencies and creates the JAR.

Then systemd runs that JAR.

So:

```text
Maven → builds
systemd → runs
```

---

# 3. What is Spring Boot?

**Spring Boot = framework used to create your Java backend application.**

In your project:

```text
Spring Boot
     ↓
REST APIs
     ↓
JPA/Hibernate
     ↓
MySQL RDS
```

For example:

```text
GET /student/all
POST /student/add
```

Your Spring Boot application handles those requests.

---

# 4. What is systemd?

**systemd = Linux service/process manager.**

Instead of manually doing:

```bash
java -jar datastore-0.0.7.jar
```

or:

```bash
nohup java -jar ... &
```

you give the application to systemd.

```text
systemd
   ↓
Spring Boot JAR
```

systemd can:

* Start application
* Stop application
* Restart application
* Start after EC2 reboot
* Restart if application crashes
* Manage service logs

Your backend:

```text
datastore.service
       ↓
java -jar datastore-0.0.7.jar
       ↓
Spring Boot :8084
```

Your frontend:

```text
streamlit.service
       ↓
Streamlit :8501
```

---

# 5. What is `nohup`?

**nohup = simple way to keep a command running after you disconnect from SSH.**

Example:

```bash
nohup java -jar app.jar &
```

It is useful for simple testing.

But it doesn't give you the full service management that systemd provides.

### Your project

You started with:

```text
nohup
  ↓
Java
```

Now you have:

```text
systemd
  ↓
Java
```

For your final project, **use systemd instead of nohup**.

---

# 6. What is PM2?

**PM2 = process manager mainly used for Node.js applications**, although it can manage other processes too.

For example:

```text
PM2
 ↓
Node.js
```

You considered PM2 for Streamlit:

```text
PM2
 ↓
Streamlit
```

But since you're already using systemd for Spring Boot, you can simply use:

```text
systemd
 ↓
Streamlit
```

So you **don't need PM2 for this project**.

---

# 7. Simple comparison

| Tool            | What it does                       | Your project         |
| --------------- | ---------------------------------- | -------------------- |
| **Maven**       | Builds Java application            | Creates JAR          |
| **Spring Boot** | Java backend framework             | REST API             |
| **systemd**     | Runs/manages Linux services        | Frontend + Backend   |
| **nohup**       | Keeps command running after logout | Testing/old approach |
| **PM2**         | Process manager                    | Not necessary        |
| **Streamlit**   | Python frontend framework          | Frontend             |
| **RDS**         | Managed database                   | MySQL                |

The most important relationship is:

```text
Maven
  ↓
Build JAR
  ↓
Spring Boot application
  ↓
systemd manages it
  ↓
Runs on EC2
```

And:

```text
Python code
  ↓
Streamlit
  ↓
systemd manages it
  ↓
Runs on EC2
```

---

# 8. Where your project is going

You are currently here:

```text
PHASE 1 — APPLICATION
────────────────────────

Streamlit
     ↓
Spring Boot
     ↓
RDS MySQL
```

You have done this **using public EC2s**, which is fine for learning.

Next:

```text
PHASE 2 — NETWORKING
────────────────────────

VPC
├── Public Subnet
│    └── Bastion / Load Balancer
│
├── Private Frontend
│    └── Streamlit
│
├── Private Backend
│    └── Spring Boot
│
└── Private Database
     └── RDS
```

Then:

```text
PHASE 3 — TERRAFORM
────────────────────────

Terraform
    ↓
VPC
EC2
ALB
Security Groups
RDS
IAM
Route Tables
Subnets
...
```

Then:

```text
PHASE 4 — CI/CD
────────────────────────

GitHub
   ↓
Jenkins / GitHub Actions
   ↓
Maven build
   ↓
JAR
   ↓
Deploy
   ↓
systemd restart
   ↓
Application
```

Eventually your complete project can look like:

```text
                    GitHub
                       │
                       ↓
                 CI/CD Pipeline
                       │
                       ↓
                 Build / Deploy
                       │
                 ┌─────┴─────┐
                 ↓           ↓
             Streamlit   Maven Build
                 │           ↓
                 │       Spring Boot
                 │           │
                 └─────┬─────┘
                       ↓
                    AWS VPC
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
      Frontend      Backend       RDS
      Private EC2   Private EC2   Private
          │            │
       :8501        :8084
```

And Terraform will create the infrastructure:

```text
Terraform
    ↓
AWS Infrastructure
    ↓
Application Deployment
    ↓
CI/CD
```

### Your current achievement

Don't underestimate what you've already done. You have the **application layer working**:

**Streamlit → Spring Boot → RDS**, with **Maven building the backend JAR and systemd managing both application processes**.

Now the next major learning step is **take this working public-EC2 setup and redesign it as a proper VPC/private-subnet architecture**. Then Terraform and CI/CD will automate that architecture rather than trying to automate something you haven't verified manually.

