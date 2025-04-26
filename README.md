# Ktor Backend Deployment Guide with PostgreSQL

This document provides a step-by-step guide to deploy a Ktor backend application with a PostgreSQL database on a Hostinger VPS. The JAR file is built locally, uploaded to the VPS, and configured to run as a service. The guide includes setting up PostgreSQL, configuring Nginx as a reverse proxy, securing the VPS, and testing endpoints with Postman.

## Prerequisites
- **Local Machine**:
  - Windows (or any OS with Gradle and JDK).
  - JDK 20 (Amazon Corretto recommended) or JDK 17 (LTS for production).
  - IntelliJ IDEA (optional, for project management).
  - Git installed.
  - SSH client (e.g., OpenSSH via PowerShell).
- **VPS**:
  - Hostinger VPS with Ubuntu 22.04 (or similar).
  - IP address (e.g., `212.85.27.167`).
  - Root access credentials.
- **Project**:
  - Ktor backend with Exposed ORM, PostgreSQL driver, and JWT authentication.
  - Example `User` data class:
    ```kotlin
    data class User(
        val id: Long? = null,
        val name: String,
        val email: String,
        val phone: String,
        val roles: String,
        val password: String,
        val status: String,
        val firebaseToken: String? = null
    )
    ```
  - `build.gradle.kts` configured with `shadowJar` to produce a fat JAR (e.g., `ktor-app.jar`).
- **Tools**:
  - Postman for testing API endpoints.
  - Terminal (PowerShell for Windows, bash for VPS).

