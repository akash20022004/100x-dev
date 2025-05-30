# What is CI and CD?

## Continuous Integration (CI)

Continuous Integration (CI) is a development practice where developers frequently integrate their code changes into a shared repository, preferably several times a day. Each integration is automatically verified by the monorepo we’re dealing with today.

[Monorepo Link](https://github.com/100xdevs-cohort-2/week-18-2-ci-cd)

This monorepo contains three applications:

- `bank-webhook`
- `merchant-app`
- `user-app`

We’ll be deploying all three to the same EC2 instance.

## Building the Project and Running Automated Tests

This process allows teams to detect problems early, improve software quality, and reduce the time it takes to validate and release new software updates.

## Continuous Deployment (CD)

As the name suggests, CD involves deploying your code continuously to various environments (dev/stage/prod).

### Continuous Deployment in GitHub

We’ll be deploying a Next.js app to EC2 servers via Docker.

💡 **Note:** You don’t necessarily need Docker when deploying to a simple EC2 server. However, if you deploy to:

1. GCP App Runner
2. ECS
3. Kubernetes

Then it makes more sense to deploy a Dockerized application.

### Architecture Diagram

![Architecture Diagram](./one.webp)

💡 **Tip:** The last step of deployment varies based on where you’re pushing your image.

## How to Create a CI/CD Pipeline?

For GitHub, you can add all your pipelines to `.github/workflows`.

Example: [GitHub Workflow Example](https://github.com/code100x/cms/blob/main/.github/workflows/lint.yml)

### CI Pipeline Example

![CD Pipeline](./two.webp)

### CI pipelines look like this finally

![CD Pipeline](./three.webp)

💡 **Hint:** Use [Online YAML to JSON Converter](https://onlineyamltools.com/convert-yaml-to-json) to visualize the pipeline in JSON format.

# Continuous Deployment (CD) Pipeline Setup

## Step 1 - Create the CD Pipeline

Make sure that whenever someone tries to create a **PR**, we build the project and make sure that it builds as expected.

![p](./four.webp)

## 🚀 Setting Up the CD Pipeline

### 1️⃣ Fork the Main Repository

Fork the repository from: [100xdevs-cohort-2/week-18-2-ci-cd](https://github.com/100xdevs-cohort-2/week-18-2-ci-cd)

### 2️⃣ Add a GitHub Actions Workflow File

Create a new file at **`.github/workflows/build.yml`** in the root directory of the repository.

### 3️⃣ Create the Workflow Configuration

Add the following YAML code to `build.yml`:

```yaml
ame: Build on PR

on:
  pull_request:
    branches:
      - master

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Use Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "20"

      - name: Install Dependencies
        run: npm install

      - name: Run Build
        run: npm run build
```

### 4️⃣ Push Changes to `master` Branch

Commit and push the `build.yml` file to the `master` branch.

### 5️⃣ Create a New Branch & Open a Pull Request

1. Create a new branch and make minimal changes.
2. Open a **Pull Request (PR)**.
3. The GitHub Actions workflow should automatically run.

### ✅ Expected Behavior

- If the build succeeds, the PR is **valid** and can be merged.
- If the build fails, the PR will show an **error**, preventing broken code from being merged.

![](./5.webp)

---

## 🚀 Let’s Add a Deploy Step

### 1️⃣ Create Dockerfiles for Your Apps

Create the necessary Dockerfile for your user application:

#### `docker/Dockerfile.user`

```dockerfile
FROM node:20.12.0-alpine3.19

WORKDIR /usr/src/app

COPY package.json package-lock.json turbo.json tsconfig.json ./
COPY apps ./apps
COPY packages ./packages

# Install dependencies
RUN npm install

# Generate Prisma client
RUN cd packages/db && npx prisma generate && cd ../..

# Build only the user app
RUN npm run build

CMD ["npm", "run", "start-user-app"]
```

### 2️⃣ Add `start-user-app` Script

Modify the root `package.json` to include:

```json
"scripts": {
  "start-user-app": "cd ./apps/user-app && npm run start"
}
```

💡 **Optimization Tip:**
You dont really need to build every app for every dockerfile. Can you change the build command so that only a single app is built for each dockerfile?

![](./6.webp)

---

## 📦 Create the CD Pipeline

This pipeline will:

1. Clone the repository
2. Build the Docker image
3. Push the image to Docker Hub

### 3️⃣ Create `.github/workflows/deploy.yml`

```yaml
name: Build and Deploy to Docker Hub

on:
  push:
    branches:
      - master

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Check Out Repo
        uses: actions/checkout@v2

      - name: Log in to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v2
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: 100xdevs/web-app:latest # Replace with your Docker Hub username and repository

      - name: Verify Pushed Image
        run: docker pull 100xdevs/web-app:latest # Replace with your Docker Hub username and repository
```

### 4️⃣ Add Secrets to GitHub

Ensure the following secrets are added to your repository:

- **DOCKER_USERNAME** (your Docker Hub username)
- **DOCKER_PASSWORD** (your Docker Hub password/token)

### 5️⃣ Verify Deployment

1. Push your changes to `master`.
2. Check the GitHub Actions workflow.

![](./7.webp)

![](./8.webp)

3. Visit **Docker Hub** to confirm the image has been uploaded.

![](./9.webp)

💡 **Note:** You may need to inject additional environment variables (like `DB_URL`) for the build to function correctly.

---

# Deployment Guide

## Let’s Pull the Docker Image

Reference: [SSH Action by Appleboy](https://github.com/appleboy/ssh-action)

### Steps to Deploy

#### 1. Create an EC2 Server
- Download its keypair file.
- Allow HTTP/HTTPS traffic.
- Use an Ubuntu base image.

#### 2. Install Docker on the Machine
Follow the official guide to install Docker on Ubuntu:
[Docker Installation Guide](https://docs.docker.com/engine/install/ubuntu/)

```sh
sudo docker run hello-world
```

#### 3. Update Workflow to Pull the Latest Image on EC2

```yaml
name: Build and Deploy to Docker Hub

on:
  push:
    branches:
      - master  # Trigger on pushes to master

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - name: Check Out Repo
      uses: actions/checkout@v2

    - name: Prepare Dockerfile
      run: cp ./docker/Dockerfile.user ./Dockerfile

    - name: Log in to Docker Hub
      uses: docker/login-action@v1
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and Push Docker image
      uses: docker/build-push-action@v2
      with:
        context: .
        file: ./Dockerfile
        push: true
        tags: 100xdevs/web-app:latest

    - name: Verify Pushed Image
      run: docker pull 100xdevs/web-app:latest

    - name: Deploy to EC2
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.SSH_HOST }}
        username: ${{ secrets.SSH_USERNAME }}
        key: ${{ secrets.SSH_KEY }}
        script: |
          sudo docker pull 100xdevs/web-app:latest
          sudo docker stop web-app || true
          sudo docker rm web-app || true
          sudo docker run -d --name web-app -p 3005:3000 100xdevs/web-app:latest
```

#### 4. Point Domain to Server IP
Configure `userapp.your_domain.com` to point to the server IP.

#### 5. Add nginx reverse proxy to forward requests from userapp.your_domain.com to port on which the app is running

```nginx
server {
        server_name userapp.100xdevs.com;

        location / {
            proxy_pass http://localhost:3005;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;

            # Basic Authentication
            auth_basic "Restricted Content";
            auth_basic_user_file /etc/nginx/.htpasswd;
        }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/userapp.100xdevs.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/userapp.100xdevs.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}
```

#### 6. Install Certbot and Refresh Certificate

```sh
sudo certbot --nginx
```

### Take-Home Assignments
- Get a database on **Neon.tech / RDS / Aiven** and add a DB migration step.
- Pass in the DB credentials while starting the Docker image.
- Start the Docker image so that it restarts if it goes down (similar to PM2).

## CI/CD
This setup ensures automated deployment and security using Docker, EC2, and Nginx with SSL.


