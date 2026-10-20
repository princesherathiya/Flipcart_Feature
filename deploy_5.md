# Practical: Dockerize a Static Portfolio and Deploy to AWS EC2 Using Jenkins and GitHub

## Aim
To containerize a static web portfolio using Docker, create a Docker image and container, automate build and deployment using Jenkins, store the project on GitHub, and deploy the Dockerized application on **AWS EC2**.

> This practical deploys to AWS EC2, **not Render**.

## Architecture

Developer pushes code → GitHub → Jenkins checks out the code → runs a basic test → builds a Docker image → pushes the image to Docker Hub → connects to EC2 over SSH → pulls the image and runs the container.

## Requirements
- Windows PC running Jenkins (this guide uses Windows-compatible Jenkins Pipeline commands).
- Git and GitHub CLI.
- Docker Desktop on Windows for local image/container testing.
- GitHub account.
- Docker Hub account.
- AWS account and an Ubuntu EC2 instance.
- Jenkins credentials for Docker Hub and EC2 SSH access.

**Cost and security:** AWS EC2, public IPv4 addresses, storage, and network usage can incur charges. Review current pricing and clean up resources afterward. Never commit passwords, tokens, private keys, or `.pem` files. Do not expose Jenkins port 8080 directly to the public internet without secure HTTPS and access controls.

---

## 1. Install required tools on Windows

Open PowerShell. After installing a tool, reopen PowerShell before verifying it.

### 1.1 Install Git

```powershell
winget install --id Git.Git -e
```

Verify:

```powershell
git --version
```

### 1.2 Install GitHub CLI

```powershell
winget install --id GitHub.cli -e
```

Verify and authenticate:

```powershell
gh --version
gh auth login
gh auth status
gh auth setup-git
```

In the prompts, choose GitHub.com and HTTPS, then finish authentication in the browser.

### 1.3 Install Java for Jenkins

Check:

```powershell
java -version
```

If Java is missing, install a JDK version supported by your Jenkins release. Example for JDK 21:

```powershell
winget install --id Microsoft.OpenJDK.21 -e
```

Official Java support policy: https://www.jenkins.io/doc/book/platform-information/support-policy-java/

### 1.4 Install Jenkins

Download the official Windows installer:
https://www.jenkins.io/download/

Install Jenkins as a Windows service. Open Jenkins at:

```text
http://localhost:8080
```

Unlock Jenkins using the initial administrator password, install suggested plugins, and create an administrator account. Keep the password private.

### 1.5 Install Docker Desktop

```powershell
winget install --id Docker.DockerDesktop -e
```

Start Docker Desktop, complete any required setup, and wait until the Docker engine is running. If prompted, enable the WSL 2 backend and restart Windows if required.

Verify:

```powershell
docker --version
docker info
```

If `docker info` cannot connect to the engine, start Docker Desktop and wait for it to finish starting.

### 1.6 Install OpenSSH client

Open PowerShell as Administrator:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

Verify in a new PowerShell window:

```powershell
ssh -V
scp
```

---

## 2. Create a simple static portfolio

In PowerShell:

```powershell
mkdir "$HOME\docker-portfolio-ec2"
cd "$HOME\docker-portfolio-ec2"
```

Create `index.html`:

```powershell
@'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>My Docker Portfolio</title>
  <style>
    body { font-family: Arial, sans-serif; max-width: 760px; margin: 60px auto; padding: 0 20px; line-height: 1.6; }
    h1 { margin-bottom: 8px; }
    .card { padding: 20px; border: 1px solid #ddd; border-radius: 12px; }
  </style>
</head>
<body>
  <h1>My Portfolio</h1>
  <p>Static portfolio deployed with Docker, Jenkins, GitHub, and AWS EC2.</p>
  <div class="card">
    <h2>About Me</h2>
    <p>I am learning DevOps and CI/CD automation.</p>
    <h2>Projects</h2>
    <p>Project 1: Dockerized static website.</p>
  </div>
  <p id="version">Version 1</p>
</body>
</html>
'@ | Set-Content -Encoding utf8 index.html
```

