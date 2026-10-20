# Practical: GitHub CLI + Jenkins + Docker + AWS EC2
## Full beginner guide with installation commands

> Run Windows commands in **PowerShell as your normal Windows user**, unless a step says Administrator. Run Linux commands after connecting to the Ubuntu EC2 instance. Replace every placeholder such as `YOUR_GITHUB_USERNAME`, `YOUR_DOCKERHUB_USERNAME`, `YOUR_EC2_PUBLIC_IP`, and `YOUR_KEY.pem` with your real values. Never commit passwords, access tokens, or `.pem` keys to GitHub.

## 1. Install Git for Windows

Open **PowerShell**. If `winget` is available, run:

```powershell
winget install --id Git.Git -e
```

Close and reopen PowerShell, then verify:

```powershell
git --version
```

Official download if `winget` is unavailable: https://git-scm.com/download/win

Set your commit identity:

```powershell
git config --global user.name "Your Name"
git config --global user.email "YOUR_GITHUB_EMAIL"
```

## 2. Install GitHub CLI (`gh`)

Run in PowerShell:

```powershell
winget install --id GitHub.cli -e
```

Close and reopen PowerShell, then verify:

```powershell
gh --version
```

If `winget` is unavailable, install from https://cli.github.com/.

## 3. Log in to GitHub CLI

```powershell
gh auth login
```

Choose:
- GitHub.com
- HTTPS
- Authenticate Git with GitHub credentials: Yes
- Login with a web browser

Follow the one-time device-code instructions in the terminal.

Verify and configure Git authentication:

```powershell
gh auth status
gh auth setup-git
gh api user --jq .login
```

## 4. Install Java for Jenkins

Check whether Java is already installed:

```powershell
java -version
```

If it is not installed, use `winget` to install a supported Java LTS version (Jenkins support depends on the Jenkins release):

```powershell
winget install --id Microsoft.OpenJDK.21 -e
```

Close and reopen PowerShell, then verify:

```powershell
java -version
```

Check current Jenkins Java requirements: https://www.jenkins.io/doc/book/platform-information/support-policy-java/

## 5. Install Jenkins on Windows

Download and install Jenkins LTS using the official guide:
https://www.jenkins.io/doc/book/installing/windows/

After installation:
1. Open `http://localhost:8080`.
2. Unlock Jenkins using the initial administrator password at the path shown by the installer.
3. Install suggested plugins.
4. Create your Jenkins administrator account.

Verify the Jenkins dashboard opens:

```text
http://localhost:8080
```

## 6. Install Docker Desktop

In PowerShell, run:

```powershell
winget install --id Docker.DockerDesktop -e
```

If `winget` is unavailable, install from:
https://docs.docker.com/desktop/setup/install/windows-install/

Start Docker Desktop. Enable its Linux container engine. Then reopen PowerShell and verify:

```powershell
docker --version
docker info
```

`docker info` must show a running Docker engine. If it fails, start Docker Desktop and resolve any WSL 2 or virtualization setup prompts first.

Install or enable OpenSSH Client on Windows if these commands are missing. In an Administrator PowerShell:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

Verify:

```powershell
ssh -V
scp
```

**Jenkins service note:** Jenkins must be able to access Git, Docker, SSH, and SCP. If these commands work only in your personal PowerShell, configure the Jenkins service account/agent before running the pipeline.

---

## 7. Create the project folder and website

In PowerShell:

```powershell
mkdir github-jenkins-docker-ec2
cd github-jenkins-docker-ec2
notepad index.html
```

Paste and save:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>GitHub Jenkins Docker EC2 Practical</title>
</head>
<body>
  <h1>GitHub Jenkins Docker EC2 Practical</h1>
  <p>Version 1</p>
</body>
</html>
```

## 8. Create the GitHub repository and push

In the same project folder:

```powershell
git init
git add index.html
git commit -m "Initial website"
gh repo create github-jenkins-docker-ec2 --public --source=. --remote=origin --push
```

Use `--private` instead of `--public` if the repository should be private.

Verify:

```powershell
git remote -v
gh repo view --web
```

If you already have a repository and project, do not run `git init` or create a nested repository. Open its folder and check:

```powershell
git status
git remote -v
```

---

## 9. Create the Dockerfile

```powershell
notepad Dockerfile
```

Paste and save:

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
```

