# 🛒 Mini Shop Kubernetes Security Lab 🛡

Welcome to the **Mini Shop Security Lab**! This project is a hands-on DevSecOps environment designed to demonstrate real-world Kubernetes vulnerabilities (like flat networks and excessive container privileges) and how to secure them using the "Defense in Depth" approach.

##  Architecture (3-Tier Microservices)
- **Frontend:** Web storefront (Socks Shop UI) exposed to the public via `NodePort`.
- **Payments:** Internal payment gateway API exposed internally via `ClusterIP`.
- **Database:** Internal Redis data vault exposed internally via `ClusterIP`.

##  Prerequisites
To run this lab locally, ensure you have the following installed:
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Kubescape](https://kubescape.io/) (for security auditing)

---

##  Quick Start Guide (Team Execution Playbook)

###  Phase 1: Infrastructure & Baseline Vulnerability (Person 1)
Start the cluster with Calico CNI (this is **required** for Network Policies to work later):
```bash
minikube start --cni=calico
```

Deploy the vulnerable baseline architecture:
```bash
kubectl apply -f frontend.yaml
kubectl apply -f payments.yaml
kubectl apply -f database.yaml
```

**Proof of Vulnerability (Lateral Movement Test):**
Simulate an attacker who breached a random pod to see if they can access the internal payments service:
```bash
kubectl run test-pod --rm -it --image=busybox --restart=Never -- wget -qO- -T 3 http://payments
```
*(Expected Result: Successfully fetches the Nginx HTML page. This proves the cluster network is completely flat and open).*

---

###  Phase 2: Security Audit (Person 2)
Scan the baseline YAML manifests against the NSA hardening framework to discover vulnerabilities:
```bash
kubescape scan framework nsa frontend.yaml payments.yaml database.yaml
```
*(Expected Result: A compliance score of ~60%. Vulnerabilities will show containers running as root, lack of resource limits, and no network segmentation).*

---

###  Phase 3: Least Privilege & RBAC (Person 3)
Lock down the containers (enforce non-root users, read-only filesystems, and drop capabilities) by deploying the secure manifests:
```bash
kubectl apply -f frontend-secure.yaml
kubectl apply -f payments-secure.yaml
kubectl apply -f database-secure.yaml
```

Secure the human/service accounts by applying RBAC (Role-Based Access Control) for the 'intern' account:
```bash
kubectl apply -f rbac-intern.yaml
```

**Verify RBAC Rules:**
```bash
kubectl auth can-i list pods --as=system:serviceaccount:default:intern
# Expected Output: yes

kubectl auth can-i delete pods --as=system:serviceaccount:default:intern
# Expected Output: no
```

---

###  Phase 4: Zero Trust Network (Person 4)
Apply the Network Policy (`policy.yaml`) to isolate the internal services. *(Note: Ensure `policy.yaml` is created and configured to only allow specific labels like `access: "true"` to reach the payments service).*
```bash
kubectl apply -f policy.yaml
```

**Proof of Security (Blocked Lateral Movement):**
Run the attacker simulation again:
```bash
kubectl run stranger --rm -it --image=busybox --restart=Never -- wget -qO- -T 3 http://payments
```
*(Expected Result: `wget: download timed out`. The zero-trust network policy successfully blocked the unauthorized connection!)*

**Final Audit:**
Re-run the Kubescape scan on the secure files to verify the score improvement (Expect ~95%+):
```bash
kubescape scan framework nsa payments-secure.yaml
```

---

##  Cleanup
To cleanly destroy the lab environment and remove the cluster:
```bash
minikube delete
```