### 2.1 Create a Dockerfile

Create a file named exactly `Dockerfile` (no extension):

```powershell
@'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
'@ | Set-Content -Encoding ascii Dockerfile
```

### 2.2 Create `.dockerignore`

```powershell
@'
.git
.gitignore
*.pem
.env
'@ | Set-Content .dockerignore
```

### 2.3 Create a basic test script

Create `test.ps1`:

```powershell
@'
$ErrorActionPreference = "Stop"
if (-not (Test-Path "index.html")) {
    throw "index.html is missing"
}
$content = Get-Content "index.html" -Raw
if ($content -notmatch "My Portfolio") {
    throw "Portfolio heading not found"
}
Write-Host "Portfolio test passed."
'@ | Set-Content -Encoding utf8 test.ps1
```

---

## 3. Build and run the Docker image locally

Run in PowerShell from the project folder:

```powershell
docker build -t docker-portfolio:local .
docker image ls
docker run -d --name portfolio-local -p 8081:80 docker-portfolio:local
docker ps
```

Open the local site:

```text
http://localhost:8081
```

To stop and remove the local container when finished:

```powershell
docker rm -f portfolio-local
```

---

## 4. Create a GitHub repository and push the project

Authenticate with GitHub first if you have not already:

```powershell
gh auth status
```

Initialize and push:

```powershell
git init
git branch -M main
git add index.html Dockerfile .dockerignore test.ps1
git commit -m "Create Dockerized portfolio"
gh repo create docker-portfolio-ec2 --public --source=. --remote=origin --push
```

If the repository already exists, use its actual URL:

```powershell
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git
git push -u origin main
```

If `origin` already exists, inspect it with `git remote -v` rather than adding it twice.

---

## 5. Create an AWS EC2 Ubuntu instance

1. Open AWS Console → EC2 → Launch instance.
2. Choose a current supported Ubuntu Server LTS image.
3. Choose a learning-sized instance and review pricing/eligibility for your account.
4. Create/select an SSH key pair and download the `.pem` file. Keep it private.
5. Configure the security group:
   - SSH TCP 22: restrict source to your current public IP and/or the Jenkins host's fixed public IP.
   - HTTP TCP 80: allow `0.0.0.0/0` if the portfolio should be publicly accessible.
   - Do not open extra ports unnecessarily.
6. Launch and wait for the instance to pass status checks.
7. Copy the EC2 **Public IPv4 address**.

Connect from Windows PowerShell, replacing the key path and address:

```powershell
ssh -i "$HOME\Downloads\my-ec2-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

### 5.1 Install Docker Engine on EC2

Run the following commands **inside the EC2 SSH session**. These use Ubuntu's packaged Docker engine and CLI:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo docker --version
sudo systemctl status docker --no-pager
```

Do not add the `ubuntu` user to the Docker group unless you understand that Docker group membership effectively grants root-level control. The deployment below uses `sudo docker`.

Exit the EC2 session when done:

```bash
exit
```

---

## 6. Create a Docker Hub repository

1. Sign in at https://hub.docker.com/
2. Create a repository, for example `docker-portfolio-ec2`.
3. Note your Docker Hub username and repository name.
4. Create a Docker Hub access token with the minimum required permissions for pushing images. Store it securely; do not use your account password in source code.

The image name will be:

```text
YOUR_DOCKERHUB_USERNAME/docker-portfolio-ec2:latest
```

---

## 7. Configure Jenkins plugins and credentials

Open `http://localhost:8080`.

### 7.1 Plugins

Open **Manage Jenkins → Plugins** and make sure the following are installed:
- Pipeline
- Git
- Credentials Binding (usually included/available with the standard Jenkins setup)

Restart Jenkins if the plugin manager requests it.

### 7.2 Docker Hub credential

Go to **Manage Jenkins → Credentials → (appropriate store/domain) → Add Credentials**:
- Kind: Username with password
- Username: your Docker Hub username
- Password: your Docker Hub access token
- ID: `dockerhub-creds`
- Description: `Docker Hub push credential`

Save it.

