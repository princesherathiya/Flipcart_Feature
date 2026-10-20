# Practical: GitHub CLI + Jenkins Build + Poll SCM

## 1. Check Git

```bat
git --version
```

Set your commit identity:

```bat
git config --global user.name "Your Name"
git config --global user.email "YOUR_GITHUB_EMAIL"
```

## 2. Install GitHub CLI

Install from: https://cli.github.com/

Verify:

```bat
gh --version
```

## 3. Log in to GitHub CLI

```bat
gh auth login
gh auth status
gh auth setup-git
gh api user --jq .login
```

During `gh auth login`, select GitHub.com, HTTPS, authenticate Git with your GitHub credentials, and log in using a web browser. Follow the device-code instructions shown in the terminal.

## 4. Create the project

```bat
mkdir jenkins-pollscm-practical
cd jenkins-pollscm-practical
notepad index.html
```

Paste into `index.html` and save:

```html
<!doctype html>
<html>
<body>
<h1>GitHub and Jenkins CI Practical</h1>
<p>Version 1</p>
</body>
</html>
```

## 5. Push project to GitHub

```bat
git init
git add index.html
git commit -m "Initial website"
gh repo create jenkins-pollscm-practical --public --source=. --remote=origin --push
```

For a private repository, use `--private` instead of `--public`.

## 6. Install Java and Jenkins

Check Java:

```bat
java -version
```

If Java is not installed, install a Java version supported by the Jenkins LTS version you choose:
https://www.jenkins.io/doc/book/platform-information/support-policy-java/

Install Jenkins for Windows:
https://www.jenkins.io/doc/book/installing/windows/

Open Jenkins:

```text
http://localhost:8080
```

Unlock Jenkins, install suggested plugins, and create the administrator account.

## 7. Create Jenkinsfile

In the project folder:

```bat
notepad Jenkinsfile
```

Paste and save:

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Building the website...'
                bat 'if exist index.html (exit /b 0) else (exit /b 1)'
                echo 'Build completed successfully!'
            }
        }
    }
}
```

Push the Jenkinsfile:

```bat
git add Jenkinsfile
git commit -m "Add Jenkins pipeline"
git push
```

## 8. Configure the Jenkins job

1. Open `http://localhost:8080`.
2. Click **New Item**.
3. Enter `website-build`.
4. Select **Pipeline**, then click **OK**.
5. Under **Pipeline**, set:
   - Definition: `Pipeline script from SCM`
   - SCM: `Git`
   - Repository URL: `https://github.com/YOUR_GITHUB_USERNAME/jenkins-pollscm-practical.git`
   - Branch Specifier: `*/main`
   - Script Path: `Jenkinsfile`
6. Click **Save**.

Replace `YOUR_GITHUB_USERNAME` with your GitHub username. If your repository is private, configure a Jenkins credential with repository read access.

## 9. Run the first build

1. Open `website-build`.
2. Click **Build Now**.
3. Open the build number.
4. Click **Console Output**.

Expected output includes:

```text
Building the website...
Build completed successfully!
Finished: SUCCESS
```

## 10. Enable Poll SCM

1. Open `website-build` → **Configure**.
2. Under **Build Triggers**, check **Poll SCM**.
3. Enter this schedule:

```text
H/1 * * * *
```

4. Click **Save**.

This polls approximately every minute; it does not run every 30 seconds.

## 11. Test automatic builds

Edit the website:

```bat
notepad index.html
```

Change `Version 1` to `Version 2`, save, then run:

```bat
git add index.html
git commit -m "Update website"
git push
```

Wait for the next polling check. Then open Jenkins → `website-build` → **Build History** and inspect the latest **Console Output**.

## Useful links

- Git for Windows: https://git-scm.com/download/win
- GitHub CLI: https://cli.github.com/
- GitHub CLI login: https://cli.github.com/manual/gh_auth_login
- GitHub CLI Git setup: https://cli.github.com/manual/gh_auth_setup-git
- Jenkins Windows installation: https://www.jenkins.io/doc/book/installing/windows/
- Jenkins Pipeline: https://www.jenkins.io/doc/book/pipeline/

**Result:** GitHub CLI is authenticated, the project is pushed to GitHub, Jenkins runs a basic build, and Poll SCM checks for new commits automatically.
