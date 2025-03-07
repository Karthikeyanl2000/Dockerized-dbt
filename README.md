# GitHub Webhook Documentation

## Overview
This documentation explains how to set up and use a Flask-based GitHub Webhook listener to automate project updates upon push events.

## Code
```python
from flask import Flask, request, jsonify
import hmac
import hashlib
import subprocess
import os
import logging

# Set up logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger('github-webhook')

app = Flask(__name__)

WEBHOOK_SECRET = "your_webhook_secret"  
PROJECT_DIR = "/home/gotoz/simple-python-project" 

@app.route('/webhook', methods=['POST'])
def webhook():
    event_type = request.headers.get('X-GitHub-Event')
    delivery_id = request.headers.get('X-GitHub-Delivery')
    signature = request.headers.get('X-Hub-Signature-256')
    
    logger.info(f"Received {event_type} event (ID: {delivery_id})")
    
    if WEBHOOK_SECRET:
        if not signature:
            logger.warning("No signature provided but webhook secret is configured")
            return jsonify({"error": "No signature provided"}), 401
        
        calculated_signature = 'sha256=' + hmac.new(
            WEBHOOK_SECRET.encode(),
            request.data,
            hashlib.sha256
        ).hexdigest()
        
        if not hmac.compare_digest(calculated_signature, signature):
            logger.warning("Invalid signature")
            return jsonify({"error": "Invalid signature"}), 401
    
    payload = request.json
    
    # Handle push events
    if event_type == 'push':
        try:
            repo_name = payload['repository']['name']
            
            branch = payload['ref'].replace('refs/heads/', '')
            commits = payload['commits']
            
            logger.info(f"Push to {repo_name}/{branch} with {len(commits)} commits")
            logger.info(f"Repository: {repo_name}")
            
            if branch == 'main':  
                logger.info(f"Processing push to {branch} branch")
                
                # Check if directory exists
                if not os.path.exists(PROJECT_DIR):
                    logger.error(f"Project directory {PROJECT_DIR} does not exist")
                    return jsonify({"status": "error", "message": "Project directory not found"}), 500
                
                # Execute git commands
                logger.info("Pulling latest changes")
                try:
                    result = subprocess.run(
                        ['git', 'pull', 'origin', branch],
                        cwd=PROJECT_DIR,
                        check=True,
                        capture_output=True,
                        text=True
                    )
                    logger.info(f"Git pull output: {result.stdout}")
                    
                    return jsonify({
                        "status": "success",
                        "message": f"Successfully pulled changes from {branch}"
                    }), 200
                    
                except subprocess.CalledProcessError as e:
                    logger.error(f"Git operation failed: {e}")
                    logger.error(f"Error output: {e.stderr}")
                    return jsonify({
                        "status": "error", 
                        "message": f"Git operation failed: {e.stderr}"
                    }), 500
            else:
                logger.info(f"Ignoring push to non-target branch: {branch}")
                return jsonify({"status": "ignored", "message": f"Branch {branch} not configured for updates"}), 200
                
        except KeyError as e:
            logger.error(f"Missing field in payload: {e}")
            return jsonify({"status": "error", "message": f"Invalid payload format: {e}"}), 400
            
        except Exception as e:
            logger.exception("Unexpected error")
            return jsonify({"status": "error", "message": str(e)}), 500
    
    else:
        logger.info(f"Received {event_type} event - no action needed")
        return jsonify({"status": "received", "message": f"Received {event_type} event"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

## Requirements
- Python 3
- Flask
- git installed on the server

## Configuration
- `WEBHOOK_SECRET`: Set this to the secret configured in your GitHub webhook settings.
- `PROJECT_DIR`: Set this to the absolute path of your local project repository (directory).
- **Branch Handling**: Modify the script to define which branches trigger updates (e.g., change 'main' if needed).

## Running the Server
Run the Flask application:
```sh
python webhook.py
```

To run the webhook as a background service using PM2:

### Using PM2
```sh
npm install pm2 -g
pm2 start webhook.py --interpreter python3
pm2 save
pm2 startup
```

## Setting Up GitHub Webhook
1. Go to your GitHub repository settings.
2. Navigate to **Webhooks** → **Add webhook**.
3. Set **Payload URL** to your server's public URL (e.g., `https://yourserver.com/webhook`).
4. Choose **Content type**: `application/json`.
5. Add your secret key (same as `WEBHOOK_SECRET`).
6. Select **Just the push event**.
7. Click **Add webhook**.