### 7.3 EC2 SSH credential

Use a dedicated deployment key where practical. The public key must be authorized for the EC2 `ubuntu` account. If using the EC2 key pair you created, ensure its public key is the one authorized for the instance.

In Jenkins, add:
- Kind: **SSH Username with private key**
- Username: `ubuntu`
- Private Key: paste the private key contents into the credential field
- ID: `ec2-deploy-key`

Save it. Never put the private key in GitHub or the Jenkinsfile.

### 7.4 SSH host key verification

Before the pipeline runs, verify the EC2 SSH host-key fingerprint through a trusted channel and configure it for the Jenkins Windows service account's known-hosts/host-key verification. Do not disable host-key verification or blindly accept unknown host keys. The Jenkins service account must have access to Git, Docker CLI/Desktop engine, `ssh`, and `scp`; a command working in your personal terminal may not work under the Jenkins service account.

---

## 8. Add the Jenkinsfile

Create a file named exactly `Jenkinsfile` in the project root.

**Replace both placeholders**:
- `YOUR_DOCKERHUB_USERNAME`
- `YOUR_EC2_PUBLIC_IP`

```groovy
pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        DOCKER_IMAGE = 'YOUR_DOCKERHUB_USERNAME/docker-portfolio-ec2:latest'
        EC2_HOST = 'YOUR_EC2_PUBLIC_IP'
        CONTAINER_NAME = 'docker-portfolio'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"
                    if (-not (Test-Path "index.html")) {
                        throw "index.html is missing"
                    }
                    $content = Get-Content "index.html" -Raw
                    if ($content -notmatch "My Portfolio") {
                        throw "Portfolio heading not found"
                    }
                    Write-Host "Portfolio test passed."
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    @echo off
                    docker build -t "%DOCKER_IMAGE%" .
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    bat '''
                        @echo off
                        powershell -NoProfile -Command "$env:DOCKERHUB_TOKEN | docker login --username $env:DOCKERHUB_USER --password-stdin"
                        if errorlevel 1 exit /b 1
                        docker push "%DOCKER_IMAGE%"
                        if errorlevel 1 exit /b 1
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ec2-deploy-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    bat '''
                        @echo off
                        ssh -i "%SSH_KEY%" -o BatchMode=yes %SSH_USER%@%EC2_HOST% "sudo docker pull %DOCKER_IMAGE%"
                        if errorlevel 1 exit /b 1

                        ssh -i "%SSH_KEY%" -o BatchMode=yes %SSH_USER%@%EC2_HOST% "sudo docker rm -f %CONTAINER_NAME% || true"
                        if errorlevel 1 exit /b 1

                        ssh -i "%SSH_KEY%" -o BatchMode=yes %SSH_USER%@%EC2_HOST% "sudo docker run -d --name %CONTAINER_NAME% --restart unless-stopped -p 80:80 %DOCKER_IMAGE%"
                        if errorlevel 1 exit /b 1

                        ssh -i "%SSH_KEY%" -o BatchMode=yes %SSH_USER%@%EC2_HOST% "sudo docker ps"
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Build, test, Docker image push, and EC2 deployment completed.'
        }
        failure {
            echo 'Pipeline failed. Review the stage logs and fix the reported issue.'
        }
    }
}
```

### Important deployment notes
- Ensure port 80 on EC2 is not already occupied by Nginx or another service. If the Ansible practical installed Nginx on this same EC2 instance, check with `sudo systemctl status nginx` and stop it only if you intend Docker to own port 80: `sudo systemctl disable --now nginx`.
- The example replaces the old container, so the site may be briefly unavailable during deployment. This is acceptable for a classroom demo, not a zero-downtime production rollout.
- For a first run, verify that Jenkins can access the Docker Desktop engine. Docker Desktop installed for a logged-in user may not be available to a Windows Jenkins service account. If it is unavailable, run Jenkins in a properly configured environment with Docker access rather than weakening machine security.
- The example uses `latest` to keep the practical simple. For production, use unique image tags per commit and a controlled rollback strategy.
- If `docker login` or SSH fails because of Windows quoting/environment issues, inspect the exact Jenkins console output and validate the command in the same account/environment Jenkins uses.

