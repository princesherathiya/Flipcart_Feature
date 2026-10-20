Use a single key pair for both only if it was selected for both instances. If they use different key pairs, configure the matching private key per host in the inventory.

---

## 3. Install Ansible on WSL Ubuntu

Open the WSL Ubuntu terminal.

Update packages:

```bash
sudo apt update
sudo apt install -y ansible git openssh-client
ansible --version
git --version
```

If your Ubuntu release does not provide the desired Ansible version, use the official Ansible installation guidance:
https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

### 3.1 Place the EC2 private key securely

For example, if your downloaded key is in Windows Downloads, copy it into WSL:

```bash
mkdir -p ~/keys
cp /mnt/c/Users/YOUR_WINDOWS_USERNAME/Downloads/my-ec2-key.pem ~/keys/
chmod 600 ~/keys/my-ec2-key.pem
```

Replace the Windows username, filename, and path with your real values. Never commit the private key to GitHub.

Test SSH connectivity to both servers (replace addresses):

```bash
ssh -i ~/keys/my-ec2-key.pem ubuntu@DEV_EC2_PUBLIC_IP
ssh -i ~/keys/my-ec2-key.pem ubuntu@PROD_EC2_PUBLIC_IP
```

Accept the host key only after verifying the server identity/fingerprint through a trusted source. Exit each SSH session with `exit`.

---

## 4. Create the GitHub project

In WSL:

```bash
mkdir -p ~/github-ansible-two-ec2
cd ~/github-ansible-two-ec2
```

Create `index.html`:

```bash
cat > index.html <<'EOF'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Ansible Two-Environment Deployment</title>
</head>
<body>
  <h1>Ansible Deployment Demo</h1>
  <p>Environment: VERSION_PLACEHOLDER</p>
  <p>Deployed with GitHub and Ansible.</p>
</body>
</html>
EOF
```

Create `.gitignore`:

```bash
cat > .gitignore <<'EOF'
*.pem
inventory.ini
.env
EOF
```

Initialize Git:

```bash
git init
git branch -M main
git add index.html .gitignore
git commit -m "Add website for dev and production deployment"
```

### 4.1 Create and push the GitHub repository

If GitHub CLI (`gh`) is installed and authenticated on the WSL machine:

```bash
gh auth status
gh repo create github-ansible-two-ec2 --public --source=. --remote=origin --push
```

If you do not use `gh`, create a repository on GitHub and add its URL:

```bash
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git
git push -u origin main
```

Do not run both remote setup paths. If `origin` already exists, inspect it with `git remote -v`.

---

## 5. Create Ansible inventory for two EC2 servers

Create `inventory.ini` in the project folder. Replace the two IP addresses and the key path with your real values:

```ini
[dev]
dev-web ansible_host=DEV_EC2_PUBLIC_IP ansible_user=ubuntu ansible_ssh_private_key_file=/home/YOUR_LINUX_USERNAME/keys/my-ec2-key.pem

[prod]
prod-web ansible_host=PROD_EC2_PUBLIC_IP ansible_user=ubuntu ansible_ssh_private_key_file=/home/YOUR_LINUX_USERNAME/keys/my-ec2-key.pem

[webservers:children]
dev
prod
```

If the two EC2 instances use different key pairs, set the correct `ansible_ssh_private_key_file` for each host.

The `inventory.ini` file is excluded by `.gitignore` because it contains environment details. Do not commit private keys.

### 5.1 Test Ansible connectivity

```bash
ansible -i inventory.ini all -m ping
```

Expected result for each host:

```text
"ping": "pong"
```

If SSH fails, check IP addresses, security-group TCP 22, username `ubuntu`, key path, key permissions (`chmod 600`), and whether each instance is running.

---

## 6. Create the Ansible deployment playbook

Create `deploy.yml` in the repository:

```bash
cat > deploy.yml <<'EOF'
---
- name: Configure and deploy the website to selected environment
  hosts: webservers
  become: true
  vars:
    web_root: /var/www/html
  tasks:
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure web root exists
      ansible.builtin.file:
        path: "{{ web_root }}"
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Deploy website from the Ansible control-node checkout
      ansible.builtin.copy:
        src: index.html
        dest: "{{ web_root }}/index.html"
        owner: root
        group: root
        mode: "0644"

    - name: Ensure Nginx is enabled and running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
EOF
```