Create `.dockerignore`:

```powershell
notepad .dockerignore
```

Paste and save:

```text
.git
.gitignore
*.pem
inventory.ini
.env
```

Do not put secrets in the project or Docker image.

---

## 10. Create AWS EC2 Ubuntu instance

In AWS Console:
1. Open **EC2 → Launch instance**.
2. Name it `docker-website-server`.
3. Choose a supported Ubuntu Server LTS image.
4. Choose an instance type after checking current AWS pricing.
5. Create/download a key pair and keep the `.pem` file secure.
6. Security group inbound rules:
   - SSH `22`: only your Jenkins machine's public IP.
   - HTTP `80`: `0.0.0.0/0` if the website should be publicly accessible.
7. Launch and copy the Public IPv4 address.

Do not expose SSH port 22 to everyone.

### Connect from PowerShell

Go to the folder containing your key, then:

```powershell
ssh -i ".\YOUR_KEY.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

Replace placeholders. Verify the server fingerprint using a trusted source before accepting it.

## 11. Install Docker Engine on EC2 Ubuntu

Follow the current official guide:
https://docs.docker.com/engine/install/ubuntu/

After installing, run on EC2:

```bash
sudo docker --version
sudo systemctl status docker --no-pager
```

Check whether port 80 is already in use:

```bash
sudo ss -ltnp | grep ':80'
```

If another web server/container is using port 80, stop or reconfigure it before deployment.

Exit SSH when done:

```bash
exit
```

---

## 12. Create Docker Hub repository

1. Open https://hub.docker.com/.
2. Create a repository named `github-jenkins-docker-ec2`.
3. Create a Docker Hub access token with the minimum permissions required to push images.
4. Keep the token private.

Image name:

```text
YOUR_DOCKERHUB_USERNAME/github-jenkins-docker-ec2:latest
```

## 13. Configure Jenkins credentials

Open `http://localhost:8080` → **Manage Jenkins → Credentials**.

### Docker Hub

Add a **Username with password** credential:
- Username: your Docker Hub username
- Password: Docker Hub access token
- ID: `dockerhub-creds`

### EC2 SSH

Add an **SSH Username with private key** credential:
- Username: `ubuntu`
- Private key: contents of the EC2 `.pem` key, entered securely
- ID: `ec2-deploy-key`

If this credential type is missing, check Jenkins plugins/credential providers. Do not put keys or tokens in GitHub.

Install/check Jenkins plugins under **Manage Jenkins → Plugins**:
- Git
- Pipeline
- Credentials Binding

---

## 14. Create Jenkinsfile

In PowerShell, inside the project folder:

```powershell
notepad Jenkinsfile
```