## Testing the Webhook
To verify that the webhook is working correctly:
1. Push a change to the configured branch (e.g., `git push origin main`).
2. Check the webhook logs using PM2: 
   ```sh
   pm2 logs webhook
   ```
3. Verify that the repository (project directory) updates as expected.

---

# Dockerized dbt Setup

## Overview
This setup allows you to containerize a **dbt** (Data Build Tool) project with SQL Server support using **Docker** and **Docker Compose**. The setup includes:

- A **Dockerfile** that installs the required dependencies, including the **Microsoft ODBC driver** for SQL Server.
- A **docker-compose.yml** file to manage the dbt container, volumes, and environment variables.
- The ability to **automate dbt commands** like `dbt deps`, `dbt build`, and `dbt debug`.

---

## Prerequisites
Ensure you have the following installed on your system:
- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)
- A `dbt_project` directory with a valid `profiles.yml` file.
- **Tested on Python 3.10-slim-buster** base image.

---

## Project Structure
```
project-root/
│── dbt_project/
│   ├── models/
│   ├── seeds/
│   ├── dbt_packages/
│   ├── profiles/
│   │   ├── profiles.yml  # dbt profiles configuration
│   ├── test.py  # Test script to validate setup
│── Dockerfile
│── docker-compose.yml
```

---

## Dockerfile Explanation
The `Dockerfile` is responsible for setting up the dbt environment within a container.

### Steps in the Dockerfile

1. **Use Python 3.10 Slim as the Base Image**
   ```dockerfile
   FROM python:3.10-slim-buster
   ```

2. **Install Required System Dependencies**
   ```dockerfile
   RUN apt-get update \
       && apt-get install -y --no-install-recommends \
       unixodbc \
       unixodbc-dev \
       gcc \
       g++ \
       libpq-dev \
       curl \
       gnupg2 \
       ca-certificates \
       git \
       && apt-get clean
   ```

3. **Install Microsoft SQL Server ODBC Driver**
   ```dockerfile
   RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - \
       && curl https://packages.microsoft.com/config/debian/10/prod.list > /etc/apt/sources.list.d/mssql-release.list \
       && apt-get update \
       && ACCEPT_EULA=Y apt-get install -y msodbcsql18 \
       && apt-get clean
   ```

4. **Copy the dbt project and configuration files**
   ```dockerfile
   COPY dbt_project /usr/src/dbt/dbt_project
   COPY dbt_project/profiles/profiles.yml /root/.dbt/profiles.yml
   ```

5. **Set up the working directory and install dbt**
   ```dockerfile
   WORKDIR /usr/src/dbt/dbt_project
   RUN pip install --upgrade pip
   RUN pip install dbt-sqlserver
   ```

6. **Define the startup command**
   ```dockerfile
   CMD dbt deps && dbt build --profiles-dir profiles
   ```

---

## Docker Compose Explanation
The `docker-compose.yml` file defines the dbt service, volumes, and runtime configuration.

### Key Components

1. **Define the dbt service**
   ```yaml
   services:
     dbt:
       build:
         context: .
         dockerfile: Dockerfile
   ```

2. **Mount local files into the container**
   ```yaml
   volumes:
     - ./dbt_project:/usr/app/dbt/dbt_project
     - ./logs:/usr/app/dbt/logs
     - dbt-cache:/usr/app/dbt/dbt_project/target
   ```

