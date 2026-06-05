# Grafana on Minikube — AWS CloudFormation Automation

> **COMM065 Cloud Security · University of Surrey · MSc Cyber Security**  
> Fully automated deployment of Grafana on Kubernetes (minikube) on an AWS EC2 instance, provisioned end-to-end with CloudFormation.

---

## Overview

This project automates the deployment of a Grafana monitoring stack running inside a Kubernetes cluster (minikube) on an AWS EC2 instance. The entire deployment — from network provisioning to a live Grafana dashboard connected to Google Sheets — runs without any manual commands after the CloudFormation stacks are created.

The automation is structured in five layers, each building on the last:

```
lab-network.yml  (CloudFormation)
    └── VPC, subnet, internet gateway, route table

lab-grafana.yml  (CloudFormation)
    └── EC2 instance (Amazon Linux 2023, t2.medium)
        └── cloud-init UserData
            ├── /tmp/grafana.yml       — Kubernetes manifests
            ├── /tmp/jwt.json          — Google Cloud service account key
            ├── /tmp/setup-grafana.py  — Grafana HTTP API provisioner
            └── /tmp/grafanainstall.sh — Docker + minikube + port-forward script
                └── Runs as ec2-user (minikube requires non-root)
```

---

## Architecture

| Layer | File | What it does |
|---|---|---|
| 1 — Network | `lab-network.yml` | Provisions VPC (10.0.0.0/16), public subnet, internet gateway, and route table. Exports `SubnetID` and `VPCID` for the application stack. |
| 2 — EC2 | `lab-grafana.yml` | Provisions a `t2.medium` EC2 instance with security groups for SSH (22), Grafana (3000), and Kubernetes dashboard (8081). |
| 3 — cloud-init | UserData in `lab-grafana.yml` | Writes all automation files to `/tmp` using `tee` heredocs, then executes the install script as `ec2-user`. |
| 4 — Bash script | `/tmp/grafanainstall.sh` | Installs Docker, downloads and starts minikube, applies the Kubernetes manifest, enables dashboard and metrics addons, and sets up port-forwarding. |
| 5 — Kubernetes | `/tmp/grafana.yml` | Defines the Grafana `Deployment`, `Service` (NodePort), and `PersistentVolumeClaim`. Includes `GF_INSTALL_PLUGINS` to auto-install the Google Sheets datasource plugin. |

A Python script (`setup-grafana.py`) runs at the end of cloud-init to provision the Google Sheets datasource and Heat Pump dashboard automatically via the Grafana HTTP API — no manual setup required.

---

## Prerequisites

- AWS Academy Learner Lab (or any AWS account with CloudFormation access)
- The `vockey` key pair must exist in your AWS region
- The network stack **must be named `lab-network`** — the application stack imports values using that prefix

---

## Deployment

### 1. Deploy the network stack

In the AWS CloudFormation console, create a new stack using `lab-network.yml`.

- **Stack name:** `lab-network` *(this name is mandatory)*

Wait for status `CREATE_COMPLETE` before proceeding.

### 2. Deploy the application stack

Create a second stack using `lab-grafana.yml`.

- **Stack name:** `lab-grafana`
- Leave all parameters at their defaults

### 3. Wait for cloud-init to complete

Cloud-init takes approximately **5–7 minutes** after `CREATE_COMPLETE`. You can monitor progress via EC2 Instance Connect:

```bash
tail -f /var/log/cloud-init-output.log
```

### 4. Access the services

Once cloud-init completes, retrieve the public IP from the CloudFormation Outputs tab or EC2 console:

| Service | URL |
|---|---|
| Grafana | `http://<PUBLIC_IP>:3000` |
| Kubernetes Dashboard | `http://<PUBLIC_IP>:8081` |

Default Grafana credentials: `admin` / `admin` (you will be prompted to change on first login).

The **Heat Pump dashboard** and **Google Sheets datasource** are provisioned automatically — they will be visible immediately after login.

---

## Key Technical Decisions

**`sg docker -c '...'` instead of `newgrp docker`**  
`newgrp` opens an interactive shell and hangs cloud-init indefinitely. `sg docker -c '...'` executes commands with the docker group active without blocking.

**`su - ec2-user -c` instead of `su - -c`**  
Minikube refuses to run under root with the Docker driver. Running as `ec2-user` is required.

**Quoted heredocs (`<<'EOF'`, `<<'JWTEOF'`)**  
Single-quoted heredoc tags prevent bash from expanding `$` characters inside the Python script and JSON credentials, which would corrupt them.

**`tee` over `>`**  
Using `tee` echoes file contents to stdout, making them visible in `/var/log/cloud-init-output.log` for verification and debugging.

**Python stdlib only (`urllib.request`)**  
The Grafana provisioner uses no third-party packages, so no `pip install` step is needed during cloud-init.

---

## Security Considerations

This project is a coursework deployment and makes several deliberate trade-offs for simplicity:

| Finding | Risk | Mitigation (production) |
|---|---|---|
| Service account key embedded in CloudFormation | Credential exposure in source | AWS Secrets Manager or SSM Parameter Store SecureString |
| Security groups open to `0.0.0.0/0` | Broad ingress exposure | Restrict to known IP ranges |
| Default `admin:admin` Grafana credentials | Trivial account takeover | Set strong password at first login |
| `grafana:latest` image tag | Unpinned version, supply chain risk | Pin to a specific digest |
| No IMDSv2 enforcement | SSRF metadata exposure | Enforce IMDSv2 via instance metadata options |
| Unencrypted EBS root volume | Data exposure at rest | Enable EBS encryption |
| No VPC Flow Logs | No network audit trail | Enable Flow Logs to S3 or CloudWatch |

---

## Repository Structure

```
.
├── lab-network.yml      # CloudFormation — VPC, subnet, internet gateway
└── lab-grafana.yml      # CloudFormation — EC2, UserData, security group
```

The Kubernetes manifest (`grafana.yml`), bash install script (`grafanainstall.sh`), and Python provisioner (`setup-grafana.py`) are all embedded inside the UserData section of `lab-grafana.yml`.

---

## Technologies Used

- **AWS CloudFormation** — infrastructure as code
- **Amazon EC2** — Amazon Linux 2023, t2.medium
- **cloud-init** — boot-time automation via UserData
- **Docker** — container runtime for minikube
- **minikube** — single-node Kubernetes cluster
- **Kubernetes** — Grafana deployment, service, PVC
- **Grafana** — monitoring and visualisation
- **Google Sheets API** — datasource via service account JWT
- **Python 3** (stdlib) — Grafana HTTP API automation

---

## Author

**Maaz Husain**  
MSc Cyber Security, University of Surrey  
