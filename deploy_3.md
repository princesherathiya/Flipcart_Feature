# Practical: CI/CD Pipeline Using GitHub, Jenkins, and AWS EC2

## Aim
Integrate GitHub, Jenkins, and an Ubuntu EC2 instance so that code is checked and automatically deployed after a change is pushed to GitHub.

## Architecture

Developer pushes code → GitHub → Jenkins checks out the code → runs tests → deploys the site to EC2 → website is available over HTTP.

This practical uses a simple static website and Nginx. Jenkins runs on **Windows** and the EC2 server runs **Ubuntu Linux**. Commands are labelled by where to run them.

---

## 0. Accounts and requirements

Prepare:
- A GitHub account and repository.
- A Jenkins installation on your Windows PC.
- An AWS account with permission to create an EC2 instance and security group.
- An SSH key pair for EC2.
- A browser.

**Cost note:** EC2, public IPv4 addresses, data transfer, and other AWS resources may incur charges. Check the current AWS pricing and stop/terminate resources when finished. Never publish your AWS secret keys or EC2 `.pem` private key.

## 1. Install tools on Windows

Open **PowerShell**. For installations that use `winget`, run PowerShell as your normal Windows user unless otherwise noted.

### 1.1 Git

```powershell
winget install --id Git.Git -e
```

Close and reopen PowerShell, then verify:

```powershell
git --version
```

### 1.2 GitHub CLI (`gh`)

```powershell
winget install --id GitHub.cli -e
```

Close and reopen PowerShell, then verify and sign in:

```powershell
gh --version
gh auth login
gh auth status
gh auth setup-git
```

In the login prompts choose **GitHub.com**, **HTTPS**, and authenticate through the browser when prompted. Verify the signed-in username:

```powershell
gh api user --jq .login
```

### 1.3 Java for Jenkins

Check Java:

```powershell
java -version
```

If Java is not installed, install a supported LTS JDK. Example for JDK 21:

```powershell
winget install --id Microsoft.OpenJDK.21 -e
```

Open a new PowerShell window and verify again:

```powershell
java -version
```

Use a Java version supported by the Jenkins release you install. Check the current official Jenkins Java support page before choosing a different version:
https://www.jenkins.io/doc/book/platform-information/support-policy-java/

### 1.4 Jenkins

Download the **Windows installer** from the official Jenkins download page:
https://www.jenkins.io/download/

Install Jenkins as a Windows service using the installer. Choose a supported Java installation when requested. Keep the initial administrator password private.

Open:

```text
http://localhost:8080
```

Unlock Jenkins using the initial administrator password shown by the installer/instructions, install the suggested plugins, and create your administrator account.

Verify Jenkins is running by opening the page above. If port 8080 is already occupied, inspect the existing service rather than randomly changing ports.

### 1.5 OpenSSH client

Open **PowerShell as Administrator**:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

Verify in a normal PowerShell window:

```powershell
ssh -V
```

### 1.6 Optional: Docker

Docker is **not required** for this GitHub → Jenkins → EC2 pipeline. This practical deploys files to Nginx over SSH. You do not need Docker Desktop for the steps below.

---

## 2. Create an EC2 Ubuntu server

1. Sign in to the AWS Console and open **EC2**.
2. Launch an instance with a current supported **Ubuntu Server LTS** image.
3. Choose an instance size suitable for a small learning test and review the current free-tier/paid eligibility in your account.
4. Create or select an SSH key pair. Download the private `.pem` file and keep it private.
5. In the instance security group, add:
   - **SSH (TCP 22):** source should be **My IP** for manual administration. For a Jenkins host on a fixed public IP, restrict access to that IP where practical.
   - **HTTP (TCP 80):** source `0.0.0.0/0` only if the demo website should be publicly reachable.
   - Do not expose Jenkins port 8080 publicly without a secure, deliberate setup.
6. Launch the instance and wait until it is running and its status checks pass.
7. Copy its **Public IPv4 address**. The address can change after a stop/start unless you configure a stable address.

### 2.1 Connect to EC2 from Windows

In PowerShell, change the path and address to your own values:

```powershell
ssh -i "$HOME\Downloads\my-ec2-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

If SSH warns that the private key permissions are too open, restrict the key file's Windows ACL permissions. Do not upload the key to GitHub or paste it into a Jenkinsfile.

### 2.2 Install and start Nginx on EC2

Run these commands **inside the EC2 SSH session**:

```bash
sudo apt-get update
sudo apt-get install -y nginx
sudo systemctl enable --now nginx
sudo systemctl status nginx --no-pager
```

Test the default page by opening:

```text
http://YOUR_EC2_PUBLIC_IP
```

If it does not load, verify the EC2 security group allows inbound TCP 80 and that Nginx is active.

---

## 3. Prepare the website and tests on Windows

Create a project folder:

```powershell
mkdir "$HOME\github-jenkins-ec2-practical"
cd "$HOME\github-jenkins-ec2-practical"
```

Create `index.html`:

```powershell
@'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>GitHub Jenkins EC2 CI/CD</title>
</head>
<body>
  <h1>CI/CD Deployment Successful</h1>
  <p>Deployed automatically from GitHub through Jenkins to AWS EC2.</p>
  <p>Version 1</p>
