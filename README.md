## 📦 Graded Project: CI-CD Pipeline Tool Setup

📁 **Project Directory Structure (Example):**

Here's how your project folder might look once everything is in place:

```
ci-cd-pipeline-bash-python/
├── images/
├── .gitignore
├── check_github.py
├── ci_cd_deploy.sh
├── index.html
├── README.md
├── requirements.txt
└── update_website.sh
```

Make sure all script references and paths in your deployment pipeline align with this structure.

### 🚀 1. GitHub Repository Setup

1. 🌐 Visit [GitHub.com](https://github.com)
2. ➕ Click the "+" icon > **New repository**
3. Fill in the details:
   - 📁 **Repository Name:** `ci-cd-pipeline-bash-python`
   - ✏️ **Description:** (Optional)
   - 🔓 **Visibility:** Public
4. ✅ Optionally check:
   - "Add a README file"
   - Add `.gitignore` (e.g., Python)
5. ✅ Click **Create repository**

---

### 🖥️ 2. AWS EC2 Setup with Nginx

To host your CI/CD-managed application, you need a Linux server where Nginx will act as the web server to serve your deployed files. We’ll use an AWS EC2 instance running Ubuntu 24.04 LTS to demonstrate the complete process, from provisioning to installation of necessary packages like Nginx.

#### 👥 EC2 Instance Configuration

🛠️ **Configuration**           | 📋 **Details**                                          | ⚠️ **Notes**                                |
----------------------------------|---------------------------------------------------------|---------------------------------------------|
Name and Tags                   | `ci-cd-sagar-b-10`                                      | Easy identification                         |
AMI Selection                   | Ubuntu Server 24.04 LTS (AMI ID: `ami-0e55dda80b595c5f5`)| Default user: `ubuntu`                      |
Instance Type                  | `t2.micro`                                              | Free Tier eligible                          |
Key Pair                        | `sagar-b10`                                             | Save `.pem` file securely                   |
Network Settings                | Default VPC/Subnet, Public IP enabled                  | SG: `my-web-app-sagar-b-10` (SSH/HTTP open) |
Storage                         | 8 GiB `gp3`                                             | Free Tier eligible                          |
Launch                          | Click **Launch Instance**                              | Review configuration                        |

![Ec2](/images/ec2.png)

---

### 🔗 Connect via SSH

```bash
ssh -i ./<pem_file>.pem ubuntu@<public_ip>
```

Replace `<pem_file>` and `<public_ip>` accordingly.

---

### 🛠️ Install Nginx on EC2

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

🔍 Verify: Visit `http://<your-ec2-public-ip>/` in browser.

---

### 🐍 3. Python Script: GitHub Commit Checker

To automate the detection of new commits pushed to your GitHub repository, we use a Python script that checks for the latest commit SHA using the GitHub API. If a new commit is found, it triggers an update in your deployment pipeline.

The script consists of four main functions:

- `get_latest_commit_sha()`: Queries the GitHub API to fetch the SHA of the latest commit on the specified branch.
- `read_stored_commit_sha()`: Reads the last known commit SHA from a local file.
- `write_commit_sha(sha)`: Writes the new SHA to the local file to keep track of updates.
- `main()`: Coordinates the process by comparing the latest SHA with the stored one and exiting accordingly.

Save as: `check_github.py`

```
import requests
import json
import os
import sys

# Configuration
REPO_OWNER = 'psagar-dev'
REPO_NAME = 'ci-cd-pipeline-bash-python'
BRANCH = 'main'
LAST_COMMIT_FILE = '/var/www/ci-cd-pipeline-bash-python/last_commit.txt'
GITHUB_API_URL = f"https://api.github.com/repos/{REPO_OWNER}/{REPO_NAME}/commits/{BRANCH}"

def get_latest_commit_sha():
    """
    Fetch the latest commit SHA from the GitHub repository.
    """
    try:
        response = requests.get(GITHUB_API_URL, timeout=10)
        response.raise_for_status()
        commit_data = response.json()
        return commit_data['sha']
    except requests.RequestException as e:
        print(f"❌ Error fetching latest commit: {e}")
        return None
    except KeyError:
        print("❌ Unexpected response format.")
        return None

def read_stored_commit_sha():
    """
    Read the stored commit SHA from the file, if it exists.
    """
    if os.path.isfile(LAST_COMMIT_FILE):
        try:
            with open(LAST_COMMIT_FILE, 'r') as file:
                return file.read().strip()
        except IOError as e:
            print(f"❌ Error reading stored commit file: {e}")
    return None

def write_commit_sha(sha):
    """
    Write the latest commit SHA to the file.
    """
    try:
        with open(LAST_COMMIT_FILE, 'w') as file:
            file.write(sha)
    except IOError as e:
        print(f"❌ Error writing commit SHA: {e}")

def main():
    """
    Main execution function to compare and update commit SHA.
    Exits with:
    - 0 if new changes are found
    - 1 if no changes or on error
    """
    latest_sha = get_latest_commit_sha()
    if not latest_sha:
        sys.exit(1)

    stored_sha = read_stored_commit_sha()

    if latest_sha != stored_sha:
        print("✅ New changes detected.")
        write_commit_sha(latest_sha)
        sys.exit(0)
    else:
        print("ℹ️ No new changes.")
        sys.exit(1)

if __name__ == "__main__":
    main()

```

### 🧩 4. Bash Script: Deploy Updates to Web Server
**📝 Note:** This script assumes the repository is being cloned to `/var/www/ci-cd-pipeline-bash-python` during the first setup. Make sure this directory is used consistently across all scripts and cron jobs.

This Bash script automates the deployment process by cloning (or updating) your GitHub repository and syncing it to the Nginx web root directory. It ensures the latest changes are available on the live server and restarts Nginx to reflect updates.

The script handles the following:

- Creates the deployment directory if it doesn’t exist.
- Clones the repo or fetches new changes.
- Ensures the commit tracking file exists.
- Uses `rsync` to sync files efficiently.
- Restarts Nginx to apply the updates.

Save as: `update_website.sh`

```bash
#!/bin/bash
REPO_URL="https://github.com/psagar-dev/ci-cd-pipeline-bash-python.git"
REPO_DIR="/var/www/ci-cd-pipeline-bash-python"
WEBSITE_DIR="/var/www/html"
BRANCH_NAME="main"
LAST_COMMIT="$REPO_DIR/last_commit.txt"

# Create repo dir if it doesn't exist
if [ ! -d "$REPO_DIR" ]; then
    echo "Creating repo directory: $REPO_DIR"
    mkdir -p "$REPO_DIR"
fi

cd "$REPO_DIR" || { echo "❌ Failed to enter $REPO_DIR"; exit 1; }

if git config --get remote.origin.url &>/dev/null; then
    git fetch origin
    git reset --hard "origin/$BRANCH_NAME"
else
    echo "Cloning repository into $REPO_DIR..."
    git clone $REPO_URL .
fi

if [ ! -f "$LAST_COMMIT" ]; then
    sudo touch "$LAST_COMMIT"
    echo "📄 Created: $LAST_COMMIT"
fi

rsync -av --delete $REPO_DIR/ $WEBSITE_DIR/
sudo systemctl restart nginx

echo "✅ Website updated successfully"
```

---

### ⏰ 5. Cron Job Setup

To ensure the deployment process runs automatically, we use a cron job to regularly check for changes in the GitHub repository. This setup allows the server to fetch updates every 5 minutes and apply them without manual intervention.

Here's how it works:

- The `ci_cd_deploy.sh` script is executed every 5 minutes via cron.
- It runs the Python script to detect changes.
- If a new commit is found, it triggers the Bash deployment script.

Save as: `ci_cd_deploy.sh`

```bash
#!/bin/bash
PROJECT_PATH="/var/www/ci-cd-pipeline-bash-python"

python3 "$PROJECT_PATH/check_github.py"

if [ $? -eq 0 ]; then
    echo "✅ Changes detected. Updating website..."
    bash "$PROJECT_PATH/update_website.sh"
else
    echo "ℹ️ No changes detected. No update needed."
fi
```

Set up cron job:

To edit your crontab (scheduler), run the following command in the terminal:

```bash
crontab -e
```

Add the following line to schedule the script to run every 5 minutes:

The command `/usr/bin/bash /var/www/ci-cd-pipeline-bash-python/ci_cd_deploy.sh` runs the deployment check.

The output (`stdout` and `stderr`) is redirected to `/var/log/ci_cd_pipeline.log` for logging and debugging purposes.

```
*/5 * * * * /usr/bin/bash /var/www/ci-cd-pipeline-bash-python/ci_cd_deploy.sh >> /var/log/ci_cd_pipeline.log 2>&1
```

### 📦 First-Time Project Setup

Follow these steps on your server:

```bash
cd /var/www
git clone https://github.com/psagar-dev/ci-cd-pipeline-bash-python.git
sudo chown -R $USER:$USER /var/log/ci_cd_pipeline.log
sudo chown -R $USER:$USER ci-cd-pipeline-bash-python
sudo chown -R $USER:$USER html
cd ci-cd-pipeline-bash-python
bash ./ci_cd_deploy.sh
```
📝 Initial Log Output
![First Time](/images/first-time.png)

#### 🔁 Auto Deployment Test
To test if auto-deployment is working:
1. Modify the `index.html` file with any text change.
2. Wait 5 minutes — a cron job runs every 5 minutes.
3. View the deployment log:

```bash
cat /var/log/ci_cd_pipeline.log
```
📄 Deployment Log

![Check Log](/images/check-log.png)

#### 🌐 Website View
Once deployed, your site should be accessible.
![WebSite View](/images/webiste-view.png)

## 📜 Project Information

### 📄 License Details
This project is released under the MIT License, granting you the freedom to:
- 🔓 Use in commercial projects
- 🔄 Modify and redistribute
- 📚 Use as educational material

## 📞 Contact

📧 Email: [Email Me](securelooper@gmail.com
)
🔗 LinkedIn: [LinkedIn Profile](https://www.linkedin.com/in/sagar-93-patel)  
🐙 GitHub: [GitHub Profile](https://github.com/psagar-dev)  

---

<div align="center">
  <p>Built with ❤️ by Sagar Patel</p>
</div>