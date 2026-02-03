# Horilla CI/CD - Deployment Architecture Guide

This document explains the "Why" and "What" behind each file added to the repository for the CI/CD pipeline. The goal of this architecture is to move from manual deployments to a **fully automated, containerized workflow**.

---

## 1. `.github/workflows/deploy.yml`
### **The "Brain" of the Operation**

*   **Why?** Without this, someone would have to manually build the code, push it to a server, and restart the app every time a change is made. This file automates that entire process.
*   **What it does:**
    *   **Trigger:** It waits for you to "push" code to the `master` branch.
    *   **Build & Push:** It starts a temporary virtual machine (Ubuntu), builds your Docker image, and sends it to **Docker Hub**.
    *   **Remote Control:** It then "talks" to your EC2 instance via SSH and tells it: *"Hey, there's a new version. Pull it and restart the app."*

## 2. `docker-compose.prod.yaml`
### **The "Architect's Blueprint"**

*   **Why?** A modern app isn't just one thing; it's a team. You need the App, a Database (PostgreSQL), and a Web Server (Nginx). Managing these separately is a headache.
*   **What it does:**
    *   It defines how these three "containers" (App, DB, Nginx) connect to each other.
    *   It ensures that if the server restarts, the app restarts too.
    *   It manages **Volumes**, which are like external hard drives so your database data doesn't disappear when you update the app.

## 3. `.dockerignore`
### **The "Security/Speed Guard"**

*   **Why?** When building a Docker image, you don't want to include your entire computer. Files like `.git`, screenshots, or local logs just make the image huge and slow to upload.
*   **What it does:**
    *   It tells Docker: *"Ignore these specific files when you are building the image."*
    *   This makes your builds **5x to 10x faster** and keeps the final image clean and secure.

## 4. `nginx/horilla.conf`
### **The "Front Door" (Reverse Proxy)**

*   **Why?** You should never expose your Python/Django app directly to the internet (on port 8000). It's not designed for high traffic or security. Nginx is.
*   **What it does:**
    *   It sits at the front (Port 80) and takes all the visitor traffic.
    *   It "forwards" the requests to your app in the background.
    *   It handles **Static Files** (CSS, Images) much faster than Django can, making your website feel snappy.

## 5. `scripts/setup_ec2.sh`
### **The "Foundational Handshake"**

*   **Why?** A brand-new Ubuntu server from AWS is "naked"—it doesn't have Docker or Docker Compose installed. Doing this manually for every new server is repetitive.
*   **What it does:**
    *   It automatically installs Docker, Docker Compose, and sets permissions in one go.
    *   It turns a "plain" server into a "Horilla-ready" hosting machine in about 60 seconds.

---

### **How they all work together:**
1. You run `setup_ec2.sh` **once** on AWS.
2. You push code to GitHub.
3. GitHub sees `deploy.yml` and builds an image, ignoring junk via `.dockerignore`.
4. GitHub tells your EC2 to run `docker-compose.prod.yaml`.
5. `docker-compose` brings up the App, DB, and Nginx.
6. `nginx.conf` directs people to your app.
