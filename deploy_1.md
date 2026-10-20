# Practical: Git Collaboration, Merge Conflict Resolution, and Vercel Deployment

## Aim
To demonstrate Git collaboration by creating a merge conflict, resolving it, merging the changes, and deploying the updated project using Vercel.

## Tools required
- Git
- GitHub account
- Node.js (optional for this simple static website)
- Vercel account

## Simple project idea
Create one simple website with an `index.html` file. Two branches will make different changes to the same line, which creates a merge conflict. Resolve the conflict, merge the branch, and deploy the final version on Vercel.

---

## 1. Install Git and GitHub CLI

Open **PowerShell** on Windows.

```powershell
winget install --id Git.Git -e
winget install --id GitHub.cli -e
```

Close and reopen PowerShell, then verify:

```powershell
git --version
gh --version
```

Sign in to GitHub:

```powershell
gh auth login
gh auth status
gh auth setup-git
```

Choose GitHub.com and HTTPS in the prompts, then complete browser authentication.

---

## 2. Create one simple project

```powershell
mkdir "$HOME\git-merge-vercel"
cd "$HOME\git-merge-vercel"
```

Create `index.html`:

```powershell
@'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Git Collaboration Demo</title>
</head>
<body>
  <h1>Git Collaboration Practical</h1>
  <p>Welcome to Version 1.</p>
  <p>Deployed using Vercel.</p>
</body>
</html>
'@ | Set-Content -Encoding utf8 index.html
```

Initialize Git and make the first commit:

```powershell
git init
git branch -M main
git add index.html
git commit -m "Create initial website"
```

---

## 3. Create a GitHub repository and push

```powershell
gh repo create git-merge-vercel --public --source=. --remote=origin --push
```

If you already created a repository, add its actual URL instead:

```powershell
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git
git push -u origin main
```

Check the remote:

```powershell
git remote -v
```

---

## 4. Create two branches and make conflicting changes

We will change the same paragraph on both branches.

### 4.1 Create the `feature-home` branch

```powershell
git switch -c feature-home
```

Replace the website with the following version:

```powershell
@'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Git Collaboration Demo</title>
</head>
<body>
  <h1>Git Collaboration Practical</h1>
  <p>Welcome to the Feature Branch.</p>
  <p>Deployed using Vercel.</p>
</body>
</html>
'@ | Set-Content -Encoding utf8 index.html
```

Commit the feature change:

```powershell
git add index.html
git commit -m "Update welcome message in feature branch"
```

### 4.2 Create a second branch from `main`

Switch back to `main`:

```powershell
git switch main
```

Create a second branch:

```powershell
git switch -c update-home
```

Make a different change to the **same paragraph**:

```powershell
@'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Git Collaboration Demo</title>
</head>
<body>
  <h1>Git Collaboration Practical</h1>
  <p>Welcome to the Updated Main Branch.</p>
  <p>Deployed using Vercel.</p>
</body>
</html>
'@ | Set-Content -Encoding utf8 index.html
```

Commit the second change:

```powershell
git add index.html
git commit -m "Update welcome message in update branch"
```

At this point, both branches changed the same line in different ways.

---

## 5. Create the merge conflict

While on `update-home`, merge `feature-home`:

```powershell
git merge feature-home
```

Git should report a conflict in `index.html`. This is expected for this practical.

Check the conflict:

```powershell
git status
```

Open `index.html` in Notepad:

```powershell
notepad index.html
```

You should see conflict markers similar to:

```text
<<<<<<< HEAD
  <p>Welcome to the Updated Main Branch.</p>
=======
  <p>Welcome to the Feature Branch.</p>
>>>>>>> feature-home
```

The exact wording may differ depending on your current branch. These markers show the two competing versions.

---

## 6. Resolve the conflict

Edit `index.html` so it contains the final chosen message, with **all conflict markers removed**. For example:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Git Collaboration Demo</title>
</head>
<body>
  <h1>Git Collaboration Practical</h1>
  <p>Welcome to the Final Merged Version.</p>
  <p>Changes from both branches have been reviewed.</p>
  <p>Deployed using Vercel.</p>
</body>
</html>
```

Save the file, then stage and commit the resolution:

```powershell
git add index.html
git commit -m "Resolve merge conflict and combine changes"
```

Confirm that the working tree is clean:

```powershell
git status
```

---

## 7. Merge the resolved work into `main`

Push the feature branch so it is backed up on GitHub:

```powershell
git push -u origin feature-home
git push -u origin update-home
```

Switch to `main` and merge the resolved branch:

```powershell
git switch main
git merge update-home
```

Push the final `main` branch:

```powershell
git push origin main
```

Check the commit history:

```powershell
git log --oneline --graph --all
```

**What happened?** The conflict was created when two branches changed the same line. You resolved it in `update-home`, committed the resolution, then merged that branch into `main`.

---

## 8. Deploy the final project using Vercel

### Option A — Easiest: deploy through the Vercel website

1. Sign in at https://vercel.com/
2. Choose **Add New → Project** (the wording may vary).
3. Import the GitHub repository `git-merge-vercel`.
4. For a plain HTML project, leave the framework preset as **Other** or **No Framework** if prompted.
5. Set the project root to the repository root.
6. No build command is needed for this static HTML page. If Vercel asks for an output directory, use the root directory (`.`) or leave the default appropriate for a static project.
7. Click **Deploy**.
8. Open the Vercel deployment URL and verify the final merged message appears.

After the GitHub repository is connected, new commits pushed to `main` normally trigger a new production deployment automatically.

### Option B — Deploy using Vercel CLI

Install Node.js LTS first if Node.js/npm are not available:
https://nodejs.org/

Verify:

```powershell
node --version
npm --version
```

Install Vercel CLI:

```powershell
npm install --global vercel
vercel --version
```

From the project directory, sign in and deploy:

```powershell
vercel login
vercel
```

Follow the prompts to link or create the Vercel project. For a production deployment:

```powershell
vercel --prod
```

For a GitHub-linked production workflow, the Vercel dashboard import is the recommended simple option.

---

## 9. Verify the practical

- Git repository was initialized.
- A GitHub repository was created and pushed.
- Two branches changed the same line differently.
- Git reported a merge conflict.
- Conflict markers were removed and the resolution was committed.
- The resolved branch was merged into `main`.
- The final code was pushed to GitHub.
- Vercel served the final website.

## Expected result
A website displaying **“Welcome to the Final Merged Version.”** is deployed on Vercel, and the Git history shows the feature branches and merge-conflict resolution.

## Troubleshooting

| Problem | Fix |
|---|---|
| `git` or `gh` is not recognized | Close and reopen PowerShell after installation; verify the installation succeeded. |
| `gh auth login` fails | Run `gh auth status`, then repeat `gh auth login`. |
| Merge reports no conflict | Confirm both branches started from `main` and changed the exact same line, then try `git merge feature-home` while on `update-home`. |
| Git says there are uncommitted changes | Run `git status`; save, commit, or safely stash your changes before switching branches. |
| Push is rejected | Run `git status` and check that the correct remote and branch are configured. Do not force-push as a first fix. |
| Vercel page is old | Check the latest deployment status and confirm the resolved commit was pushed to the connected production branch. |

## Conclusion
This practical demonstrates Git branching, a merge conflict, manual conflict resolution, merging the resolved branch into `main`, pushing to GitHub, and deploying the updated static website using Vercel.
