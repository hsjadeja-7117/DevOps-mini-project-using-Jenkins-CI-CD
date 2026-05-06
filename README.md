# DevOps-mini-project-using-Jenkins-CI-CD

<div align="center">

# 🚀 Automated Static Website Deployment
### GitHub → Jenkins → Azure App Service

![RHEL](https://img.shields.io/badge/OS-RHEL%209-red?style=for-the-badge&logo=redhat&logoColor=white)
![Jenkins](https://img.shields.io/badge/CI%2FCD-Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![Azure](https://img.shields.io/badge/Cloud-Azure%20App%20Service-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Git](https://img.shields.io/badge/VCS-Git%20%26%20GitHub-F05032?style=for-the-badge&logo=git&logoColor=white)

*A beginner-friendly DevOps mini-project that automates website deployment using a Jenkins CI/CD pipeline on a RHEL 9 server, pushing code directly to Microsoft Azure App Service.*

</div>

---

## 📌 What is This Project?

Every time you update your website code, someone has to manually upload it to the server. That's slow, error-prone, and unprofessional.

This project eliminates that entirely.

Once set up, you only need to **edit your code on GitHub** and commit the change. Within 60 seconds, Jenkins automatically detects the change, pulls the latest code, and pushes it live to Azure — **without you touching anything else.**

This is called a **CI/CD Pipeline** (Continuous Integration / Continuous Deployment), and it is a core skill used by DevOps engineers in the real world.

---

## 🏗️ System Architecture

```
Developer (You)
     |
     | git commit + push
     ▼
┌─────────────┐
│   GitHub    │  ← Stores your source code (index.html)
└──────┬──────┘
       │ Jenkins polls every 1 min
       ▼
┌──────────────────────────────┐
│  Jenkins on RHEL 9 VM        │  ← CI/CD Engine (your Linux server)
│  - Pulls latest code         │
│  - Runs deploy shell script  │
└──────────────┬───────────────┘
               │ git push via HTTPS
               ▼
┌──────────────────────────────┐
│  Microsoft Azure App Service │  ← Live website on the internet
│  (your-app.azurewebsites.net)│
└──────────────────────────────┘
```

---

## 🧰 Tech Stack

| Tool | Role |
|---|---|
| **GitHub** | Source code storage (version control) |
| **Red Hat Enterprise Linux 9** | Server OS where Jenkins runs |
| **Jenkins** | CI/CD automation engine |
| **Microsoft Azure App Service** | Cloud hosting for the live website |
| **Git** | Code transfer between all three systems |
| **Azure CLI** | Used to fix Azure credential issues via terminal |

---

## ✅ Prerequisites

Before you begin, make sure you have the following ready:

- [ ] A **GitHub account** with your `index.html` file in a public repository
- [ ] A **RHEL 9 Virtual Machine** (physical, VMware, or a cloud VM) with internet access
- [ ] A **Microsoft Azure account** (free tier works perfectly for this project)
- [ ] Basic comfort using a **Linux terminal**

> 💡 **Tip:** You can get a free Azure account with $200 credits at [azure.microsoft.com/free](https://azure.microsoft.com/free). This project costs nothing on the free tier.

---

## 📁 Phase 0 — Prepare Your Code on GitHub

1. Log in to [github.com](https://github.com) and create a **new repository** (e.g., `My-DevOps-Project`).
2. Upload your `index.html` file into the repository.
3. Make sure your default branch is named **`main`**.

> 💡 **Tip:** Keep your GitHub repository **Public**. This way Jenkins can access it without needing any login credentials for GitHub.

---

## ☁️ Phase 1 — Set Up Azure App Service

This is where your website will live on the internet.

### 1.1 Create the Web App

1. Log in to the [Azure Portal](https://portal.azure.com).
2. Click **Create a resource** → search for **Web App** → click **Create**.
3. Fill in the details:
   - **Resource Group:** Create a new one (e.g., `devops-project-rg`)
   - **Name:** Give it a unique name (e.g., `my-portfolio-2024`) — this becomes your URL
   - **Publish:** Select `Code`
   - **Runtime Stack:** Select `PHP 8.2` or `Node.js` — a plain HTML file works on either
   - **Operating System:** `Linux`
   - **Region:** Choose the one closest to you
4. Click **Review + Create** → **Create**.

> ⚠️ **Important:** Choose your app name carefully. Azure uses it as your public URL (`yourname.azurewebsites.net`) and it **cannot be changed later**.

### 1.2 Configure Deployment Source

1. Once your Web App is created, open it and go to **Deployment Center** on the left menu.
2. Under the **Settings** tab, set **Source** to **Local Git**.
3. Click **Save** at the top.
4. After saving, the page will show a **Git Clone URI**. Copy and save it. It looks like:
   ```
   https://your-app-name.scm.azurewebsites.net/your-app-name.git
   ```

### 1.3 Enable SCM Authentication

By default, Azure blocks external git pushes. You must turn this on.

1. On the left menu, click **Configuration** → then click the **General settings** tab.
2. Find **SCM Basic Auth Publishing Credentials** and switch it to **On**.
3. Click **Save** and wait for the confirmation message.

> 💡 **Real World Tip:** This step is frequently missed in tutorials and causes most `Authentication failed` errors. Always enable this before testing your pipeline.

### 1.4 Create a Deployment Username and Password

Because Azure deployment usernames must be **globally unique** (shared across all Azure customers worldwide), you cannot use a generic name like `admin`.

The safest way to set this is via the **Azure Cloud Shell** (the `>_` icon at the top of the Azure portal):

```bash
az webapp deployment user set --user-name YOUR_UNIQUE_NAME --password YourPassword123
```

**Rules for the password:**
- Minimum 8 characters
- Must contain uppercase, lowercase, and a number
- **Do NOT use special characters like `@` or `#`** — they break the Git URL format later

> ⚠️ **Common Mistake:** Using a password with an `@` symbol (e.g., `Jenkins@123`) will cause a `Could not resolve host` error in Jenkins because Git misreads the `@` in the URL. Use a plain alphanumeric password like `Jenkins123` instead.

Once the command succeeds, you will see a JSON output confirming your `publishingUserName`. Save your username and password securely.

---

## 🔧 Phase 2 — Prepare Your RHEL 9 VM

Connect to your RHEL 9 server via terminal and run the following commands **one by one**.

### 2.1 Install Java 21

Jenkins now requires **Java 21 or higher**. Java 17 will cause Jenkins to fail at startup.

```bash
sudo dnf install java-21-openjdk -y
```

Verify the installation:

```bash
java -version
# Expected output: openjdk version "21.x.x"
```

> ⚠️ **Common Mistake:** Installing `java-17-openjdk` (which is what most older guides recommend) will cause Jenkins to fail with: `Running with Java 17... which is older than the minimum required version (Java 21)`. Always install Java 21.

### 2.2 Install Git and wget

```bash
sudo dnf install git wget -y
```

---

## ⚙️ Phase 3 — Install Jenkins on RHEL 9

### 3.1 Add the Jenkins Repository

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
```

### 3.2 Import the Security Key

```bash
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
```

### 3.3 Install Jenkins

```bash
sudo dnf install jenkins -y
```

### 3.4 Extend the Startup Timeout

On a fresh VM, Jenkins can take 2–3 minutes to unpack and start for the first time. Linux's default timeout is only 90 seconds, which causes Jenkins to be killed before it finishes. Fix this before starting:

```bash
sudo mkdir -p /etc/systemd/system/jenkins.service.d

sudo bash -c 'echo "[Service]" > /etc/systemd/system/jenkins.service.d/override.conf'

sudo bash -c 'echo "TimeoutStartSec=600" >> /etc/systemd/system/jenkins.service.d/override.conf'

sudo systemctl daemon-reload
```

> 💡 **Real World Tip:** This timeout issue is extremely common on VMs with limited RAM (less than 2GB). The fix above gives Jenkins a full 10 minutes to start, which is more than enough.

### 3.5 Start Jenkins

```bash
sudo systemctl enable --now jenkins
```

Verify it is running:

```bash
sudo systemctl status jenkins
# Look for: Active: active (running)
```

### 3.6 Open the Firewall Port

RHEL has a strict firewall by default. Open port 8080 so you can access Jenkins from your browser:

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

---

## 🌐 Phase 4 — Unlock and Configure Jenkins in the Browser

### 4.1 Find Your VM's IP Address

```bash
curl ifconfig.me
```

### 4.2 Get the Admin Password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the long string it prints.

### 4.3 First-Time Setup

1. Open a browser and go to: `http://<your-vm-ip>:8080`
2. Paste the admin password and click **Continue**.
3. Click **Install suggested plugins** and wait for them to finish (takes 3–5 minutes).
4. Create your Jenkins admin **username and password**.
5. Click **Save and Finish** → **Start using Jenkins**.

> 💡 **Tip:** Write down the Jenkins username and password you create here. You will use it every time you log in.

---

## 🔗 Phase 5 — Create the CI/CD Pipeline in Jenkins

This is the core of the project — connecting GitHub to Azure through Jenkins.

### 5.1 Create a New Job

1. On the Jenkins dashboard, click **New Item**.
2. Type a project name (e.g., `Azure-Deploy-Pipeline`).
3. Select **Freestyle project**.
4. Press **Enter** or scroll down and click **OK**.

### 5.2 Connect to GitHub (Source Code Management)

1. Scroll down to **Source Code Management**.
2. Select **Git**.
3. In the **Repository URL** box, paste your GitHub repository URL:
   ```
   https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git
   ```
4. Leave **Credentials** as `None` (since the repo is public).
5. Under **Branches to build**, change `*/master` to **`*/main`**.

> ⚠️ **Common Mistake:** GitHub's default branch is now called `main`, not `master`. If you leave it as `*/master`, Jenkins will fail with: `Couldn't find any revision to build`.

### 5.3 Set the Automation Trigger (Polling)

1. Scroll down to **Build Triggers**.
2. Check the box for **Poll SCM**.
3. In the **Schedule** box, type exactly:
   ```
   * * * * *
   ```
   (Five asterisks separated by spaces — this tells Jenkins to check GitHub every 1 minute.)

> 💡 **Real World Tip:** In production environments, teams use **GitHub Webhooks** instead of polling. Webhooks are instant and more efficient. Polling every minute is simpler to set up and perfect for learning and small projects.

### 5.4 Add the Deployment Script (Build Steps)

1. Scroll down to **Build Steps**.
2. Click **Add build step** → select **Execute shell**.
3. Paste the following command into the text box, replacing the placeholders with your actual values:

```bash
git push https://YOUR_AZURE_USERNAME:YOUR_AZURE_PASSWORD@YOUR_SCM_URI:443/YOUR_APP_NAME.git HEAD:refs/heads/main --force
```

**Example with real values:**
```bash
git push https://xyz_admin:xyz123@devops-myapp.scm.centralindia-01.azurewebsites.net:443/DevOps-MyApp.git HEAD:refs/heads/main --force
```

> 💡 **Why `refs/heads/main`?** When pushing to a brand new, empty Azure Git repository for the first time, you must use the full Git refname (`HEAD:refs/heads/main`). Using just `HEAD:main` will fail with: `The destination you provided is not a full refname`. After the first successful push, both formats work.

### 5.5 Save the Configuration

Click the **Save** button at the bottom of the page.

---

## 🧪 Phase 6 — Test and Verify

### 6.1 Run the First Manual Build

1. On your project dashboard, click **Build Now** on the left side menu.
2. A new entry will appear under **Build History** (e.g., `#1`).
3. Click on the build number → click **Console Output**.
4. Scroll to the bottom. You should see:

```
remote: Deployment successful.
Finished: SUCCESS
```

### 6.2 View Your Live Website

Open your browser and go to your Azure URL:
```
devops-iife-cycle-by-harshvardhansinh-aqhmewb0czgwbph9.centralindia-01.azurewebsites.net
```

You should see your `index.html` live on the internet. 🎉

### 6.3 Test the Full Automation (The Real Proof)

This is the step that proves your CI/CD pipeline is genuinely working.

1. Go to your `index.html` file on GitHub and click the **Edit (pencil) icon**.
2. Change any visible text (e.g., change a heading from `"Hello"` to `"Hello - v2"`).
3. Click **Commit changes**.
4. Go back to your Jenkins dashboard and **do not click anything**.
5. Within 60 seconds, a new build will automatically appear and run.
6. Once it finishes with `SUCCESS`, refresh your Azure website.
7. Your change will be live.

---

## 🛠️ Troubleshooting Guide

These are the real issues encountered during this project, and how they were fixed.

| Error | Cause | Fix |
|---|---|---|
| `Running with Java 17... older than minimum required` | Wrong Java version | Install `java-21-openjdk` and remove Java 17 |
| `Start request repeated too quickly` | Jenkins crashed too fast, systemd locked it | Run `sudo systemctl reset-failed jenkins` then `daemon-reload` |
| `start operation timed out` | Jenkins took too long to unpack on first boot | Add `TimeoutStartSec=600` override (see Phase 2.4) |
| `Couldn't find any revision to build` | Branch set to `*/master` but repo uses `main` | Change branch to `*/main` in Source Code Management |
| `Authentication failed` | Wrong Azure credentials or SCM auth disabled | Enable SCM Basic Auth in Azure → General Settings |
| `The publishing username has already been taken` | Azure usernames are globally unique | Use the Azure CLI to set a unique username |
| `Could not resolve host: 123@devops...` | Password contained `@` symbol, breaking the URL | Use an alphanumeric-only password (no special characters) |
| `not a full refname` | New Azure repo doesn't recognize shorthand branch | Use `HEAD:refs/heads/main` instead of `HEAD:main` |

---

## 💡 Key Learnings

- Jenkins on Linux is a **production-grade** tool used by large enterprises — setting it up manually on RHEL gives you hands-on experience that GUI-based tools don't.
- Azure's web portal can sometimes be unreliable for credential management. The **Azure CLI (`az` commands) is always more reliable** and is the preferred method for automation.
- CI/CD is not just about saving time — it removes **human error** from deployments. Every push is done exactly the same way, every time.
- Special characters in passwords (`@`, `#`, `!`) **break URL-embedded credentials**. Always use alphanumeric passwords in automation scripts.

---

## 📂 Project Structure

```
your-github-repo/
│
└── index.html        ← Your entire website (single file)
```

---

## 📝 A Note for Anyone Who Tries This

Every error in the troubleshooting table above was a real wall that had to be broken through — the wrong Java version, the locked Azure portal, the password with the wrong character. None of it was in any tutorial.

If you are going through this guide and something breaks in a way that isn't listed here, that's not a failure — that's just how DevOps actually works. Debug it, fix it, and that fix becomes yours to keep.

The troubleshooting table in this README grew one error at a time. Maybe yours will too. 🙂

---

<div align="center">

**Built with patience, a lot of terminal output, and one very stubborn RHEL firewall.** 🐧

*GitHub → Jenkins → Azure — One push at a time.*

</div>