This playbook copies the `index.html` file from the Ansible control node's current project directory to the selected EC2 host. It does not clone GitHub on the EC2 machines.

---

## 7. Deploy to Development first

From the project directory, run:

```bash
ansible-playbook -i inventory.ini deploy.yml --limit dev
```

Open this URL in your browser:

```text
http://DEV_EC2_PUBLIC_IP
```

Verify the development site works before proceeding.

### 7.1 Optional: make the dev page visibly different

Change the version line in `index.html` to `Environment: DEVELOPMENT - Version 1`, then deploy to dev again:

```bash
ansible-playbook -i inventory.ini deploy.yml --limit dev
```

Commit and push the change if you want GitHub to contain this version:

```bash
git add index.html deploy.yml
git commit -m "Label development deployment"
git push
```

---

## 8. Promote the approved version to Production

Only after you have reviewed and tested the version on Dev, deploy to Production:

```bash
ansible-playbook -i inventory.ini deploy.yml --limit prod
```

Open:

```text
http://PROD_EC2_PUBLIC_IP
```

**Promotion principle:** Dev and Production are separate hosts, but this basic playbook deploys the same checked-out `index.html` to whichever group you select. Keep the code version under Git control and promote the tested commit deliberately.

If the `index.html` was modified after the last commit, first commit and push the intended version, and make sure the Ansible control node is deploying that reviewed version.

---

## 9. Update the site and repeat the workflow

Edit `index.html`:

```bash
nano index.html
```

Change the page text/version and save.

Commit and push the new version:

```bash
git add index.html deploy.yml
git commit -m "Update website version"
git push origin main
```

Deploy to Dev and test it:

```bash
ansible-playbook -i inventory.ini deploy.yml --limit dev
```

After approval, deploy the same version to Production:

```bash
ansible-playbook -i inventory.ini deploy.yml --limit prod
```

Verify the two URLs in your browser.

---

## 10. Useful Ansible commands

Check inventory:

```bash
ansible-inventory -i inventory.ini --graph
```

Ping both servers:

```bash
ansible -i inventory.ini all -m ping
```

Run a playbook only against Dev:

```bash
ansible-playbook -i inventory.ini deploy.yml --limit dev
```

Run a playbook only against Production:

```bash
ansible-playbook -i inventory.ini deploy.yml --limit prod
```

Check Nginx status on both servers:

```bash
ansible -i inventory.ini webservers -b -m command -a "systemctl is-active nginx"
```

---

## Troubleshooting

| Problem | What to check |
|---|---|
| `ansible: command not found` | Install Ansible in WSL Ubuntu and verify `ansible --version`. |
| SSH connection timeout | Confirm EC2 is running, IP is correct, route is working, and security group allows TCP 22 from the control node. |
| `Permission denied (publickey)` | Verify `ansible_user=ubuntu`, key file path, matching key pair, and `chmod 600` permissions. |
| Ansible cannot find `index.html` | Run `ansible-playbook` from the project directory containing `deploy.yml` and `index.html`. |
| Website does not load | Confirm inbound TCP 80 is allowed and Nginx is active on the target EC2 host. |
| Wrong environment updated | Check the inventory group and always use `--limit dev` or `--limit prod` explicitly. |
| Production is updated too early | Deploy to Dev first, verify it, then run the Production command only after approval. |
| `gh` command missing | Install GitHub CLI in the environment where you run it, or create the GitHub repository in the browser and configure `origin` manually. |

## Result
The project code is stored in GitHub. Ansible connects to two separate Ubuntu EC2 instances and deploys the website to **Development** first and **Production** only after verification. The inventory groups and `--limit` option control which environment receives a deployment.

## Important limitation
This guide demonstrates **manual Ansible deployment after pushing code to GitHub**. A Git push alone does not start Ansible automatically. To automate that next, connect a secure CI workflow (such as Jenkins or GitHub Actions) to run Ansible after a push, with protected credentials and an approval gate before Production.

## Cleanup
When finished, stop or terminate EC2 instances you no longer need, review security-group rules, and keep `.pem` keys and inventory details out of GitHub. AWS resources may continue to incur charges while running.
