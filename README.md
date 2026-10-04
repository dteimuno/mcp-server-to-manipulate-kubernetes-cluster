# Managing a Kubernetes Cluster with MCP + GitHub Copilot Agent

Connecting an AI agent to a self-managed Kubernetes cluster through the **Model Context Protocol (MCP)**, so the cluster can be inspected and changed in plain English from VS Code.

This was my first hands-on MCP lab, following MCP fundamentals from KodeKloud. It took me from understanding the theory (resources, tools, prompts) to having an agent create and expose a Jenkins deployment on my cluster.

---

## What I Built

```
┌──────────────────────┐   Remote SSH    ┌──────────────────────────────────────────┐
│  VS Code (macOS)     │ ──────────────► │  controlplane node (Ubuntu, kubeadm)     │
│  Copilot Chat        │                 │                                          │
│  (Agent mode)        │                 │  VS Code Server                          │
└──────────────────────┘                 │     │ stdio                              │
                                         │     ▼                                    │
                                         │  docker run mcp/kubernetes  (MCP server) │
                                         │     │ ~/.kube/config (read-only mount)   │
                                         │     ▼                                    │
                                         │  kube-apiserver                          │
                                         └──────────────────────────────────────────┘
```

- **Cluster:** kubeadm cluster (1 control plane + 1 worker) with containerd runtime, running cert-manager, ingress-nginx, kube-prometheus-stack, GitLab and Redis via Helm
- **MCP server:** [`mcp/kubernetes`](https://hub.docker.com/r/mcp/kubernetes) Docker image (mcp-server-kubernetes v3.9.3)
- **MCP client:** GitHub Copilot Chat in VS Code, Agent mode, connected to the node over Remote SSH

---

## MCP Concepts in Brief

MCP servers expose three kinds of capabilities, separated by **who controls them**:

| Capability | Controlled by | Purpose | Example |
|---|---|---|---|
| **Resource** | Application / user | Read-only context data | A list of airports, a config file |
| **Tool** | The LLM | Actions the model decides to call | `search_flights()`, `kubectl get pods` |
| **Prompt** | The user | Reusable message templates | `/find_best_flight budget=300` |

In this project, the Kubernetes MCP server exposes **tools** (list pods, get logs, create deployments, Helm operations). The agent chooses which ones to call, and I approve each call before it runs.

---

## Setup

### 1. Prerequisites on the control plane

```bash
kubectl get nodes
```
```bash
ls -l ~/.kube/config
```

### 2. Install Docker without breaking the cluster

The node runs Kubernetes on `containerd.io` (from Docker's apt repo). Installing Ubuntu's `docker.io` package would pull in a conflicting containerd package and could disrupt running pods. I checked first:

```bash
dpkg -l | grep containerd
```

Since it showed `containerd.io`, I installed `docker-ce` from Docker's repo, which shares the same containerd. Then I added my user to the docker group:

```bash
sudo usermod -aG docker ubuntu
```

And confirmed the cluster was unaffected:

```bash
kubectl get pods -A
```

### 3. Test the MCP server manually

```bash
docker run -i --rm --network host --user 1000:1000 -v /home/ubuntu/.kube/config:/kube/config:ro -e KUBECONFIG=/kube/config mcp/kubernetes
```

Expected output (it then waits silently for MCP messages on stdin):
```
Starting Kubernetes MCP server v3.9.3, handling commands...
Telemetry: Disabled
```

### 4. Configure VS Code

`/home/ubuntu/.vscode/mcp.json`, with `/home/ubuntu` opened as the workspace folder:

```json
{
  "servers": {
    "kubernetes": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "--network", "host",
        "--user", "1000:1000",
        "-v", "/home/ubuntu/.kube/config:/kube/config:ro",
        "-e", "KUBECONFIG=/kube/config",
        "mcp/kubernetes"
      ]
    }
  }
}
```

Each flag fixes a real problem:

| Flag | Why it's needed |
|---|---|
| `-i` | MCP stdio transport talks over stdin/stdout |
| `--network host` | The kubeconfig points at the API server by node IP; on the default bridge network the container can't reach it the same way |
| `--user 1000:1000` | The kubeconfig is mode `600`. The image's default user can't read it, and running as my UID avoids making the file world-readable |
| `-v ...:ro` | Mounts the kubeconfig read-only into the container |
| `-e KUBECONFIG=...` | Tells the server where the mounted config is |

### 5. Use it

Open Copilot Chat (`Ctrl+Cmd+I`), switch to **Agent** mode, confirm the kubernetes tools are enabled under the tools icon, and start asking.

---

## Results

| Prompt | What the agent did |
|---|---|
| *"Please list all restarted pods"* | Found 43 pods with restarts across all namespaces and summarized them in a table |
| *"List all my Helm releases irrespective of namespace"* | Returned all 5 releases (cert-manager, gitlab, ingress-nginx, prometheus, redis) with chart and app versions |
| *"Create a Jenkins deployment with image jenkins/jenkins:lts-jdk21"* | Created the deployment in `default` and verified the rollout reached 1/1 available |
| *"Expose the Jenkins deployment using NodePort 30080"* | **Detected that 30080 was already used by ingress-nginx** and asked how to proceed instead of failing |
| *"Use another node port, maybe 32000"* | Created a NodePort Service on 32000 → container port 8080 and confirmed the endpoint was populated |

Jenkins was then reachable at `http://<node-IP>:32000`.

The NodePort conflict was the most interesting moment. The agent checked existing allocations before acting and asked a clarifying question, which is the behavior you want from anything with write access to a cluster.

---

## Troubleshooting Journey

Most of my learning came from what went wrong:

| Problem | Cause | Fix |
|---|---|---|
| `Command 'docker' not found` | kubeadm nodes often run containerd only | Installed `docker-ce` matching the existing `containerd.io` |
| `EACCES: permission denied, open '/kube/config'` | Container user ≠ file owner; kubeconfig is `600` | `--user 1000:1000` (not `chmod 644` on admin credentials) |
| MCP config ignored | Used `"mcpServers"` (Claude Desktop format) | VS Code expects `"servers"` |
| No **Start** option for the server | Opened `~/.vscode` itself as the workspace, so VS Code looked for `~/.vscode/.vscode/mcp.json` | Opened `/home/ubuntu` as the workspace folder |
| Pasted config into chat | Chat isn't where MCP config lives | Config goes in `mcp.json` |
| "Language model unavailable" / "Chat took too long to get ready" | Copilot Chat required VS Code ^1.120; I was on an old version (1.90.1 locally, 1.106.3 server) | Removed the old app, installed the latest VS Code (1.140.0), reconnected |

**Lesson:** read the logs. The VS Code Output panel had the exact answer (`Extension is not compatible with Code 1.106.3. Extension requires: ^1.120.0`) long before I found it.

---

## Security Notes

- The mounted kubeconfig is **cluster-admin**, so the agent effectively has full control of the cluster. That's acceptable in a lab, but not elsewhere.
- Copilot asks for approval before each tool call or terminal command. Read them before approving.
- Copilot's agent also has a built-in terminal tool, so some requests ran as `kubectl` commands in the terminal instead of through MCP tools. Check the approval prompt to see which path it's using.

### Next improvements
- [ ] Create a dedicated ServiceAccount with a scoped Role/ClusterRole and generate a kubeconfig for it
- [ ] Try `ALLOW_ONLY_NON_DESTRUCTIVE_TOOLS=true` for read-only investigation sessions
- [ ] Build my own small MCP server with FastMCP that exposes cluster resources and prompts, not just tools
- [ ] Put Jenkins behind the existing ingress-nginx with a cert-manager TLS certificate instead of a NodePort

---

## Tech Stack

`Kubernetes (kubeadm)` · `containerd` · `Docker` · `Helm` · `MCP` · `GitHub Copilot (Agent mode)` · `VS Code Remote SSH` · `Ubuntu`