3. **Set environment variables**
   ```yaml
   environment:
     - DBT_PROFILES_DIR=/root/.dbt
   ```

4. **Custom startup script in `command`**
   ```yaml
   command: >
     bash -c "cd /usr/app/dbt/dbt_project &&
             echo 'Run test.py file...' &&
             python test.py &&
             echo 'Removing existing dbt_packages directory...' &&
             rm -rf dbt_packages &&
             echo 'Running dbt deps...' &&
             dbt deps &&
             echo 'Running dbt debug...' &&
             dbt debug &&
             echo 'Running dbt build...' &&
             dbt build &&
             echo 'All dbt commands completed. Container will remain running...' &&
             tail -f /dev/null"
   ```

5. **Define a volume for dbt cache**
   ```yaml
   volumes:
     dbt-cache:
   ```

---

## Running the Project

### Step 1: Build the Docker Image
```bash
docker compose build
```

### Step 2: Start the dbt Container in detached mode
```bash
docker compose up -d
```

### Step 3: Check Logs
```bash
docker compose logs -f
```

---


# DBT Docker Compose Scheduler with PM2

This guide explains how to schedule `docker compose up` for a specific `docker-compose.yml` using `pm2`.

## Prerequisites
- [Docker](https://docs.docker.com/get-docker/) installed
- [PM2](https://pm2.keymetrics.io/) installed globally

## Steps to Automate `docker compose up`

### 1. Create a Script
Create a script file named `start_docker.sh`:

```sh
#!/bin/bash
cd <PROJECT_PATH>
docker compose -f docker-compose.yml up -d
```
Replace `<PROJECT_PATH>` with the actual path of your project directory.

Make the script executable:
```sh
chmod +x start_docker.sh
```

### 2. Configure PM2 for Multiple Schedules
Create a `config.js` file to define multiple schedules:

```js
module.exports = {
  apps: [
    {
      name: "docker-compose",
      script: "./start_docker.sh",
      cron_restart: "0 0 * * *", 
      autorestart: false,
    },
    {
      name: "docker-compose-evening",
      script: "./start_docker.sh",
      cron_restart: "0 23 * * *", 
      autorestart: false,
    },
  ],
};
```

### 3. Start PM2 with the Config File
Run the following command to apply the schedule:

```sh
pm2 start config.js
```

### 4. Ensure PM2 Runs on System Restart
Save the PM2 process list and enable it to start on boot:
```sh
pm2 save
pm2 startup
```

### 5. Verify the Scheduled Task
Check scheduled tasks with:
```sh
pm2 list
pm2 show docker-compose
pm2 show docker-compose-evening
```

This setup ensures that `docker compose up -d` runs automatically at the scheduled times using `pm2`. 

---

## Architectural Diagram
```
+--------------------------------------------------+
|                 Host Machine                     |
|                                                  |
|   +----------------------+                       |
|   | Docker Engine        |                       |
|   +----------------------+                       |
|           |                                      |
|   +--------------------+                         |
|   | dbt Container      |                         |
|   |--------------------|                         |
|   | Python 3.10        |                         |
|   | dbt-sqlserver      |                         |
|   | ODBC Driver        |                         |
|   | /dbt_project       | <----+                  |
|   | /root/.dbt         |       |                 |
|   | Logs               |       | Mounted         |
|   +--------------------+       | Volumes         |
|                                |                 |
|   +----------------------+     |                 |
|   | Local File System    | <---+                 |
|   |----------------------|                       |
|   | dbt_project/         |                       |
|   | profiles.yml         |                       |
|   | logs/                |                       |
|   | Webhook Integrated   | <--> API/Webhooks     |
|   +----------------------+                       |
|                                                  |
|   +----------------------+                       |
|   | PM2 Scheduler        |                       |
|   |----------------------|                       |
|   | Runs docker-compose  |                       |
|   | at scheduled times   |                       |
|   +----------------------+                       |
+--------------------------------------------------+


```

---