Paste and save the pipeline below. Replace `YOUR_DOCKERHUB_USERNAME` and `YOUR_EC2_PUBLIC_IP`.

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'YOUR_DOCKERHUB_USERNAME/github-jenkins-docker-ec2:latest'
        EC2_HOST = 'YOUR_EC2_PUBLIC_IP'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                echo 'Checking website...'
                bat 'if exist index.html (exit /b 0) else (exit /b 1)'
                echo 'Website check passed!'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t "%DOCKER_IMAGE%" .'
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_PASS'
                    )
                ]) {
                    bat '''
                        @echo off
                        powershell -NoProfile -Command "$env:DOCKERHUB_PASS | docker login --username $env:DOCKERHUB_USER --password-stdin"
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
                        ssh -i "%SSH_KEY%" %SSH_USER%@%EC2_HOST% "sudo docker pull %DOCKER_IMAGE%"
                        if errorlevel 1 exit /b 1
                        ssh -i "%SSH_KEY%" %SSH_USER%@%EC2_HOST% "sudo docker rm -f github-jenkins-docker-ec2 || true"
                        if errorlevel 1 exit /b 1
                        ssh -i "%SSH_KEY%" %SSH_USER%@%EC2_HOST% "sudo docker run -d --name github-jenkins-docker-ec2 --restart unless-stopped -p 80:80 %DOCKER_IMAGE%"
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }
    }
}
```

Before running:
- Verify `docker info` works for the Jenkins build account.
- Verify `ssh -V` works for the Jenkins build account.
- Verify EC2 SSH access and host key. Preconfigure the Jenkins service account's `known_hosts` with the independently verified EC2 host fingerprint; do not blindly trust an unknown key.
- Confirm the `ubuntu` account can run `sudo docker`.
- Confirm port 80 is available on EC2.
- This beginner pipeline replaces the running container, so a brief interruption is possible. It uses the `latest` tag for simplicity; production pipelines should use immutable version tags.

---

## 15. Push all project files

From the project directory:

```powershell
git add index.html Dockerfile .dockerignore Jenkinsfile
git commit -m "Add Docker EC2 deployment pipeline"
git push
```

Check your branch and remote if needed:

```powershell
git status
git remote -v
git branch
```

---

## 16. Configure Jenkins Pipeline job

1. Open `http://localhost:8080`.
2. Click **New Item**.
3. Name: `github-docker-ec2`.
4. Select **Pipeline** → **OK**.
5. Under **Pipeline**, configure:
   - Definition: `Pipeline script from SCM`
   - SCM: `Git`
   - Repository URL: `https://github.com/YOUR_GITHUB_USERNAME/github-jenkins-docker-ec2.git`
   - Branch Specifier: `*/main`
   - Script Path: `Jenkinsfile`
6. Save.

Replace `YOUR_GITHUB_USERNAME`. For a private repository, configure a Git credential with read access.

## 17. Run the deployment

1. Open `github-docker-ec2`.
2. Click **Build Now**.
3. Open the build number → **Console Output**.
4. Confirm Checkout, Test, Build Docker Image, Push Image to Docker Hub, and Deploy to EC2 succeed.
5. Open the website:

```text
http://YOUR_EC2_PUBLIC_IP
```

Check the running container on EC2:

```bash
sudo docker ps
sudo docker logs github-jenkins-docker-ec2
```

## 18. Enable automatic deployment with Poll SCM

In Jenkins:
1. Open `github-docker-ec2` → **Configure**.
2. Under **Build Triggers**, enable **Poll SCM**.
3. Enter:

```text
H/1 * * * *
```

4. Save.

This polls approximately once per minute, not every 30 seconds.

Test it by changing `Version 1` to `Version 2` in `index.html`, then:

```powershell
git add index.html
git commit -m "Update website version"
git push
```

Wait for polling, then inspect Jenkins Build History and Console Output. Refresh the EC2 website after deployment succeeds.

---

## 19. Quick troubleshooting

- `gh` not recognized: reopen PowerShell after installation or install from https://cli.github.com/.
- `java` not recognized: reopen PowerShell and check Java installation/PATH.
- Jenkins cannot use Docker: Docker Desktop may not be accessible to the Jenkins Windows service account.
- Docker Hub push denied: check `dockerhub-creds`, token, and repository name.
- SSH timeout: check EC2 public IP, port 22, security group, routing, and source IP.
- Permission denied (publickey): verify the key and username `ubuntu`.
- SSH host key error: verify and configure `known_hosts` securely.
- Port 80 already allocated: stop/reconfigure the existing web server or container.
- Website does not update: inspect Jenkins Console Output and `sudo docker logs github-jenkins-docker-ec2`.

## Result

GitHub stores the website and pipeline. Jenkins checks out the code, tests the website file, builds a Docker image, pushes it to Docker Hub, and deploys the container to Ubuntu EC2. Poll SCM checks for new commits and triggers the pipeline automatically.

## Official links

- Git: https://git-scm.com/download/win
- GitHub CLI: https://cli.github.com/
- GitHub CLI authentication: https://cli.github.com/manual/gh_auth_login
- Java requirements for Jenkins: https://www.jenkins.io/doc/book/platform-information/support-policy-java/
- Jenkins Windows install: https://www.jenkins.io/doc/book/installing/windows/
- Docker Desktop Windows install: https://docs.docker.com/desktop/setup/install/windows-install/
- Docker Engine Ubuntu install: https://docs.docker.com/engine/install/ubuntu/
- Docker Hub: https://hub.docker.com/
- AWS EC2: https://docs.aws.amazon.com/ec2/
