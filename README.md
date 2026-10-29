# Wanderlust: Automated DevSecOps CI/CD Pipeline on Kubernetes (kind) in GitHub Codespaces

A complete, working DevSecOps pipeline that runs on **one GitHub Codespace** with **no cloud account (no AWS/EKS)**.

When a developer pushes code and starts the pipeline, everything after that is automatic:
security scans, code-quality checks, Docker image build, push to Docker Hub, GitOps deployment to Kubernetes through Argo CD, an automated email with the build log, and live monitoring in Grafana.

> **Credit:** This project is based on the open-source [Wanderlust Mega Project](https://github.com/DevMadhup/Wanderlust-Mega-Project) by Madhup Pandey (DevMadhup), which was written for AWS EKS with separate Jenkins master/worker servers. This repository adapts it to run on a single GitHub Codespace using a kind cluster.

---

## Table of contents

1. [What this project does](#1-what-this-project-does)
2. [Architecture](#2-architecture)
3. [Tools and tested versions](#3-tools-and-tested-versions)
4. [Prerequisites](#4-prerequisites)
5. [Step-by-step deployment](#5-step-by-step-deployment)
6. [Daily start and stop](#6-daily-start-and-stop-after-the-codespace-restarts)
7. [Disk space management](#7-disk-space-management)
8. [Troubleshooting](#8-troubleshooting)
9. [Known limitations](#9-known-limitations)
10. [Appendix A to D: files you need](#appendix-a-ci-jenkinsfile-root-jenkinsfile)

---

## 1. What this project does

| Stage | Tool | What happens |
|---|---|---|
| CI | Jenkins | Pulls code from GitHub |
| CI | Trivy | Scans the files for vulnerabilities |
| CI | OWASP Dependency-Check | Checks libraries for known vulnerabilities |
| CI | SonarQube | Analyses code quality and applies a Quality Gate |
| CI | Docker | Builds the frontend and backend images |
| CI | Docker Hub | Stores the images |
| CD | Jenkins (second job) | Writes the new image tags into `kubernetes/*.yaml` and pushes to GitHub |
| CD | Argo CD | Detects the Git change and syncs it to the cluster (GitOps) |
| Run | Kubernetes (kind) | Replaces the old frontend and backend pods with new ones |
| Notify | Jenkins + Gmail | Emails the build result and log |
| Monitor | Prometheus + Grafana | Dashboards for cluster and pod health |

The application has four parts: React frontend, Node.js backend, MongoDB and Redis.

---

## 2. Architecture

```mermaid
flowchart LR
    Dev["Developer pushes code"] --> GH[("GitHub repository")]
    subgraph CI["Jenkins job: Wanderlust-CI"]
        A["Checkout"] --> B["Trivy scan"] --> C["OWASP Dependency-Check"] --> D["SonarQube analysis"] --> E["Quality Gate"] --> F["Docker build"] --> G["Push to Docker Hub"]
    end
    GH --> A
    G --> H["Jenkins job: Wanderlust-CD"]
    H -->|"updates image tags in kubernetes yaml files and pushes"| GH
    GH -->|"Argo CD detects the change"| I["Argo CD"]
    I --> K["kind cluster: frontend, backend, MongoDB, Redis"]
    K --> M["Prometheus and Grafana"]
    H -.->|"build result and log"| Mail["Gmail"]
```

Everything runs inside one Codespace:

| Component | How it runs | Port |
|---|---|---|
| Jenkins | Linux service (installed with apt) | 8080 |
| SonarQube | Docker container | 9000 |
| kind cluster | Docker container `wanderlust-control-plane` | n/a |
| Argo CD UI | `kubectl port-forward` | 8443 |
| Frontend | `kubectl port-forward` to the Kubernetes service | 31000 |
| Backend | `kubectl port-forward` to the Kubernetes service | 31100 |
| Grafana / Prometheus (optional) | `kubectl port-forward` | 3001 / 9091 |

---

## 3. Tools and tested versions

| Tool | Version used |
|---|---|
| GitHub Codespaces | 4-core, 16 GB RAM, 32 GB disk, Ubuntu 24.04 |
| Jenkins | LTS, installed from the official apt repository, Java 21 |
| SonarQube | 9.9.8 (`sonarqube:lts-community`) |
| Trivy | 0.74.0 |
| OWASP Dependency-Check | 13.0.0 (installed automatically by the Jenkins plugin) |
| Kubernetes (kind node image) | v1.37.0 |
| Argo CD | v3.5.4 (from the `stable` manifests) |
| kube-prometheus-stack (Helm chart) | 92.3.0 |
| Application runtime | Node.js 21 (see the Dockerfiles) |

Newer versions will probably work, but these are the ones this guide was tested with.

---

## 4. Prerequisites

### Accounts

| Account | Why you need it |
|---|---|
| GitHub | Repositories, Codespaces, token for Jenkins |
| Docker Hub | Stores the built images. Create an **access token** (Account Settings, Security, New Access Token, permission Read and Write) |
| Gmail | Sends build emails. Turn on 2-Step Verification, then create an **App password** at https://myaccount.google.com/apppasswords |
| NVD API key (free) | Lets OWASP Dependency-Check download its vulnerability database. Request one at https://nvd.nist.gov/developers/request-an-api-key and confirm it from the email |

### Machine size (important)

| Resource | Needed | Why |
|---|---|---|
| Codespace type | **4-core, 16 GB RAM** | On the 2-core (8 GB) machine only about 1.8 GB of RAM was free once the cluster, Argo CD, SonarQube and Jenkins were running |
| Disk | 32 GB (fixed for 2-core and 4-core) | The disk is the tightest resource. See [Disk space management](#7-disk-space-management) |

GitHub's free plan includes 120 core-hours per month (check your own numbers at https://github.com/settings/billing). A 4-core Codespace uses 4 core-hours per hour, so that is about 30 hours of runtime. **Stop the Codespace when you are not using it.**

Plan for roughly 6 hours for a first complete setup.

### Placeholders used in this guide

Replace these every time you see them:

| Placeholder | Replace with |
|---|---|
| `YOUR_GITHUB_USERNAME` | Your GitHub username |
| `YOUR_DOCKERHUB_USERNAME` | Your Docker Hub username (lowercase) |
| `YOUR_GITHUB_EMAIL` | The email on your GitHub account |
| `YOUR_EMAIL` | Your Gmail address (sender and receiver of build emails) |

---

## 5. Step-by-step deployment

Run all commands in the **Codespace terminal** unless a step says otherwise.

### Step 1: Fork the repositories and create the Codespace

1. Fork the project: https://github.com/DevMadhup/Wanderlust-Mega-Project
2. Fork the Jenkins Shared Library: https://github.com/muhammad-hussian-zahid/Jenkins_SharedLib
   (The Jenkinsfiles call steps such as `code_checkout`, `trivy_scan`, `owasp_dependency`, `sonarqube_analysis`, `sonarqube_code_quality`, `docker_build` and `docker_push`. They are defined in this library.)
3. Open **your fork** of the project on GitHub, click **Code**, then **Codespaces**, then the **...** menu, then **New with options**. Choose machine type **4-core** and click **Create codespace**.
4. When the terminal opens, check the machine:

```bash
nproc
free -h
df -h /workspaces
```

Expected: `4`, about 15 GiB total memory, and a 32 GB disk.

### Step 2: Check the tools that Codespaces provides

```bash
docker --version
kubectl version --client
gh --version
helm version || curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

The last line installs Helm only if it is missing.

### Step 3: Configure the project for your own accounts

Work inside your fork:

```bash
cd /workspaces/Wanderlust-Mega-Project
```

**3.1 Set the application URLs.** The frontend is built with the backend URL inside it, so these must be correct **before** the images are built:

```bash
FRONTEND_URL="https://${CODESPACE_NAME}-31000.${GITHUB_CODESPACES_PORT_FORWARDING_DOMAIN}"
BACKEND_URL="https://${CODESPACE_NAME}-31100.${GITHUB_CODESPACES_PORT_FORWARDING_DOMAIN}"
echo "$FRONTEND_URL"
echo "$BACKEND_URL"
```

**3.2 Write the URLs and a new secret into the env files:**

```bash
sed -i "s|^VITE_API_PATH=.*|VITE_API_PATH=\"$BACKEND_URL\"|" frontend/.env.docker
sed -i "s|^FRONTEND_URL=.*|FRONTEND_URL=\"$FRONTEND_URL\"|" backend/.env.docker
sed -i "s|^JWT_SECRET=.*|JWT_SECRET=$(openssl rand -hex 64)|" backend/.env.docker
```

Check the result (leave `MONGODB_URI` and `REDIS_URL` unchanged):

```bash
cat frontend/.env.docker
cat backend/.env.docker
```

> Your fork is public, so this `JWT_SECRET` will be public. Use a throw-away secret for this demo, never one you use elsewhere.

**3.3 Point the Kubernetes manifests to your own images:**

```bash
sed -i "s|image: .*|image: YOUR_DOCKERHUB_USERNAME/wanderlust-backend-beta:v1|"  kubernetes/backend.yaml
sed -i "s|image: .*|image: YOUR_DOCKERHUB_USERNAME/wanderlust-frontend-beta:v1|" kubernetes/frontend.yaml
grep -n "image" kubernetes/backend.yaml kubernetes/frontend.yaml
```

Expected: one `image:` line in each file with your username, and `imagePullPolicy: IfNotPresent` below it.

The image names `wanderlust-backend-beta` and `wanderlust-frontend-beta` must match the names in both Jenkinsfiles. The CD job finds the image line by this name and only replaces the tag after it.

**3.4 Replace the two Jenkinsfiles** with the versions in this README:

- Replace the content of `Jenkinsfile` with [Appendix A](#appendix-a-ci-jenkinsfile-root-jenkinsfile).
- Replace the content of `GitOps/Jenkinsfile` with [Appendix B](#appendix-b-cd-jenkinsfile-gitopsjenkinsfile).

Replace every placeholder inside them. The original versions call AWS scripts (`Automations/*.sh`) and a Jenkins node labelled `Node`, which do not exist here. The `Automations/` folder is not used in this setup.

**3.5 Commit and push** (Jenkins reads everything from GitHub, so unpushed changes are not used):

```bash
git add backend/.env.docker frontend/.env.docker kubernetes/backend.yaml kubernetes/frontend.yaml Jenkinsfile GitOps/Jenkinsfile
git commit -m "Configure for Codespaces"
git push origin main
```

### Step 4: Create the Kubernetes cluster (kind)

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

kind create cluster --name wanderlust
kubectl get nodes
```

Expected: one node with status `Ready`. The Docker container is named `wanderlust-control-plane`.

> Do **not** use `kubernetes/kind-config.yaml`. It publishes ports 31000 and 31100 on the host, which clashes with the `kubectl port-forward` used in this guide. This guide reaches the app through port-forwarding instead.

The cluster downloads the application images from Docker Hub, so your Docker Hub repositories must be **public** (Step 11).

### Step 5: Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w
```

Wait until all pods are `Running` (about 2 to 4 minutes), then press `Ctrl+C`.

Install the Argo CD command-line tool:

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
argocd version --client
```

Open the Argo CD API and log in:

```bash
nohup kubectl port-forward --address 0.0.0.0 -n argocd svc/argocd-server 8443:443 > /tmp/pf-argocd.log 2>&1 &
sleep 3
ARGO_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "$ARGO_PASSWORD"
argocd login localhost:8443 --insecure --username admin --password "$ARGO_PASSWORD"
```

The UI is on port **8443**. Open the **Ports** tab in VS Code and click the globe icon next to 8443. The UI uses HTTPS, so if the page does not load, right-click the port, choose **Change Port Protocol**, then **HTTPS**. Log in with `admin` and the password printed above. Keep this port **Private**.

### Step 6: Install Jenkins and Trivy

**6.1 Install Jenkins** from the official repository. The repository signing key was replaced in 2026, so use the `2026` key. Older guides that use the `2023` key fail with a "not signed" error.

```bash
sudo apt-get update
sudo apt-get install -y fontconfig openjdk-21-jre wget gnupg apt-transport-https

sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install -y jenkins
```

Start it and check:

```bash
sudo service jenkins start
sudo service jenkins status
```

**6.2 Let Jenkins use Docker.** Jenkins runs as the user `jenkins`, which must be allowed to use the Docker socket:

```bash
sudo usermod -aG docker jenkins
sudo service jenkins restart
sudo su -s /bin/bash jenkins -c 'docker ps'
```

Expected: the last command lists your running containers. If it says `permission denied`, run `sudo chmod 666 /var/run/docker.sock` and test again (this setting is lost whenever the Codespace restarts).

> Do not use `sudo -u jenkins ...`. It asks for a password that does not exist. Use `sudo su -s /bin/bash jenkins -c '...'` as shown above.

**6.3 Install Trivy** (the Jenkins pipeline calls it from the machine):

```bash
sudo apt-get install -y wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy
trivy --version
```

### Step 7: Start SonarQube

```bash
docker run -d --name SonarQube-Server --restart unless-stopped -p 9000:9000 \
  -e SONAR_SEARCH_JAVAADDITIONALOPTS="-Dnode.store.allow_mmap=false" \
  -e SONAR_WEB_JAVAOPTS="-Xmx512m" \
  -e SONAR_CE_JAVAOPTS="-Xmx512m" \
  -e SONAR_SEARCH_JAVAOPTS="-Xmx512m" \
  sonarqube:lts-community
```

What the extra options do: `allow_mmap=false` avoids a kernel setting that cannot be changed in a Codespace, and the `-Xmx512m` options keep SonarQube from using several GB of RAM.

Wait 2 to 3 minutes, then check:

```bash
curl -s localhost:9000/api/system/status
```

Expected: `"status":"UP"`. While it says `STARTING`, wait longer.

> This container has no volume. Its data (projects, tokens) lives inside the container. **Never delete or recreate `SonarQube-Server`**, or you lose them.

### Step 8: Open Jenkins and SonarQube

Open the **Ports** tab in VS Code. Ports **8080** (Jenkins) and **9000** (SonarQube) should be listed. If one is missing, click **Add Port** and add it. Keep both **Private**. Click the globe icon next to each port to open it.

**SonarQube:** log in with `admin` / `admin` and set a new password when asked. Then go to **My Account, Security, Generate Tokens**, name the token `jenkins`, click **Generate**, and copy the token. You will need it in Step 9.

### Step 9: Configure Jenkins

**9.1 Unlock Jenkins.** In the terminal:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Paste the password into the Jenkins page, choose **Install suggested plugins**, and create your admin user. Accept the default Jenkins URL.

**9.2 Install plugins.** Go to **Manage Jenkins, Plugins, Available plugins**, and install:

- SonarQube Scanner
- Docker Pipeline
- Pipeline Stage View
- Email Extension
- OWASP Dependency-Check

Restart Jenkins if it asks.

**9.3 Add credentials.** Go to **Manage Jenkins, Credentials, System, Global credentials (unrestricted), Add Credentials**:

| ID | Kind | Values |
|---|---|---|
| `Github-cred` | Username with password | Username: your GitHub username. Password: a GitHub **personal access token (classic)** with the `repo` scope |
| Docker Hub credential | Username with password | Username: your Docker Hub username. Password: your Docker Hub **access token** |
| `sonar-token` | Secret text | The SonarQube token from Step 8 |
| `nvd-api-key` | Secret text | Your NVD API key |

The ID of the Docker Hub credential must match the one used inside the shared library. Open `vars/docker_push.groovy` in your `Jenkins_SharedLib` fork, look for `credentialsId`, and use exactly that ID.

The ID `Github-cred` is case-sensitive and must match `Github-cred` in the CD Jenkinsfile.

**9.4 Configure tools.** Go to **Manage Jenkins, Tools**:

| Section | Name (exact) | Setting |
|---|---|---|
| SonarQube Scanner installations | `Sonar` | Install automatically (latest version) |
| Dependency-Check installations | `OWASP` | Install automatically (from github.com) |

**9.5 Connect Jenkins to SonarQube.** Go to **Manage Jenkins, System**, find **SonarQube servers**, and click **Add SonarQube**:

| Field | Value |
|---|---|
| Name | `Sonar` |
| Server URL | `http://localhost:9000` |
| Server authentication token | Select `sonar-token` |

Click **Save**. If the token field stays on `- none -`, the pipeline fails later with `Not authorized` (401).

**9.6 Add the shared library.** On the same **Manage Jenkins, System** page, find **Global Trusted Pipeline Libraries** and click **Add**:

| Field | Value |
|---|---|
| Name | `Shared` (exactly this, the Jenkinsfiles use `@Library('Shared')`) |
| Default version | `main` |
| Retrieval method | Modern SCM, then Git |
| Project repository | `https://github.com/YOUR_GITHUB_USERNAME/Jenkins_SharedLib.git` |

In your library fork, the OWASP step must pass the NVD key. Edit `vars/owasp_dependency.groovy` to match [Appendix C](#appendix-c-shared-library-file-varsowasp_dependencygroovy), then commit and push to the `main` branch of the library.

**9.7 Configure email.** On **Manage Jenkins, System**:

1. **Jenkins Location:** set **Jenkins URL** to your Codespace address so links in the emails work. Print it with:

   ```bash
   echo "https://${CODESPACE_NAME}-8080.${GITHUB_CODESPACES_PORT_FORWARDING_DOMAIN}/"
   ```

   Set **System Admin e-mail address** to your Gmail address.

2. **Extended E-mail Notification:**

   | Field | Value |
   |---|---|
   | SMTP server | `smtp.gmail.com` |
   | SMTP Port | `465` |
   | Use SSL | Checked |
   | Credentials | Add a **Username with password** credential: your Gmail address and the Google **App password** |

3. Use **Test configuration by sending test e-mail** to confirm it works. Click **Save**.

### Step 10: Create the two Jenkins jobs

**10.1 CI job.** **New Item**, name `Wanderlust-CI`, type **Pipeline**, **OK**. Then:

- Tick **This project is parameterized** and add two **String Parameter** entries named exactly `FRONTEND_DOCKER_TAG` and `BACKEND_DOCKER_TAG`. (Without this, the **Build with Parameters** button does not appear before the first run.)
- Under **Pipeline**: Definition **Pipeline script from SCM**, SCM **Git**, Repository URL `https://github.com/YOUR_GITHUB_USERNAME/Wanderlust-Mega-Project.git`, Branch `*/main`, Script Path `Jenkinsfile`.
- **Save**.

**10.2 CD job.** **New Item**, name **exactly** `Wanderlust-CD`, type **Pipeline**. The CI pipeline starts it by this name. Add the same two string parameters. Under **Pipeline**: the same repository and branch, but Script Path `GitOps/Jenkinsfile`. **Save**.

### Step 11: Run the CI pipeline for the first time

1. Open **Wanderlust-CI** and click **Build with Parameters**.
2. Enter `v1` for both `FRONTEND_DOCKER_TAG` and `BACKEND_DOCKER_TAG`, then click **Build**.
3. Open the build and watch **Console Output**, or use **Stage View**.

Stages, in order: Workspace cleanup, Git checkout, Trivy scan, OWASP Dependency-Check, SonarQube analysis, SonarQube Quality Gate, Docker build, Docker push.

Timing: the **first** OWASP run downloads the whole vulnerability database and can take **10 to 30 minutes**. Later runs reuse it and take about 3 to 4 minutes.

When CI succeeds it starts **Wanderlust-CD** automatically. On this first run, CD finds the tags already set to `v1` and prints `Nothing to commit`. That is correct.

**After the push, make your images public.** Open https://hub.docker.com, then your repositories `wanderlust-backend-beta` and `wanderlust-frontend-beta`, and confirm each shows **Public** (Settings, Visibility). The cluster cannot pull private images.

You should also receive the build email in Gmail.

### Step 12: Deploy the application with Argo CD

Create the Argo CD application. It watches the `kubernetes` folder of your fork and deploys it to the namespace `wanderlust` in this cluster:

```bash
argocd app create wanderlust \
  --repo https://github.com/YOUR_GITHUB_USERNAME/Wanderlust-Mega-Project.git \
  --revision main \
  --path kubernetes \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace wanderlust \
  --directory-exclude 'kind-config.yaml' \
  --sync-policy automated \
  --auto-prune \
  --sync-option CreateNamespace=true
```

Notes:

- `https://kubernetes.default.svc` is the cluster Argo CD itself runs in. It is already registered, so you do **not** run `argocd cluster add`.
- `--directory-exclude 'kind-config.yaml'` is required. That file is a kind setup file, not a Kubernetes resource, and without the exclude the whole sync fails.

Check the result:

```bash
argocd app get wanderlust
kubectl get pods -n wanderlust
```

Expected: status `Synced` and `Healthy`, with four pods `Running` (backend, frontend, mongo, redis). If the status is `OutOfSync` or `Missing`, wait a minute, then run `argocd app sync wanderlust`.

**Expose the application.** Create the port-forward script from [Appendix D](#appendix-d-port-forward-script-pfsh) and start it. First stop the manual Argo CD port-forward from Step 5, because the script starts its own:

```bash
pkill -f "kubectl port-forward"
chmod +x ~/pf.sh
nohup ~/pf.sh > /tmp/pf.log 2>&1 &
sleep 5
curl -sI localhost:31000 | head -1
curl -s localhost:31100/ | head -c 100; echo
```

**Make the application ports public.** The browser calls the backend directly, so both ports must be public. In the **Ports** tab, right-click **31000**, choose **Port Visibility, Public**, and do the same for **31100**. (Alternatively: `gh codespace ports visibility 31000:public 31100:public -c $CODESPACE_NAME`, once both ports are listed.) Keep 8080, 9000 and 8443 **Private**.

Open the application:

```bash
echo "$FRONTEND_URL"
```

If the variable is empty (new terminal), print it again with:
`echo "https://${CODESPACE_NAME}-31000.${GITHUB_CODESPACES_PORT_FORWARDING_DOMAIN}"`

### Step 13: Add monitoring with Prometheus and Grafana (optional)

This is the heaviest optional part. Check that you have **at least 4 GB free** first (`df -h /workspaces`).

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kps prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set alertmanager.enabled=false \
  --set kubeEtcd.enabled=false \
  --set kubeScheduler.enabled=false \
  --set kubeControllerManager.enabled=false \
  --set kubeProxy.enabled=false \
  --set prometheus.prometheusSpec.retention=2d \
  --set prometheus.prometheusSpec.resources.requests.memory=400Mi \
  --set grafana.adminPassword='CHOOSE_A_PASSWORD'

kubectl get pods -n monitoring -w
```

Why these options: kind does not expose etcd, scheduler, controller-manager or kube-proxy metrics (they would show as permanently down), and the small retention and memory request keep disk and RAM use low. With no storage configured, Prometheus data is lost if its pod restarts, which is fine for a demo.

When all pods are `Running`, press `Ctrl+C` and open the dashboards:

```bash
nohup kubectl port-forward svc/kps-grafana -n monitoring 3001:80 > /tmp/pf-grafana.log 2>&1 &
nohup kubectl port-forward svc/kps-kube-prometheus-stack-prometheus -n monitoring 9091:9090 > /tmp/pf-prom.log 2>&1 &
```

Open ports **3001** (Grafana) and **9091** (Prometheus) from the **Ports** tab (keep them Private). Log in to Grafana with user `admin` and the password you chose. Useful dashboards:

- Kubernetes / Compute Resources / Namespace (Pods): choose namespace `wanderlust`
- Kubernetes / Compute Resources / Pod
- Node Exporter / Nodes

To remove the monitoring stack later: `helm uninstall kps -n monitoring && kubectl delete namespace monitoring`.

### Step 14: Test the full automation

1. Make a small change in the frontend or backend source, then commit and push it to `main`.
2. In Jenkins, open **Wanderlust-CI**, click **Build with Parameters**, and enter a **new tag** for both fields, for example `v2`. Use a new tag every time: if the tag does not change, nothing is rolled out.
3. Wait for CI to succeed. It starts **Wanderlust-CD**, which commits `Updated image tags` to your repository.
4. In the Argo CD UI, click **Refresh** on the `wanderlust` application. Without a click, Argo CD checks Git about every 3 minutes.
5. Watch the cluster replace the pods:

   ```bash
   kubectl get pods -n wanderlust -w
   ```

   You will see new pods in `ContainerCreating` and old pods in `Terminating`.
6. Confirm the new tag is running:

   ```bash
   kubectl get deploy -n wanderlust -o wide | grep -E "backend|frontend"
   ```

7. Wait about 20 seconds (the port-forward script reconnects to the new pods) and refresh the application page.
8. Check Gmail for the build result email with the build log attached.

Jenkins does not start automatically when you push, because a Codespace cannot receive GitHub webhooks. You start CI manually. Everything after that is automatic.

---

## 6. Daily start and stop (after the Codespace restarts)

When a Codespace stops, all Docker containers and background processes stop. Your data is kept. To bring everything back:

```bash
docker start wanderlust-control-plane SonarQube-Server
sleep 30
kind export kubeconfig --name wanderlust
kubectl get pods -A

sudo service jenkins start
nohup ~/pf.sh > /tmp/pf.log 2>&1 &
```

Then:

1. Wait 2 to 3 minutes for SonarQube (`curl -s localhost:9000/api/system/status` shows `UP`).
2. In the **Ports** tab, check that 31000 and 31100 are **Public** again (they can reset to Private).
3. If you use monitoring, start its two port-forwards again (Step 13).
4. If `docker ps` fails inside Jenkins builds, run the Docker permission fix from Step 6.2.

**Stop the Codespace when you finish**, to save hours. Closing the browser tab does not stop it:

```bash
gh codespace stop -c $CODESPACE_NAME
```

---

## 7. Disk space management

The 32 GB disk is the biggest risk. A full disk stops Jenkins with `No space left on device`.

Check often:

```bash
df -h /workspaces
```

Free space safely **after** the Docker push stage has finished:

```bash
docker builder prune -af
docker exec wanderlust-control-plane crictl rmi --prune
sudo apt-get clean
```

Rules:

- Keep at least **5 to 6 GB free** before starting a CI run (the build uses about 3 to 4 GB temporarily).
- **Never** run `docker image prune -a` between a build and its push. It deletes the freshly built images and the push fails. Run it only when no build is running.
- **Never** run `docker volume prune` or `docker system prune --volumes`, and never delete the `SonarQube-Server` container.
- Do not delete the Dependency-Check data under `/var/lib/jenkins`. It holds the downloaded vulnerability database, and downloading it again takes 10 to 30 minutes.
- Remove the monitoring stack (Step 13) if you need space. It is the largest optional item inside the cluster.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Build with Parameters` is missing on the job page | Jenkins does not know the parameters before the first run | Add the two String Parameters in the job configuration (Step 10) |
| `No such library 'Shared'` | Library not configured, or name differs | Configure it exactly as in Step 9.6 |
| `Not authorized. Analyzing this project requires authentication` (SonarQube, 401) | The token is not attached to the Sonar server entry | **Manage Jenkins, System, SonarQube servers**: select `sonar-token`, then **Save** |
| OWASP stage fails with `Invalid API Key, length of 0` | The NVD key is not passed | Credential `nvd-api-key` must exist and the library file must match Appendix C |
| `permission denied ... /var/run/docker.sock` | The `jenkins` user is not in the `docker` group | Step 6.2, then restart Jenkins |
| `invalid tag "YOUR_DOCKERHUB_USERNAME/...": repository name must be lowercase` | A placeholder was left in the Jenkinsfile, or Jenkins reads an old copy | Replace it, commit and push. Check the job's Script Path and branch |
| `[sudo] password for codespace` after `sudo -u jenkins ...` | `sudo -u` needs a password that does not exist | Use `sudo su -s /bin/bash jenkins -c '...'` |
| `No space left on device` | Disk full | Section 7. Then restart Jenkins and run the build again |
| `denied` or `unauthorized` in the Docker push stage | Wrong Docker Hub credential ID or token | Use the ID found in `vars/docker_push.groovy`, and a token with Read and Write permission |
| Argo CD sync fails with `kind.x-k8s.io/Cluster` or "could not find the requested resource" | `kind-config.yaml` is being applied as a resource | `argocd app set wanderlust --directory-exclude 'kind-config.yaml'` then `argocd app sync wanderlust` |
| Pods show `ImagePullBackOff` | The image tag was never pushed, or the Docker Hub repository is private | Confirm the tag exists on Docker Hub and the repository is public |
| App page shows `HTTP ERROR 502` or "This page isn't working" | The port-forward died (pods were replaced) | Restart the script: `pkill -f "kubectl port-forward"` then `nohup ~/pf.sh > /tmp/pf.log 2>&1 &` |
| Argo CD UI stops opening | Its port-forward was stopped (for example by the `pkill` above) | Start `~/pf.sh` again, it includes Argo CD on 8443. Restart the Grafana/Prometheus forwards too |
| Page loads but shows errors or no data | Port 31100 is private, or the URLs in the env files do not match this Codespace | Make 31100 public. Compare the URLs in `.env.docker` with your `$CODESPACE_NAME`, then rebuild |
| `kubectl` cannot connect after a restart | Cluster container stopped | `docker start wanderlust-control-plane`, wait 30 seconds, `kind export kubeconfig --name wanderlust` |
| Backend pod in `CrashLoopBackOff` right after deploy | Backend started before MongoDB was ready | `kubectl delete pod -l app=backend -n wanderlust` (a new pod is created automatically) |
| `git push` returns `403` to a repository other than the Codespace's own | The built-in Codespace login covers only its own repository | Push using a personal access token: `git push https://YOUR_GITHUB_USERNAME:TOKEN@github.com/YOUR_GITHUB_USERNAME/REPO.git main` (never paste a real token into shared logs) |
| Gmail email fails to send | Normal Gmail password used | Use a Google **App password** (needs 2-Step Verification) |
| Links inside the build email do not open | The Jenkins URL is still `localhost` | Set the Jenkins URL as in Step 9.7 |

---

## 9. Known limitations

- **The Codespace name is part of the URLs.** If you create a **new** Codespace, redo Step 3.1 and 3.2 (URLs), rebuild the images and push again.
- **No automatic trigger on `git push`.** GitHub webhooks cannot reach a Codespace, so CI is started manually.
- **Port-forwarding, not a load balancer.** The app and Argo CD are reached through `kubectl port-forward`, which stops when the Codespace stops. The script reconnects after pod replacements.
- **Prometheus data is not persistent.** It is lost if the Prometheus pod restarts.
- **Secrets in a public fork.** `backend/.env.docker` is committed, so use a throw-away `JWT_SECRET`.
- **This is a learning and demo environment**, not a production cluster.

---

## Appendix A: CI Jenkinsfile (root `Jenkinsfile`)

Replace `YOUR_GITHUB_USERNAME` and `YOUR_DOCKERHUB_USERNAME`:

```groovy
@Library('Shared') _
pipeline {
    agent any

    environment{
        SONAR_HOME = tool "Sonar"
    }

    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Setting docker image for latest push')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: '', description: 'Setting docker image for latest push')
    }

    stages {

        stage("Workspace cleanup"){
            steps{
                script{
                    cleanWs()
                }
            }
        }

        stage('Git: Code Checkout') {
            steps {
                script{
                    code_checkout("https://github.com/YOUR_GITHUB_USERNAME/Wanderlust-Mega-Project.git","main")
                }
            }
        }

        stage("Trivy: Filesystem scan"){
            steps{
                script{
                    trivy_scan()
                }
            }
        }

        stage("OWASP: Dependency check"){
            steps{
                script{
                    owasp_dependency()
                }
            }
        }

        stage("SonarQube: Code Analysis"){
            steps{
                script{
                    sonarqube_analysis("Sonar","wanderlust","wanderlust")
                }
            }
        }

        stage("SonarQube: Code Quality Gates"){
            steps{
                script{
                    sonarqube_code_quality()
                }
            }
        }

        stage("Docker: Build Images"){
            steps{
                script{
                    dir('backend'){
                        docker_build("wanderlust-backend-beta","${params.BACKEND_DOCKER_TAG}","YOUR_DOCKERHUB_USERNAME")
                    }

                    dir('frontend'){
                        docker_build("wanderlust-frontend-beta","${params.FRONTEND_DOCKER_TAG}","YOUR_DOCKERHUB_USERNAME")
                    }
                }
            }
        }

        stage("Docker: Push to DockerHub"){
            steps{
                script{
                    docker_push("wanderlust-backend-beta","${params.BACKEND_DOCKER_TAG}","YOUR_DOCKERHUB_USERNAME")
                    docker_push("wanderlust-frontend-beta","${params.FRONTEND_DOCKER_TAG}","YOUR_DOCKERHUB_USERNAME")
                }
            }
        }
    }
    post{
        success{
            archiveArtifacts artifacts: '*.xml', allowEmptyArchive: true, followSymlinks: false
            build job: "Wanderlust-CD", parameters: [
                string(name: 'FRONTEND_DOCKER_TAG', value: "${params.FRONTEND_DOCKER_TAG}"),
                string(name: 'BACKEND_DOCKER_TAG', value: "${params.BACKEND_DOCKER_TAG}")
            ]
        }
    }
}
```

What changed from the original project file: `agent any` instead of the node label `Node`, the AWS "Exporting environment variables" stage is removed (the URLs are already set in the env files in Step 3), `allowEmptyArchive: true` was added, and the names are your own.

## Appendix B: CD Jenkinsfile (`GitOps/Jenkinsfile`)

Replace `YOUR_GITHUB_USERNAME`, `YOUR_GITHUB_EMAIL` and `YOUR_EMAIL`:

```groovy
@Library('Shared') _
pipeline {
    agent any

    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Frontend Docker tag of the image built by the CI job')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: '', description: 'Backend Docker tag of the image built by the CI job')
    }

    stages {
        stage("Workspace cleanup"){
            steps{
                script{
                    cleanWs()
                }
            }
        }

        stage('Git: Code Checkout') {
            steps {
                script{
                    code_checkout("https://github.com/YOUR_GITHUB_USERNAME/Wanderlust-Mega-Project.git","main")
                }
            }
        }

        stage('Verify: Docker Image Tags') {
            steps {
                script{
                    echo "FRONTEND_DOCKER_TAG: ${params.FRONTEND_DOCKER_TAG}"
                    echo "BACKEND_DOCKER_TAG: ${params.BACKEND_DOCKER_TAG}"
                }
            }
        }

        stage("Update: Kubernetes manifests"){
            steps{
                script{
                    dir('kubernetes'){
                        sh """
                            sed -i -e s/wanderlust-backend-beta.*/wanderlust-backend-beta:${params.BACKEND_DOCKER_TAG}/g backend.yaml
                        """
                    }

                    dir('kubernetes'){
                        sh """
                            sed -i -e s/wanderlust-frontend-beta.*/wanderlust-frontend-beta:${params.FRONTEND_DOCKER_TAG}/g frontend.yaml
                        """
                    }
                }
            }
        }

        stage("Git: Code update and push to GitHub"){
            steps{
                script{
                    withCredentials([gitUsernamePassword(credentialsId: 'Github-cred', gitToolName: 'Default')]) {
                        sh '''
                        git config user.email "YOUR_GITHUB_EMAIL"
                        git config user.name "YOUR_GITHUB_USERNAME"

                        echo "Checking repository status: "
                        git status

                        echo "Adding changes to git: "
                        git add .

                        echo "Commiting changes: "
                        git commit -m "Updated image tags" || echo "Nothing to commit"

                        echo "Pushing changes to github: "
                        git push https://github.com/YOUR_GITHUB_USERNAME/Wanderlust-Mega-Project.git main
                    '''
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                emailext attachLog: true,
                from: 'YOUR_EMAIL',
                subject: "Wanderlust Application has been updated and deployed - '${currentBuild.result}'",
                body: """
                    <html>
                    <body>
                        <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px;">
                            <p style="color: black; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                        </div>
                        <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px;">
                            <p style="color: black; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                        </div>
                        <div style="background-color: #87CEEB; padding: 10px; margin-bottom: 10px;">
                            <p style="color: black; font-weight: bold;">URL: ${env.BUILD_URL}</p>
                        </div>
                    </body>
                    </html>
            """,
            to: 'YOUR_EMAIL',
            mimeType: 'text/html'
            }
        }
    }
}
```

How it works: the two `sed` lines replace everything after the image name with the new tag, so the registry and username part of the `image:` line in the manifests (set in Step 3.3) stays as it is. `|| echo "Nothing to commit"` keeps the build green when you run it again with the same tag.

## Appendix C: Shared library file `vars/owasp_dependency.groovy`

In your `Jenkins_SharedLib` fork:

```groovy
def call(){
  dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'OWASP', nvdCredentialsId: 'nvd-api-key'
  dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
}
```

- `odcInstallation: 'OWASP'` is the tool name from Step 9.4.
- `nvdCredentialsId: 'nvd-api-key'` is the Secret text credential from Step 9.3. Without it the stage fails with `Invalid API Key, length of 0`.
- The `--disable...Audit` flags skip the extra Yarn and npm audit lookups.

The other library steps (`code_checkout`, `trivy_scan`, `sonarqube_analysis`, `sonarqube_code_quality`, `docker_build`, `docker_push`) are used unchanged.

## Appendix D: Port-forward script (`~/pf.sh`)

Create the file:

```bash
cat > ~/pf.sh <<'SCRIPT'
#!/bin/bash
fwd() { while true; do kubectl port-forward --address 0.0.0.0 -n "$1" "$2" "$3"; sleep 2; done; }
fwd wanderlust svc/frontend-service 31000:5173 &
fwd wanderlust svc/backend-service  31100:8080 &
fwd argocd     svc/argocd-server    8443:443   &
wait
SCRIPT
chmod +x ~/pf.sh
```

Start it in the background: `nohup ~/pf.sh > /tmp/pf.log 2>&1 &`

Each line keeps retrying, so when Argo CD replaces a pod the forward reconnects by itself within a few seconds.

Note: `pkill -f "kubectl port-forward"` stops **all** port-forwards, including Argo CD's and Grafana's. Always restart `~/pf.sh` afterwards.

---

## License

See the `LICENSE` file in this repository. The original project is by Madhup Pandey (DevMadhup).
