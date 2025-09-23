# Wazuh Deployment (Kubernetes + GitHub Actions)

This repository automates the deployment of a **Wazuh SOC platform** (Manager, Indexer, Dashboard, and Agents) on Kubernetes using **GitHub Actions**.  
It works with clusters hosted on **Ronin, AWS, or on-prem** that are connected as **self-hosted GitHub runners**.

---

## ✅ Prerequisites

1. **GitHub Runner Connected to Your Platform**
   - Ensure your cluster nodes (Ronin VM, AWS EC2, or on-prem) are registered as **self-hosted GitHub Actions runners**.
   - Verify the runner has Kubernetes access:
     ```bash
     kubectl get nodes -o wide
     ```

2. **Sanity Check**
   - Run the workflow: **`01-sanity-check.yml`**
   - Confirms GitHub ↔ Kubernetes connectivity, runner health, and kubeconfig access.

---

## 🚀 Deployment Workflow Order

Run the workflows **in the following order**:

### 1. Sanity Check
**Workflow:** `01-sanity-check.yml`  
Check connectivity:
```bash
kubectl get nodes -o wide
kubectl -n wazuh get pods

2. Storage Setup

Workflow: 02-storage-wazuh.yml
Creates persistent volumes and PVCs for Wazuh components:

kubectl -n wazuh get pvc
kubectl -n wazuh get pv

3. Wazuh Core Deployment

Workflow: 03-wazuh-deployment.yml
Deploys the main SOC stack:

kubectl -n wazuh get pods -l app=wazuh-manager
kubectl -n wazuh get pods -l app=wazuh-indexer
kubectl -n wazuh get pods -l app=wazuh-dashboar

4. Passwordless Authentication (Manager)

Workflow: 04-passwordlesss-auth.yml
Configures the manager(s) for passwordless agent enrollment:

# Verify <auth> block
kubectl -n wazuh exec -it <manager-pod> -c wazuh-manager -- \
  sed -n "/<auth>/,/<\/auth>/p" /var/ossec/etc/ossec.conf

# Verify port 1515 open
kubectl -n wazuh exec -it <manager-pod> -c wazuh-manager -- \
  ss -tulpn | grep 1515


5. Agent Installation + Config

Workflow: 05-agent-install-config.yml
Deploys the Wazuh Agent as a DaemonSet on all nodes:

# Check agent pods
kubectl -n wazuh get pods -l app=wazuh-agent -o wide

# Tail agent logs
kubectl -n wazuh logs -f <agent-pod>

# Verify agent enrollment inside pod
kubectl -n wazuh exec -it <agent-pod> -- \
  grep "agent-auth" /var/ossec/logs/ossec.log | tail -n 10


Agents monitor:

/var/log/containers/*.log

/var/log/pods/*/*.log

/var/log/syslog

/var/log/messages

🔍 Deployment Flow Recap

Run 01-sanity-check.yml → Confirm connectivity

Run 02-storage-wazuh.yml → Provision storage

Run 03-wazuh-deployment.yml → Deploy Wazuh core components

Run 04-passwordlesss-auth.yml → Enable passwordless enrollment on managers

Run 05-agent-install-config.yml → Deploy agents and start log shipping

🛠 Troubleshooting

If something fails, use these commands to investigate:

🔹 Manager Side
# Check manager pods
kubectl -n wazuh get pods -l app=wazuh-manager -o wide

# View <auth> block in ossec.conf
kubectl -n wazuh exec -it <manager-pod> -c wazuh-manager -- \
  sed -n "/<auth>/,/<\/auth>/p" /var/ossec/etc/ossec.conf

# Verify authd (1515) is listening
kubectl -n wazuh exec -it <manager-pod> -c wazuh-manager -- \
  ss -tulpn | grep 1515

# List registered agents
kubectl -n wazuh exec -it <manager-pod> -c wazuh-manager -- \
  /var/ossec/bin/agent_control -l

# Remove a stale/duplicate agent
kubectl -n wazuh exec -it <manager-pod> -c wazuh-manager -- \
  /var/ossec/bin/manage_agents -r <AGENT_ID>

🔹 Agent Side
# Check agent pods
kubectl -n wazuh get pods -l app=wazuh-agent -o wide

# Tail agent logs
kubectl -n wazuh logs -f <agent-pod>

# Check enrollment messages in logs
kubectl -n wazuh logs <agent-pod> | grep "agent-auth"

# Check agent status inside pod
kubectl -n wazuh exec -it <agent-pod> -- /var/ossec/bin/wazuh-control status

🔹 Connectivity
# Check DNS resolution to manager service
kubectl -n wazuh exec -it <agent-pod> -- \
  nslookup wazuh.wazuh.svc.cluster.local

# Check TCP reachability to manager on port 1515
kubectl -n wazuh exec -it <agent-pod> -- \
  nc -vz -w 2 wazuh.wazuh.svc.cluster.local 1515

▶️ Running Workflows

You can run workflows in two ways:

1. From GitHub Actions UI

Navigate to the Actions tab in this repo.

Select the workflow (e.g., 02-storage-wazuh.yml).

Click “Run workflow” and confirm.

2. From GitHub CLI

If you have the GitHub CLI
 installed:

# Run sanity check workflow
gh workflow run 01-sanity-check.yml

# Run storage setup
gh workflow run 02-storage-wazuh.yml

# Run wazuh deployment
gh workflow run 03-wazuh-deployment.yml

# Run passwordless auth
gh workflow run 04-passwordlesss-auth.yml

# Run agent install
gh workflow run 05-agent-install-config.yml

🎯 Final Outcome

At the end of this process you will have:

A fully deployed Wazuh SOC platform (manager, indexer, dashboard)

Passwordless agent enrollment

Agents automatically collecting and reporting logs to Wazuh Manager