---

## 9. Push Jenkinsfile and configure the Jenkins job

In PowerShell from the project directory:

```powershell
git add Jenkinsfile
git commit -m "Add Jenkins Docker EC2 pipeline"
git push origin main
```

In Jenkins:
1. Select **New Item**.
2. Name it `docker-portfolio-ec2`.
3. Choose **Pipeline** → **OK**.
4. Under Pipeline, set **Definition** to `Pipeline script from SCM`.
5. SCM: `Git`.
6. Repository URL: `https://github.com/YOUR_GITHUB_USERNAME/docker-portfolio-ec2.git`
7. Branch Specifier: `*/main`.
8. Script Path: `Jenkinsfile`.
9. Save and click **Build Now**.
10. Open **Console Output**. Checkout, Test, Build Docker Image, Push Image, and Deploy to EC2 should pass.

For a private GitHub repository, add a GitHub credential and select it in the job configuration.

---

## 10. Enable automatic builds when code is pushed

### Option A — Poll SCM (simple for a local Jenkins)

Open the Jenkins job → **Configure → Build Triggers → Poll SCM** and enter:

```text
H/2 * * * *
```

Save. Jenkins checks approximately every two minutes; this is not an immediate webhook trigger.

### Option B — GitHub webhook

GitHub cannot reach `http://localhost:8080` on your private PC. A webhook requires Jenkins to have a stable, publicly reachable HTTPS endpoint configured securely. Do not expose an unsecured Jenkins instance directly to the internet. If secure public HTTPS access already exists, configure a GitHub webhook to:

```text
https://YOUR_SECURE_JENKINS_HOST/github-webhook/
```

Use the GitHub push event and enable the appropriate GitHub hook trigger in Jenkins. For a local classroom setup, Poll SCM is simpler.

---

## 11. Test the complete pipeline

Change `Version 1` in `index.html` to `Version 2`, then run:

```powershell
git add index.html
git commit -m "Update portfolio to version 2"
git push origin main
```

Wait for the next Poll SCM check, or use the configured webhook. Inspect the latest Jenkins Console Output. Then open:

```text
http://YOUR_EC2_PUBLIC_IP
```

The page should show Version 2.

You can also verify the running container from an SSH session:

```bash
sudo docker ps
sudo docker logs docker-portfolio
```

---

## Troubleshooting

| Problem | What to check |
|---|---|
| Docker command cannot connect to engine | Start Docker Desktop and verify `docker info`. Check the Jenkins service account's access to the Docker engine. |
| Jenkins cannot find `docker`, `git`, `ssh`, or `scp` | Configure the Windows service account's PATH and restart Jenkins. |
| Docker Hub login/push fails | Verify `dockerhub-creds`, username, access token permissions, and repository name. |
| SSH `Permission denied (publickey)` | Check `ec2-deploy-key`, username `ubuntu`, and the public key authorized on EC2. |
| SSH host-key check fails | Verify the EC2 host fingerprint and configure the known-hosts/host-key verification for the Jenkins service account. |
| EC2 SSH times out | Check instance status, public IP, routing, and security-group TCP 22 source. |
| Website cannot open | Check inbound TCP 80, `sudo docker ps`, and `sudo docker logs docker-portfolio` on EC2. |
| Port 80 already in use | Check for Nginx or another container/service; stop or reconfigure only the service you intend to replace. |
| Push does not trigger a build | Check Poll SCM, branch `*/main`, repository URL, and that the commit was pushed to `main`. |

## Result
A static portfolio is stored on GitHub, tested and built into a Docker image by Jenkins, pushed to Docker Hub, and deployed as a running Docker container on AWS EC2. New GitHub changes trigger the Jenkins pipeline through Poll SCM or a properly secured webhook.

## Cleanup
After the practical, stop or terminate unused EC2 resources, review security-group rules, remove unused tokens/credentials, and delete local test containers/images if no longer needed. Consider ongoing AWS costs before leaving the instance running.