## Step 1: Set Up Local Environment
1. **Install JDK**:
   - Download and install Amazon Corretto 20 (or OpenJDK 17 for production) from [aws.amazon.com/corretto](https://aws.amazon.com/corretto/). You can us your suitable one.
   - Set `JAVA_HOME`:
     ```powershell
     $env:JAVA_HOME = "C:\Program Files\Amazon Corretto\jdk20.0.2_10"
     $env:Path += ";$env:JAVA_HOME\bin"
     ```
     Verify:
     ```powershell
     java -version
     ```

2. **Clone or Open Project**:
   - Navigate to your project directory: (for example)
     ```powershell
     cd C:\Users\YourUserName\IdeaProjects\Locker-Backend
     ```
   - Ensure `build.gradle.kts` includes:
     ```kotlin
     plugins {
         kotlin("jvm")
         id("io.ktor.plugin")
     }

     tasks {
         shadowJar {
             archiveBaseName.set("ktor-app")
             archiveClassifier.set("")
             archiveVersion.set("")
             manifest {
                 attributes["Main-Class"] = application.mainClass.get()
             }
         }
     }
     ```

3. **Build the JAR**:
   - Run:
     ```powershell
     .\gradlew shadowJar
     ```
   - Output: `build/libs/ktor-app.jar`.
   - Test locally:
     ```powershell
     java -jar build/libs/ktor-app.jar
     ```
     If it fails (e.g., PostgreSQL permissions), configure a local PostgreSQL database (see Step 4) or use H2 for testing:
     ```hocon
     database {
         driver = "org.h2.Driver"
         url = "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1"
         user = "sa"
         password = ""
     }
     ```

## Step 2: Set Up Hostinger VPS
1. **Access the VPS**:
   - SSH as `root`:
     ```powershell
     ssh root@212.85.27.167
     ```
     Use the password from Hostinger’s control panel.

2. **Create a Deploy User**:
   ```bash
   adduser deploy
   usermod -aG sudo deploy
   ```
   Set a password for `deploy`.

3. **Install Dependencies**:
   - Update the system:
     ```bash
     sudo apt update && sudo apt upgrade -y
     ```
   - Install Java (OpenJDK 17 recommended):
     ```bash
     sudo apt install openjdk-17-jdk -y
     ```
     Verify:
     ```bash
     java -version
     ```
   - Install Nginx:
     ```bash
     sudo apt install nginx -y
     ```
   - Install UFW:
     ```bash
     sudo apt install ufw -y
     ```

4. **Set Up SSH Key Authentication** (Optional but recommended):
   - Generate SSH key locally:
     ```powershell
     ssh-keygen -t rsa -b 4096
     ```
   - Copy to VPS:
     ```powershell
     ssh-copy-id deploy@212.85.27.167
     ```
   - Disable password authentication:
     ```bash
     sudo nano /etc/ssh/sshd_config
     ```
     Set:
     ```bash
     PasswordAuthentication no
     ```
     Restart SSH:
     ```bash
     sudo systemctl restart sshd
     ```

## Step 3: Configure PostgreSQL on the VPS
1. **Install PostgreSQL**:
   ```bash
   sudo apt install postgresql postgresql-contrib -y
   sudo systemctl enable postgresql
   sudo systemctl start postgresql
   ```

2. **Create Database and User**:
   - Log in as `postgres`:
     ```bash
     sudo -u postgres psql
     ```
   - Create database and user:
     ```sql
     CREATE DATABASE ktor_app;
     CREATE USER ktor_user WITH PASSWORD 'your_password';
     GRANT ALL PRIVILEGES ON DATABASE ktor_app TO ktor_user;
     \c ktor_app
     GRANT USAGE, CREATE ON SCHEMA public TO ktor_user;
     ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO ktor_user;
     \q
     ```

3. **Test Database Connection**:
   ```bash
   psql -h localhost -U ktor_user -d ktor_app
   ```
   Enter `your_password`. Create a test table:
   ```sql
   CREATE TABLE test (id SERIAL PRIMARY KEY);
   ```

## Step 4: Upload the JAR to the VPS
1. **Create Directory**:
   - On the VPS:
     ```bash
     mkdir -p /home/deploy/locker
     chown deploy:deploy /home/deploy/locker
     ```

2. **Transfer the JAR**:
   - From your local machine:
     ```powershell
     cd C:\Users\mdnra\IdeaProjects\Locker-Backend
     scp build/libs/ktor-app.jar deploy@212.85.27.167:~/locker/
     ```

3. **Verify Transfer**:
   - On the VPS:
     ```bash
     ls ~/locker
     ```
     Should show `ktor-app.jar`.

## Step 5: Configure the Ktor Application
1. **Ensure `application.conf`**:
   - Your `application.conf` (in `src/main/resources`) should match the VPS PostgreSQL setup:
     ```hocon
     database {
         driver = "org.postgresql.Driver"
         url = "jdbc:postgresql://localhost:5432/ktor_app"
         user = "ktor_user"
         password = "your_password"
     }
     ```
   - Rebuild if updated:
     ```powershell
     .\gradlew shadowJar
     scp build/libs/ktor-app.jar deploy@212.85.27.167:~/locker/
     ```

2. **Test the JAR**:
   - On the VPS:
     ```bash
     cd ~/locker
     java -jar ktor-app.jar
     ```
   - If it fails with `permission denied for schema public`, re-grant permissions (see Step 3.2).

## Step 6: Set Up Systemd Service
1. **Create Service File**:
   ```bash
   sudo nano /etc/systemd/system/ktor.service
   ```
   Add:
   ```ini
   [Unit]
   Description=Ktor Backend Service
   After=network.target

   [Service]
   User=deploy
   WorkingDirectory=/home/deploy/locker
   ExecStart=/usr/bin/java -jar /home/deploy/locker/ktor-app.jar
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```

2. **Enable and Start**:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable ktor
   sudo systemctl start ktor
   ```

3. **Verify**:
   ```bash
   sudo systemctl status ktor
   ```

## Step 7: Configure Nginx
1. **Create Nginx Config**:
   ```bash
   sudo nano /etc/nginx/sites-available/ktor
   ```
   Add:
   ```nginx
   server {
       listen 80;
       server_name 212.85.27.167;

       location / {
           proxy_pass http://localhost:8080;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```

2. **Enable Config**:
   ```bash
   sudo ln -s /etc/nginx/sites-available/ktor /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl restart nginx
   ```

## Step 8: Secure the VPS
1. **Configure Firewall**:
   ```bash
   sudo ufw allow 22  # SSH
   sudo ufw allow 80  # HTTP
   sudo ufw allow 443 # HTTPS
   sudo ufw enable
   sudo ufw status
   ```

2. **Disable Root SSH**:
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
   Set:
   ```bash
   PermitRootLogin no
   ```
   Restart:
   ```bash
   sudo systemctl restart sshd
   ```

3. **Add SSL (Optional)**:
   - If you have a domain:
     ```bash
     sudo apt install certbot python3-certbot-nginx -y
     sudo certbot --nginx -d yourdomain.com
     ```

## Step 9: Test with Postman
1. **Install Postman**:
   - Download from [postman.com](https://www.postman.com/downloads/).

2. **Create a Collection**:
   - Name: “Locker Backend API”.
   - Add requests for endpoints.

3. **Test Endpoints**:
   - **POST /users** (Create User):
     - Method: `POST`
     - URL: `http://212.85.27.167/users`
     - Headers: `Content-Type: application/json`
     - Body:
       ```json
       {
         "name": "Alice Smith",
         "email": "alice@example.com",
         "phone": "5551234567",
         "roles": "admin",
         "password": "adminpass123",
         "status": "active",
         "firebaseToken": null
       }
       ```
     - Expected: `201 Created`, user object with `id`.
   - **GET /users** (List Users):
     - Method: `GET`
     - URL: `http://212.85.27.167/users`
     - Headers: `Authorization: Bearer <jwt_token>` (if required)
     - Expected: `200 OK`, array of users.
   - **POST /login** (Authenticate):
     - Method: `POST`
     - URL: `http://212.85.27.167/login`
     - Headers: `Content-Type: application/json`
     - Body:
       ```json
       {
         "email": "alice@example.com",
         "password": "adminpass123"
       }
       ```
     - Expected: `200 OK`, `{ "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." }`.

4. **Use Environment**:
   - Create a Postman environment:
     - Variable: `base_url`
     - Value: `http://212.85.27.167`
   - Use `{{base_url}}/users` in requests.

## Step 10: Monitor and Maintain
1. **Monitor Logs**:
   ```bash
   sudo journalctl -u ktor -f
   sudo tail -f /var/log/nginx/access.log
   ```

2. **Backup Database**:
   ```bash
   pg_dump -U ktor_user ktor_app > backup.sql
   ```

3. **Update Application**:
   - Rebuild and upload:
     ```powershell
     .\gradlew shadowJar
     scp build/libs/ktor-app.jar deploy@212.85.27.167:~/locker/
     ```
   - Restart:
     ```bash
     sudo systemctl restart ktor
     ```

## Troubleshooting
- **PostgreSQL Permissions**:
  ```bash
  sudo -u postgres psql
  \c ktor_app
  GRANT ALL ON SCHEMA public TO ktor_user;
  ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO ktor_user;
  ```
- **Nginx Errors**:
  ```bash
  sudo nginx -t
  sudo tail -f /var/log/nginx/error.log
  ```
- **Service Fails**:
  ```bash
  sudo systemctl status ktor
  sudo journalctl -u ktor
  ```
- **Postman 404**:
  - Check routing in `Application.kt`.
- **Postman 401**:
  - Ensure valid JWT token from `/login`.

## Recommendations
- **Use JDK 17**:
  ```kotlin
  kotlin {
      jvmToolchain(17)
  }
  ```
  ```bash
  sudo apt install openjdk-17-jdk -y
  ```
- **Environment Variables**:
  ```hocon
  database {
      driver = "org.postgresql.Driver"
      url = ${?DB_URL}
      user = ${?DB_USER}
      password = ${?DB_PASSWORD}
  }
  ```
  Update `systemd`:
  ```ini
  Environment="DB_URL=jdbc:postgresql://localhost:5432/ktor_app"
  Environment="DB_USER=ktor_user"
  Environment="DB_PASSWORD=your_password"
  ```
- **Regular Updates**:
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```