</body>
</html>
'@ | Set-Content -Encoding utf8 index.html
```

Create a simple test script `test.ps1`:

```powershell
$ErrorActionPreference = 'Stop'

if (-not (Test-Path 'index.html')) {
    Write-Error 'Test failed: index.html is missing.'
    exit 1
}

$content = Get-Content 'index.html' -Raw
if ($content -notmatch 'CI/CD Deployment Successful') {
    Write-Error 'Test failed: expected page heading was not found.'
    exit 1
}

Write-Host 'All tests passed.'
```

Create `.gitignore`:

```powershell
@'
*.pem
.env
'@ | Set-Content .gitignore
```

**Never** add your private key, AWS credentials, passwords, or tokens to the repository.

---

## 4. Create the GitHub repository and push the project

From the project folder in PowerShell:

```powershell
git init
git branch -M main
git add index.html test.ps1 .gitignore
git commit -m "Add website and CI test"
gh repo create github-jenkins-ec2-practical --public --source=. --remote=origin --push
```

If the repository already exists, use its actual URL instead of creating a second repository:

```powershell
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git
git push -u origin main
```

If `origin` already exists, check it first:

```powershell
git remote -v
```

---

## 5. Configure Jenkins credentials

Jenkins needs an SSH key to copy the website to EC2. Do not put the private key inside the repository or Jenkinsfile.

### 5.1 Prepare a deployment key

For a practical, create a **separate deployment key pair** or use a carefully controlled existing key. The public key must be authorized for the EC2 `ubuntu` account. Keep the private key on the Jenkins machine and restrict who can access it.

For a new key generated on Windows, use PowerShell:

```powershell
ssh-keygen -t ed25519 -f "$HOME\.ssh\jenkins-ec2-deploy" -C "jenkins-ec2-deploy"
```

This produces:
- Private key: `$HOME\.ssh\jenkins-ec2-deploy`
- Public key: `$HOME\.ssh\jenkins-ec2-deploy.pub`

Authorize the **public** key for the EC2 `ubuntu` account through a secure administrative session. Example from an already authorized EC2 SSH session:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste the single line from the `.pub` file, save, then:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Do not paste the private key into `authorized_keys`.

### 5.2 Add the key to Jenkins

In Jenkins:
1. Open **Manage Jenkins → Credentials**.
2. Choose the appropriate global/domain credentials store and click **Add Credentials**.
3. Kind: **SSH Username with private key**.
4. Username: `ubuntu`.
5. Private Key: choose **Enter directly** and paste the contents of `jenkins-ec2-deploy` (private key file) into the secure credential field.
6. ID: `ec2-deploy-key`.
7. Save.

Jenkins must also trust the EC2 SSH host key. Before automating SSH, verify the host key fingerprint through a trusted AWS/administrator channel and configure it in the Jenkins service account's known-hosts file or Jenkins SSH host-key verification configuration. Do not disable host-key verification or blindly accept unknown keys in production.

### 5.3 Ensure Jenkins can run the required programs

Jenkins is running on Windows in this setup, so the Pipeline uses `bat` and PowerShell, not Linux `sh` steps.

The Jenkins Windows service account must be able to run Git and OpenSSH. If Jenkins cannot find them, configure the service's PATH to include the installed Git and OpenSSH directories, then restart the Jenkins service. Do not assume a command working in your personal PowerShell also works under the Jenkins service account.

---

## 6. Add a Jenkinsfile to the repository

Create a file named exactly `Jenkinsfile` (no `.txt` extension) in the project root:

```groovy
pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        EC2_HOST = 'YOUR_EC2_PUBLIC_IP'
        REMOTE_DIR = '/home/ubuntu/ci-cd-site'
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
                    $ErrorActionPreference = 'Stop'
                    if (-not (Test-Path 'index.html')) {
                        throw 'index.html is missing'
                    }
                    $content = Get-Content 'index.html' -Raw
                    if ($content -notmatch 'CI/CD Deployment Successful') {
                        throw 'Expected heading not found'
                    }
                    Write-Host 'All tests passed.'
                '''
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
                        ssh -i "%SSH_KEY%" -o BatchMode=yes %SSH_USER%@%EC2_HOST% "mkdir -p %REMOTE_DIR%"
                        if errorlevel 1 exit /b 1

                        scp -i "%SSH_KEY%" -o BatchMode=yes index.html %SSH_USER%@%EC2_HOST%:%REMOTE_DIR%/index.html
                        if errorlevel 1 exit /b 1

                        ssh -i "%SSH_KEY%" -o BatchMode=yes %SSH_USER%@%EC2_HOST% "sudo install -m 0644 %REMOTE_DIR%/index.html /var/www/html/index.html"
                        if errorlevel 1 exit /b 1

                        ssh -i "%SSH_KEY%" -o BatchMode=yes %SSH_USER%@%EC2_HOST% "sudo nginx -t && sudo systemctl reload nginx"
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Review the stage logs before retrying.'
        }
    }
}
```

**Important:** Replace `YOUR_EC2_PUBLIC_IP` with the real EC2 public IP. `REMOTE_DIR` is a path on EC2. The commands above assume the `ubuntu` account has the authorized deployment key and that the `ubuntu` user can run the listed Nginx deployment commands via `sudo` (default on many Ubuntu EC2 images). If your host uses different permissions, configure a restricted deployment method rather than broadening privileges unnecessarily.

### Add and push the Jenkinsfile

```powershell
git add Jenkinsfile
git commit -m "Add Jenkins CI CD pipeline"
git push
```

---

## 7. Create the Jenkins Pipeline job

1. Open `http://localhost:8080`.
2. Select **New Item**.
3. Name: `github-ec2-cicd`.
4. Choose **Pipeline**, then **OK**.
5. Under **Pipeline**:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/YOUR_GITHUB_USERNAME/github-jenkins-ec2-practical.git`
   - Branch Specifier: `*/main`
   - Script Path: `Jenkinsfile`
6. Save.
7. Click **Build Now**.
8. Open **Console Output** and check that Checkout, Test, and Deploy all pass.

For a public GitHub repository, checkout normally needs no credentials. For a private repository, configure a GitHub credential in Jenkins and select it in the job configuration.

### If PowerShell is not recognized by Jenkins

Install/configure Windows PowerShell or update the Jenkinsfile to use the installed shell. Check the Jenkins agent log and service account PATH. Keep the Pipeline steps consistent with the actual agent OS.

---

## 8. Make Jenkins start builds automatically

### Option A — Poll SCM (easiest for localhost)

1. Open the Jenkins job → **Configure**.
2. Find **Build Triggers**.
3. Check **Poll SCM**.
4. Enter this schedule:

```text
H/2 * * * *
```

This checks for changes approximately every two minutes. It is polling, so a build may not start immediately after a push. Save.

### Option B — GitHub webhook (near-immediate)

A GitHub webhook cannot reach `http://localhost:8080` on your private PC. For webhooks, Jenkins needs a stable, publicly reachable **HTTPS** URL, with appropriate authentication, firewall rules, and reverse-proxy/security configuration. Do not expose an unsecured Jenkins instance directly to the internet.

Once secure public HTTPS access is configured:
1. In the GitHub repository, open **Settings → Webhooks → Add webhook**.
2. Payload URL: `https://YOUR_SECURE_JENKINS_HOST/github-webhook/`
3. Content type: `application/json`.
4. Select the push event.
5. In the Jenkins job, enable the GitHub hook trigger (wording may vary by installed plugins).
6. Save and test the webhook delivery in GitHub.

For a local classroom practical, use **Poll SCM** if you do not already have a secure public Jenkins endpoint.

---

## 9. Verify automatic deployment

1. Open `index.html` and change `Version 1` to `Version 2`.
2. Commit and push:

```powershell
git add index.html
git commit -m "Update website to version 2"
git push
```

3. Wait for the Poll SCM interval, or use the configured webhook.
4. Open Jenkins → `github-ec2-cicd` → Build History → latest build → Console Output.
5. Confirm Checkout, Test, and Deploy succeeded.
6. Refresh:

```text
http://YOUR_EC2_PUBLIC_IP
```

The page should show the new version.

---

## 10. Troubleshooting

| Error | What to check |
|---|---|
| `java is not recognized` | Install a supported JDK, reopen the terminal, and check `java -version`. |
| Jenkins page does not open | Check that the Jenkins Windows service is running and that port 8080 is correct. |
| `git` or `ssh` not found in Jenkins | Configure PATH for the Jenkins service account and restart the service. |
| `Permission denied (publickey)` | Verify the credential ID, SSH username (`ubuntu`), matching public key in `authorized_keys`, and private-key access. |
| SSH host-key verification failed | Verify the server fingerprint securely and update the Jenkins service account's known-hosts/host-key configuration. Do not turn off verification as a workaround. |
| Connection timeout on SSH | Check EC2 state, public IP, route, security group inbound TCP 22, and source IP. |
| Website cannot be opened | Check inbound TCP 80, `sudo systemctl status nginx`, and `sudo nginx -t` on EC2. |
| Pipeline fails at Test | Ensure `index.html` contains the expected heading and is checked into Git. |
| Changes do not trigger a build | Check Poll SCM is enabled, job branch is `*/main`, the push reached the same repository, and the latest commit is on `main`. |
| Jenkins works only in personal PowerShell | The Jenkins Windows service runs under a different account/environment; configure its PATH and credentials separately. |

## Result

A CI/CD pipeline is configured in which GitHub stores the website source, Jenkins checks out pushed changes, runs a content test, and securely deploys the updated HTML file to Nginx on AWS EC2. Poll SCM provides automatic detection of new commits; a properly secured GitHub webhook can be used for faster triggering.

## Cleanup

When the practical is complete, remove any unused EC2 resources and review security groups, credentials, and key files. Stop or terminate resources you no longer need, considering whether any public IP or storage charges remain.
