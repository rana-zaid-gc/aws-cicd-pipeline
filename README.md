# AWS CI/CD Pipeline — Automated Deployment to the Cloud

A CI/CD pipeline that automatically deploys a website to an **Nginx server running on AWS EC2** whenever code is pushed to GitHub. Jenkins detects the change, connects to the cloud server over SSH, pulls the latest code, and publishes it — turning a `git push` into a live update on a real internet-facing server.

This project evolved from an earlier **local** version (Jenkins + Nginx on a VirtualBox VM) into a **cloud deployment** on AWS — replacing the local VM with a real EC2 instance that has a public IP, reachable from anywhere.

🔗 **Live site:** ``

---

## Architecture

```
  ┌──────────┐    ┌──────────────────┐    ┌────────────────────────────┐
  │  GitHub  │───▶│  Jenkins (local) │───▶│        AWS EC2 (cloud)       │
  │  (code)  │    │   Poll SCM       │SSH │  git pull → copy to Nginx     │
  └──────────┘    └──────────────────┘    └────────────────────────────┘
                                                        │
                                                        ▼
                                              Live site on the internet
```

**Flow:** `git push` → Jenkins detects the new commit → connects to the EC2 server over SSH → pulls the latest code → copies it into Nginx's web root (`/var/www/html`) → site is live.

---

## Tech Stack

| Tool | Role |
|------|------|
| **AWS EC2** | Cloud server (Ubuntu) hosting the site with a public IP |
| **Jenkins** | CI/CD automation server that runs the pipeline |
| **Nginx** | Web server on the EC2 instance |
| **Git / GitHub** | Source control and pipeline trigger |
| **SSH** | Secure, key-based connection from Jenkins to EC2 |
| **Ubuntu** | Operating system on the EC2 instance |

---

## How It Works

The pipeline is defined in a `Jenkinsfile` (Pipeline-as-Code):

1. **Trigger** — Jenkins polls GitHub (Poll SCM, every ~2 min) and starts a build when it finds a new commit.
2. **Deploy to EC2** — Using a stored SSH credential, Jenkins connects to the EC2 instance and:
   - Pulls the latest code (`git pull`)
   - Copies the site files into Nginx's web root (`/var/www/html`)
3. **Serve** — Nginx serves the updated site instantly on port 80.

### Jenkinsfile

```groovy
pipeline {
    agent any
    stages {
        stage('Deploy to AWS EC2') {
            steps {
                sshagent(['vm-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@<EC2_IP> "
                            cd ~/app &&
                            git pull origin main &&
                            cp -r ~/app/*.html /var/www/html/ &&
                            echo 'Deployed to AWS successfully'
                        "
                    '''
                }
            }
        }
    }
}
```

---

## AWS Setup Summary

1. **Launched an EC2 instance** — Ubuntu, t3.micro (free tier), in the `eu-north-1` region.
2. **Configured the security group** (firewall) to allow inbound **SSH (22)** and **HTTP (80)**.
3. **Connected via SSH** using the EC2 key pair (`.pem`).
4. **Installed Nginx** on the server and confirmed it served the default page publicly.
5. **Set up an SSH key bridge** so the Jenkins user could deploy to EC2 without a password.
6. **Pointed the pipeline at the EC2 server's public IP** and automated it with Poll SCM.

---

## Challenges I Solved

- **Cross-machine deployment** — Jenkins runs locally while the site runs on AWS, so the pipeline had to deploy *across machines* over SSH. I set up key-based authentication from the Jenkins user to the EC2 instance and stored the private key as a Jenkins credential.
- **AWS security groups** — Learned that EC2 blocks all inbound traffic by default; the site only became reachable after explicitly opening port 80 (HTTP) and port 22 (SSH) in the security group.
- **Repo / Jenkinsfile path** — Hit an "Unable to find Jenkinsfile" error caused by the pipeline pointing at a repo without the `Jenkinsfile` at the expected path; fixed by ensuring the correct filename and branch.
- **Migrating from local to cloud** — Adapted the existing local pipeline (which deployed to a VirtualBox VM) to target a real cloud server, including swapping the deployment IP and SSH user (`ubuntu`).

---

## Screenshots

> Add your images to a `screenshots/` folder and update the paths below.

**EC2 instance running in AWS:**

<img width="1918" height="844" alt="image" src="https://github.com/user-attachments/assets/6519ebee-bb35-427b-8904-c9f949d3087e" />


**Jenkins build succeeded (green):**

<img width="814" height="417" alt="image" src="https://github.com/user-attachments/assets/ee8c36d3-c783-42a0-853a-ac4d90e147ea" />


**Live site served from AWS:**

<img width="1916" height="974" alt="image" src="https://github.com/user-attachments/assets/085c26d8-546d-4ed9-9b34-2c1aac16d0c7" />


---

## What I'd Improve for Production

- **Run Jenkins on the cloud too** (its own server) so GitHub webhooks reach it directly — enabling instant deploys with no polling or tunneling.
- **Use an Elastic IP** so the server's address never changes.
- **Restrict SSH** in the security group to my own IP instead of `0.0.0.0/0`.
- **Add HTTPS** with a free Let's Encrypt certificate.
- **Add a build/test stage** before deployment.

---

## What I Learned

- Provisioning and securing a cloud server on AWS EC2
- How security groups control inbound network access
- Deploying across machines with key-based SSH
- Adapting a local CI/CD pipeline to a real cloud environment
- The difference between a project that "runs on my laptop" and one that's genuinely internet-facing

---

*Part of my DevOps learning journey — the cloud evolution of my first CI/CD pipeline project.*
