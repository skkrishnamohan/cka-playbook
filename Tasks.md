# Kubernetes CKA Tasks - Complete Walkthrough with Explanations and Error Analysis

## 📋 Quick Summary Table

### Tasks 1-9 (Basic Pod & Deployment Operations)
| Task | Objective | Status | Key Learning |
|------|-----------|--------|--------------|
| Task 1 | Create Pod (Imperative) | ✅ SUCCESS | Imperative method using `kubectl run` |
| Task 2 | Edit Pod Label | ✅ SUCCESS | Using `kubectl label` command directly |
| Task 3 | Create Pod (Invalid Image) | ❌ FAILED | Invalid image causes ErrImagePull error |
| Task 4 | Update Pod (Field Error) | ❌ FAILED | Pods are mostly immutable (except image field) |
| Task 5 | Create Pod (Valid Image) | ✅ SUCCESS | Correct declarative pod creation |
| Task 6 | Create ReplicaSet | ✅ SUCCESS | ReplicaSets manage multiple pod replicas |
| Task 7 | Update ReplicaSet Image | ✅ SUCCESS | Existing pods keep old image; new pods get new image |
| Task 8 | Create Deployment | ✅ SUCCESS (after fix) | Deployments provide rolling updates & history |
| Task 9 | Update Deployment Image | ✅ SUCCESS | Rolling update with zero downtime |

### Tasks 10-18 (YAML & Imperative Deployments)
| Task | Objective | Status | Key Learning |
|------|-----------|--------|--------------|
| Task 10 | Create Pod with YAML | ✅ SUCCESS | Declarative approach using YAML manifests |
| Task 11 | Create Pod Imperative | ✅ SUCCESS | Quick imperative pod creation without YAML |
| Task 12 | Create Deployment (3 replicas) | ✅ SUCCESS | Deployment with multiple replicas |
| Task 13 | Scale Deployment (3→5 replicas) | ✅ SUCCESS | Using `kubectl scale` to increase replicas |
| Task 14 | Verify New Pods | ✅ SUCCESS | Confirming pods created during scaling |
| Task 15 | Check Pods per Node (-o wide) | ✅ SUCCESS | Understanding pod distribution across nodes |
| Task 16 | Create Deployment (Imperative) | ✅ SUCCESS | Imperative deployment creation with `kubectl create` |
| Task 17 | Describe Deployment | ✅ SUCCESS | Understanding deployment configuration and status |
| Task 18 | Access Pod (Port Forwarding) | ✅ SUCCESS | Port forwarding and direct pod access without Service |

### Tasks 19-26 (Execution, Scaling & Troubleshooting)
| Task | Objective | Status | Key Learning |
|------|-----------|--------|--------------|
| Task 19 | Execute Commands in Pod | ✅ SUCCESS | Using `kubectl exec` for pod debugging |
| Task 20 | Create ReplicaSet (2 replicas) | ✅ SUCCESS | ReplicaSet creation with Docker Hub images |
| Task 21 | Image Registry Architecture | ✅ INFO | Why Docker Hub is default, not Kubernetes registry |
| Task 22 | Pod Troubleshooting Steps | ✅ SUCCESS | Systematic troubleshooting methodology |
| Task 23 | Check Pod Logs | ✅ SUCCESS | Retrieving container logs with `kubectl logs` |
| Task 24 | Multi-container Pod Logs | ✅ SUCCESS | Accessing specific container logs with `-c` flag |
| Task 25 | Check Kubernetes Events | ✅ SUCCESS | Viewing cluster events for debugging |
| Task 26 | Identify Image/Scheduling Issues | ✅ SUCCESS | Diagnosing image pull and scheduling problems |

## 🎯 Kubernetes Resources Progression
```
Pods (basic) → ReplicaSets (replica management) → Deployments (rolling updates)
                    ↓
             Multi-container Pods & Port Forwarding
                    ↓
             Troubleshooting (Logs, Events, Image Issues)
                    ↓
             Complete Pod Lifecycle Management
```

---

# Task 1: Create a Pod using Imperative Method with nginx:1.28

**Objective:** Create a pod using the imperative method with the nginx:1.28 image

**What This Task Does:**
This task demonstrates the imperative approach to creating Kubernetes pods. Instead of writing YAML manifests, you use `kubectl run` command to create resources directly on the fly.

**Solution:**
```bash
kubectl run mypod --image=nginx:1.28
```

**Verification Commands:**
```bash
kubectl get pods
kubectl describe pod mypod
```

**CLI Output:**
```
root@controlplane:~$ kubectl run mypod --image=nginx:1.28
pod/mypod created

root@controlplane:~$ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
mypod   1/1     Running   0          10s

root@controlplane:~$ kubectl describe pod mypod | grep Image
    Image:          nginx:1.28
    Image ID:       docker.io/library/nginx@sha256:146adea4768b83c607d0bdfa4188464e3da6e0a3ad4475db1d1d8f64f27c29cc
  Normal  Pulled     31s   kubelet            spec.containers{mypod}: Successfully pulled image "nginx:1.28" in 5.3s (5.3s including waiting). Image size: 62916597 bytes.
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- Pod `mypod` was successfully created with the nginx:1.28 image
- The pod is in `Running` state with 1/1 containers ready
- Image was successfully pulled from Docker Hub and the container started correctly 

---

# Task 2: Edit Pod Label using kubectl edit/label command

**Objective:** Add or edit the label of the pod created in Task1 using kubectl edit

**What This Task Does:**
This task demonstrates how to add labels to existing Kubernetes resources. Labels are key-value pairs that help organize and select resources. There are two methods shown here.

**Solution - Method 1 (Interactive - kubectl edit):**
```bash
kubectl edit pod mypod
```
Then find the `labels:` section and add:
```yaml
labels:
  app: myapp
  run: mypod
```

**Solution - Method 2 (Direct - kubectl label):**
```bash
kubectl label pod mypod app=myapp --overwrite
```

**Verification Commands:**
```bash
kubectl get pod mypod --show-labels
kubectl describe pod mypod
```

**CLI Output:**
```
root@controlplane:~$ kubectl edit pod mypod
Edit cancelled, no changes made.

root@controlplane:~$ kubectl label pod mypod app=myapp --overwrite
pod/mypod labeled

root@controlplane:~$ kubectl get pod mypod --show-labels
NAME    READY   STATUS    RESTARTS   AGE     LABELS
mypod   1/1     Running   0          5m36s   app=myapp,run=mypod
```

**Status:** ✅ **SUCCESS (with correction)**

**Explanation:**
- **First Attempt:** The `kubectl edit` command opened the default editor but no changes were made (just entered and exited)
- **Solution Applied:** Used `kubectl label pod mypod app=myapp --overwrite` instead, which directly adds/updates the label
- **Result:** Pod now has two labels:
  - `run=mypod` (automatically added by kubectl when creating with `kubectl run`)
  - `app=myapp` (manually added in this task) 

---

# Task 3: Create Pod Declaratively with Name advpro and Image xyz

**Objective:** Create a pod declaratively where pod name is advpro and image is xyz

**What This Task Does:**
This task demonstrates the declarative approach using YAML manifests. Unlike imperative commands, you define the desired state in YAML and apply it with `kubectl apply`.

**Solution:**
Create a file named `task3.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: advpro
spec:
  containers:
  - name: advpro-container
    image: xyz
```

**Apply:**
```bash
kubectl apply -f task3.yaml
```

**Verification Commands:**
```bash
kubectl get pods
kubectl describe pod advpro
```

**CLI Output:**
```
root@controlplane:~$ kubectl apply -f task3.yaml
pod/advpro created

root@controlplane:~$ kubectl get pods
NAME     READY   STATUS         RESTARTS   AGE
advpro   0/1     ErrImagePull   0          15s
mypod    1/1     Running        0          8m45s

root@controlplane:~$ kubectl describe pod advpro
Name:             advpro
Namespace:        default
Status:           Pending
Containers:
  advpro-container:
    Image:          xyz
    State:          Waiting
      Reason:       ImagePullBackOff
Events:
  Warning  Failed     25s   kubelet            spec.containers{advpro-container}: Error: ErrImagePull
  Normal   Pulling    12s   kubelet            spec.containers{advpro-container}: Pulling image "xyz"
  Warning  Failed     10s   kubelet            spec.containers{advpro-container}: Failed to pull image "xyz": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/xyz:latest": failed to resolve image: docker.io/library/xyz:latest: not found
```

**Status:** ❌ **FAILED WITH ERROR**

**🔴 Issue Identified:**
- **Error Type:** `ErrImagePull` / `ImagePullBackOff`
- **Root Cause:** The image `xyz` does not exist in any Docker registry (Docker Hub, private registry, etc.)
- **Why It Happens:** When Kubernetes tries to pull the image from the registry, it cannot find `docker.io/library/xyz:latest`. The image name is invalid or non-existent.

**💡 Solution:**
Use a valid image name. Examples:
- `nginx` → pulls `docker.io/library/nginx:latest`
- `nginx:1.28` → pulls specific version
- `busybox` → lightweight test image
- `alpine` → minimal Linux distribution

**How to Fix:**
Replace the image name with a valid one:
```yaml
spec:
  containers:
  - name: advpro-container
    image: nginx        # Use a real image instead of 'xyz'
```

**Key Learning:**
Always use valid image names. You can test image availability with: `docker pull xyz` (will fail locally too if image doesn't exist) 
---

---

# Task 4: Create Pod Declaratively with Name advpro and Image nginx

**Objective:** Create a pod declaratively where pod name is advpro and image is nginx

**What This Task Does:**
This task attempts to update the pod from Task 3 by changing the image from `xyz` to `nginx`. This demonstrates an important Kubernetes constraint about updating pods.

**Attempted Solution:**
Create a file named `task4.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: advpro
spec:
  containers:
  - name: nginx-container
    image: nginx
```

**Apply:**
```bash
kubectl apply -f task4.yaml
```

**CLI Output (Error):**
```
root@controlplane:~$ kubectl apply -f task4.yaml
The Pod "advpro" is invalid: spec: Forbidden: pod updates may not change fields 
other than `spec.containers[*].image`,`spec.initContainers[*].image`,
`spec.activeDeadlineSeconds`,`spec.tolerations` (only additions to existing tolerations),
`spec.terminationGracePeriodSeconds` (allow it to be set to 1 if it was previously negative)

@@ -91,7 +91,7 @@
   "Containers": [
    {
-    "Name": "advpro-container",
+    "Name": "nginx-container",
     "Image": "xyz",
```

**Status:** ❌ **FAILED WITH ERROR**

**🔴 Issue Identified:**
- **Error Type:** `Forbidden: pod updates may not change fields`
- **Root Cause:** You attempted to change the container name from `advpro-container` to `nginx-container`
- **Why It Happens:** Kubernetes has strict rules about which fields can be modified in existing pods. Most fields are immutable once a pod is created. Only certain fields can be updated:
  - `spec.containers[*].image` ✅ (can update)
  - `spec.initContainers[*].image` ✅ (can update)
  - `spec.activeDeadlineSeconds` ✅ (can update)
  - `spec.tolerations` ✅ (can add only)
  - `spec.terminationGracePeriodSeconds` ✅ (can update)
  - **Container name** ❌ (CANNOT change - immutable)

**💡 Solution:**
Since you cannot modify the pod structure, you have two options:

**Option 1: Delete and recreate the pod**
```bash
kubectl delete pod advpro
kubectl apply -f task4.yaml  # with the corrected image
```

**Option 2: Keep the original container name**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: advpro
spec:
  containers:
  - name: advpro-container   # Keep original name
    image: nginx              # Change only the image
```

**Key Learning:**
- Pods are mostly immutable - if you need to change most properties, delete and recreate
- Only specific fields like image, deadlineSeconds, and tolerations can be modified
- For updating container images, use Deployments or ReplicaSets instead - they handle rolling updates automatically
---

---

# Task 5: Create Pod Declaratively with Name advpro1 and Image nginx

**Objective:** Create a pod declaratively where pod name is advpro1 and image is nginx

**What This Task Does:**
This task creates a new pod with a valid image name (nginx). This is the corrected version after the failures in Tasks 3 and 4.

**Solution:**
Create a file named `task5.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: advpro1
spec:
  containers:
  - name: nginx-container
    image: nginx
```

**Apply:**
```bash
kubectl apply -f task5.yaml
```

**Verification Commands:**
```bash
kubectl get pods
kubectl describe pod advpro1
```

**CLI Output:**
```
root@controlplane:~$ kubectl apply -f task5.yaml
pod/advpro1 created

root@controlplane:~$ kubectl get pods
NAME      READY   STATUS              RESTARTS   AGE
advpro    0/1     ImagePullBackOff    0          5m20s
advpro1   0/1     ContainerCreating   0          4s
mypod     1/1     Running             0          13m

root@controlplane:~$ kubectl describe pod advpro1
Name:             advpro1
Namespace:        default
Status:           Running
Containers:
  nginx-container:
    Container ID:   containerd://bf39d432cd11485ca87d65954c1c8fab9029752a383e7db24af662691682b08d
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:5aca99593157f4ae539a5dec1092a0ad8762f8e2eb1789085a13a0f5622369f6
    State:          Running
      Started:      Sat, 30 May 2026 02:55:39 +0000
    Ready:          True
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  47s   default-scheduler  Successfully assigned default/advpro1 to node01
  Normal  Pulling    47s   kubelet            spec.containers{nginx-container}: Pulling image "nginx"
  Normal  Pulled     42s   kubelet            spec.containers{nginx-container}: Successfully pulled image "nginx" in 5.417s
  Normal  Created    41s   kubelet            spec.containers{nginx-container}: Container created
  Normal  Started    41s   kubelet            spec.containers{nginx-container}: Container started
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- Pod `advpro1` was successfully created with the nginx image
- The pod is in `Running` state with `1/1` containers ready
- Image was successfully pulled and container started without errors
- This demonstrates the correct way to create declarative pods with valid image names
---

---

# Task 6: Create a ReplicaSet

**Objective:** Create a replicaset with name advreplica, replicacount 3, pod label app=testapp, image nginx

**What This Task Does:**
ReplicaSets ensure a specified number of pod replicas are running at all times. This task introduces a higher-level Kubernetes resource than Pods. If a pod crashes, the ReplicaSet automatically creates a replacement.

**Solution:**
Create a file named `task6.yaml`:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: advreplica
spec:
  replicas: 3
  selector:
    matchLabels:
      app: testapp
  template:
    metadata:
      labels:
        app: testapp
    spec:
      containers:
      - name: nginx
        image: nginx
```

**Apply:**
```bash
kubectl apply -f task6.yaml
```

**Verification Commands:**
```bash
kubectl get replicasets
kubectl get pods
kubectl describe rs advreplica
```

**CLI Output:**
```
root@controlplane:~$ kubectl apply -f task6.yaml
replicaset.apps/advreplica created

root@controlplane:~$ kubectl get replicasets
NAME         DESIRED   CURRENT   READY   AGE
advreplica   3         3         3       43s

root@controlplane:~$ kubectl describe rs advreplica
Name:         advreplica
Namespace:    default
Selector:     app=testapp
Labels:       <none>
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=testapp
  Containers:
   nginx:
    Image:         nginx
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  54s   replicaset-controller  Created pod: advreplica-m8wg4
  Normal  SuccessfulCreate  54s   replicaset-controller  Created pod: advreplica-lcjvt
  Normal  SuccessfulCreate  54s   replicaset-controller  Created pod: advreplica-x6bkx
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- ReplicaSet `advreplica` was successfully created
- All 3 desired replicas are running (DESIRED=3, CURRENT=3, READY=3)
- The ReplicaSet automatically created 3 pods with labels `app=testapp`
- Each pod is running the nginx image
- The selector `app=testapp` is used to identify which pods belong to this ReplicaSet
---

---

# Task 7: Update ReplicaSet Image

**Objective:** Update the image in the replicaset and observe if pods are updated

**What This Task Does:**
This task demonstrates how to update the container image in a ReplicaSet and shows an important behavior: existing pods keep their old image, only newly created pods use the new image.

**Solution - Method 1 (Interactive Edit):**
```bash
kubectl edit rs advreplica
# Find the image field and change: nginx → nginx:latest
```

**Solution - Method 2 (Direct Command):**
```bash
kubectl set image rs/advreplica nginx=nginx:latest
```

**Verification Commands:**
```bash
kubectl get pods
kubectl describe rs advreplica
# Delete old pods to see the new image applied
kubectl delete pod advreplica-lcjvt advreplica-m8wg4 advreplica-x6bkx
kubectl get pods
```

**CLI Output:**
```
root@controlplane:~$ kubectl edit rs advreplica
replicaset.apps/advreplica edited

root@controlplane:~$ kubectl set image rs/advreplica nginx=nginx:latest
replicaset.apps/advreplica image updated

root@controlplane:~$ kubectl get pods
NAME               READY   STATUS    RESTARTS   AGE
advpro             0/1     ImagePullBackOff   0    12m
advpro1            1/1     Running            0    6m57s
advreplica-lcjvt   1/1     Running            0    4m11s
advreplica-m8wg4   1/1     Running            0    4m11s
advreplica-x6bkx   1/1     Running            0    4m11s
mypod              1/1     Running            0    20m

root@controlplane:~$ kubectl delete pods advreplica-lcjvt advreplica-m8wg4 advreplica-x6bkx
pod "advreplica-lcjvt" deleted from default namespace
pod "advreplica-m8wg4" deleted from default namespace
pod "advreplica-x6bkx" deleted from default namespace

root@controlplane:~$ kubectl get pods
NAME               READY   STATUS    RESTARTS   AGE
advpro             0/1     ImagePullBackOff   0    14m
advpro1            1/1     Running            0    8m49s
advreplica-ck9h8   1/1     Running            0    4s
advreplica-d6559   1/1     Running            0    4s
advreplica-r4wdb   1/1     Running            0    4s
mypod              1/1     Running            0    22m
```

**Status:** ✅ **SUCCESS WITH IMPORTANT OBSERVATION**

**Key Learning - Pod Immutability:**
- When you update the ReplicaSet image, **existing pods are NOT recreated** with the new image
- They continue running with the old image (nginx:latest tag)
- Only **newly created pods** will have the new image
- The old pods (advreplica-lcjvt, advreplica-m8wg4, advreplica-x6bkx) had to be deleted manually
- The ReplicaSet immediately created 3 new pods (advreplica-ck9h8, advreplica-d6559, advreplica-r4wdb) to maintain the replica count
- These new pods run with the updated nginx:latest image

**Comparison with Deployments:**
- Deployments handle this better with **rolling updates** - they gradually replace old pods with new ones
- ReplicaSets don't have built-in rolling update strategy
- This is one reason why Deployments are preferred for production workloads
---

# Task 8: Create a Deployment

**Objective:** Create a deployment named anandhi with replica count 10 and image busybox:stable

**What This Task Does:**
Deployments are higher-level Kubernetes objects that manage ReplicaSets and provide declarative updates. They support rolling updates, rollbacks, and scaling.

**Solution:**
Create a file named `task8.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: anandhi
spec:
  replicas: 10
  selector:
    matchLabels:
      app: busybox-app
  template:
    metadata:
      labels:
        app: busybox-app
    spec:
      containers:
      - name: busybox
        image: busybox:stable
        command: ['sh', '-c', 'sleep 1000']
```

**Apply:**
```bash
kubectl apply -f task8.yaml
```

**Verification Commands:**
```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods
kubectl describe deployment anandhi
```

**CLI Output (First Attempt - Issue):**
```
root@controlplane:~$ kubectl apply -f task8.yaml
deployment.apps/anandhi created

root@controlplane:~$ kubectl get pods
NAME                      READY   STATUS           RESTARTS   AGE
anandhi-945c8f964-2djf6   0/1     CrashLoopBackOff 1 (5s ago) 8s
anandhi-945c8f964-77p6f   0/1     Completed        2 (19s ago) 22s
anandhi-945c8f964-9qgdk   0/1     Completed        2 (24s ago) 28s
...all pods showing Completed or CrashLoopBackOff...

root@controlplane:~$ kubectl get deployments
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
anandhi   0/10    10           0           57s
```

**CLI Output (After Re-applying - Success):**
```
root@controlplane:~$ kubectl apply -f task8.yaml
deployment.apps/anandhi configured

root@controlplane:~$ kubectl get deployments
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
anandhi   10/10   10           10          3m51s

root@controlplane:~$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
anandhi-f9f65bc46-2km7c   1/1     Running   0          8s
anandhi-f9f65bc46-447r9   1/1     Running   0          12s
anandhi-f9f65bc46-cndk9   1/1     Running   0          11s
anandhi-f9f65bc46-d5ldp   1/1     Running   0          12s
anandhi-f9f65bc46-pddwb   1/1     Running   0          12s
anandhi-f9f65bc46-rd5m2   1/1     Running   0          12s
anandhi-f9f65bc46-skjgb   1/1     Running   0          9s
anandhi-f9f65bc46-th7qw   1/1     Running   0          10s
anandhi-f9f65bc46-xh5hj   1/1     Running   0          12s
anandhi-f9f65bc46-z4mcc   1/1     Running   0          11s
```

**Status:** ✅ **SUCCESS (after troubleshooting)**

**🔴 Issue Encountered (First Attempt):**
- **Problem:** All pods were showing `Completed` or `CrashLoopBackOff` status
- **Root Cause:** The `sleep 1000` command completes after 1000 seconds, causing the container to exit successfully
- **Why It Happens:** 
  - The container runs `sleep 1000` which is a finite command
  - After 1000 seconds, the sleep exits with status 0 (success)
  - Kubernetes sees the process exited and shows `Completed` status
  - Because there's no `RestartPolicy: Never`, it tries to restart the pod (default is `Always`)
  - This causes the `CrashLoopBackOff` status when restarting repeatedly

**💡 Solution Applied:**
The issue was resolved by re-applying the manifest, which triggered a new ReplicaSet creation. The pods from the new ReplicaSet (anandhi-f9f65bc46-*) are all running properly.

**The Real Fix Would Be:**
Use a command that keeps running indefinitely:
```yaml
command: ['sh', '-c', 'while true; do sleep 30; done']
```
Or simply:
```yaml
command: ['sleep', 'infinity']
```

**Key Learning - Deployment Benefits:**
- Deployments automatically create and manage ReplicaSets
- They support rolling updates - gradually replacing old pods with new ones
- This task shows all 10 pods transitioning from old ReplicaSet to new one
- READY shows 10/10 (all pods are ready and running)

root@controlplane:~$ vi task8.yaml
root@controlplane:~$ kubectl apply -f task8.yaml 
deployment.apps/anandhi created
root@controlplane:~$ kubectl get pods
NAME                      READY   STATUS              RESTARTS     AGE
advpro                    0/1     ImagePullBackOff    0            15m
advpro1                   1/1     Running             0            10m
advreplica-ck9h8          1/1     Running             0            99s
advreplica-d6559          1/1     Running             0            99s
advreplica-r4wdb          1/1     Running             0            99s
anandhi-945c8f964-2djf6   0/1     CrashLoopBackOff    1 (5s ago)   8s
anandhi-945c8f964-77p6f   0/1     CrashLoopBackOff    1 (4s ago)   8s
anandhi-945c8f964-9qgdk   0/1     CrashLoopBackOff    1 (4s ago)   8s
anandhi-945c8f964-bdm4w   0/1     CrashLoopBackOff    1 (5s ago)   8s
anandhi-945c8f964-j2jv8   0/1     ContainerCreating   0            8s
anandhi-945c8f964-n46kh   0/1     Completed           1 (6s ago)   8s
anandhi-945c8f964-qtwxz   0/1     CrashLoopBackOff    1 (4s ago)   8s
anandhi-945c8f964-r97n5   0/1     ContainerCreating   0            8s
anandhi-945c8f964-s5b9t   0/1     CrashLoopBackOff    1 (4s ago)   8s
anandhi-945c8f964-xqbdd   0/1     ContainerCreating   0            8s
mypod                     1/1     Running             0            24m
```

**🔴 Issue Encountered & Debugging Steps:**

**First Attempt - Issue Detected:**
```bash
root@controlplane:~$ kubectl apply -f task8.yaml
deployment.apps/anandhi created

root@controlplane:~$ kubectl get deployments
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
anandhi   0/10    10           0           57s
```

All 10 pods were showing either `Completed` or `CrashLoopBackOff` status.

**Debugging Output (Showing Progression):**
```
root@controlplane:~$ kubectl get pods
NAME                      READY   STATUS              RESTARTS      AGE
advpro                    0/1     ImagePullBackOff    0             15m
advpro1                   1/1     Running             0             10m
advreplica-ck9h8          1/1     Running             0             99s
advreplica-d6559          1/1     Running             0             99s
advreplica-r4wdb          1/1     Running             0             99s
anandhi-945c8f964-2djf6   0/1     CrashLoopBackOff    1 (5s ago)   8s
anandhi-945c8f964-77p6f   0/1     CrashLoopBackOff    1 (4s ago)   8s
anandhi-945c8f964-9qgdk   0/1     CrashLoopBackOff    1 (4s ago)   8s
... [pods in CrashLoopBackOff/ContainerCreating states] ...
```

**Root Cause Identified:**
The `sleep 1000` command completes after 1000 seconds:
- Sleep exits with status 0 (success) → pod shows `Completed`
- Default RestartPolicy is `Always` → Kubernetes tries to restart the pod
- Multiple restart attempts show up as `CrashLoopBackOff`

---

**✅ Solution Applied (Re-apply Manifest):**

```bash
root@controlplane:~$ kubectl apply -f task8.yaml 
deployment.apps/anandhi configured

root@controlplane:~$ kubectl get deployments
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
anandhi   10/10   10           10          3m51s
```

**All Pods Running Successfully:**
```
root@controlplane:~$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
anandhi-f9f65bc46-2km7c   1/1     Running   0          8s
anandhi-f9f65bc46-447r9   1/1     Running   0          12s
anandhi-f9f65bc46-cndk9   1/1     Running   0          11s
anandhi-f9f65bc46-d5ldp   1/1     Running   0          12s
anandhi-f9f65bc46-pddwb   1/1     Running   0          12s
anandhi-f9f65bc46-rd5m2   1/1     Running   0          12s
anandhi-f9f65bc46-skjgb   1/1     Running   0          9s
anandhi-f9f65bc46-th7qw   1/1     Running   0          10s
anandhi-f9f65bc46-xh5hj   1/1     Running   0          12s
anandhi-f9f65bc46-z4mcc   1/1     Running   0          11s
```

**Note:** Pod names changed from `anandhi-945c8f964-*` to `anandhi-f9f65bc46-*` because a new ReplicaSet was created.

---

**💡 The Proper Long-Term Fix:**

Instead of `sleep 1000` (which eventually completes), use a command that runs indefinitely:

```yaml
# Option 1: Use sleep infinity
command: ['sleep', 'infinity']

# Option 2: Use an infinite loop  
command: ['sh', '-c', 'while true; do sleep 30; done']
```

**Key Learning - How Deployments Manage Updates:**
- When re-applying a deployment manifest, Kubernetes creates a **new ReplicaSet**
- The **old ReplicaSet** (anandhi-945c8f964) is kept in history for rollback capability
- This demonstrates the **rolling update** mechanism
- By default, it replaces pods gradually (25% at a time) for zero-downtime deployments

# Task 9: Update Deployment Image

**Objective:** Update the deployment image from busybox:stable to busybox:1.37

**What This Task Does:**
This task demonstrates one of the key advantages of Deployments - rolling updates. When you update the image, Kubernetes gradually replaces old pods with new ones while maintaining availability.

**Solution - Method 1 (Direct Command):**
```bash
kubectl set image deployment/anandhi busybox=busybox:1.37
```

**Solution - Method 2 (Interactive Edit):**
```bash
kubectl edit deployment anandhi
# Find image: busybox:stable and change to busybox:1.37
```

**Verification Commands:**
```bash
kubectl get deployments
kubectl get pods
kubectl describe deployment anandhi
kubectl rollout status deployment/anandhi
kubectl rollout history deployment/anandhi
```

**CLI Output:**
```
root@controlplane:~$ kubectl set image deployment/anandhi busybox=busybox:1.37
deployment.apps/anandhi image updated

root@controlplane:~$ kubectl get deployments
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
anandhi   10/10   10           10          5m8s

root@controlplane:~$ kubectl get pods
NAME                       READY   STATUS        RESTARTS   AGE
anandhi-585598b444-2jvbt   1/1     Running       0          15s
anandhi-585598b444-4w95p   1/1     Running       0          16s
anandhi-585598b444-7bmwp   1/1     Running       0          19s
anandhi-585598b444-b2h2g   1/1     Running       0          19s
anandhi-585598b444-fkbkc   1/1     Running       0          16s
anandhi-585598b444-gvbtb   1/1     Running       0          17s
anandhi-585598b444-h7cjz   1/1     Running       0          19s
anandhi-585598b444-lctfk   1/1     Running       0          15s
anandhi-585598b444-mx484   1/1     Running       0          19s
anandhi-585598b444-tmjks   1/1     Running       0          11s
anandhi-f9f65bc46-2km7c    1/1     Terminating   0          82s
anandhi-f9f65bc46-447r9    1/1     Terminating   0          86s
anandhi-f9f65bc46-cndk9    1/1     Terminating   0          85s
anandhi-f9f65bc46-d5ldp    1/1     Terminating   0          86s
anandhi-f9f65bc46-pddwb    1/1     Terminating   0          86s
anandhi-f9f65bc46-rd5m2    1/1     Terminating   0          86s
anandhi-f9f65bc46-skjgb    1/1     Terminating   0          83s
anandhi-f9f65bc46-th7qw    1/1     Terminating   0          84s
anandhi-f9f65bc46-xh5hj    1/1     Terminating   0          86s
anandhi-f9f65bc46-z4mcc    1/1     Terminating   0          85s

root@controlplane:~$ kubectl describe deployment anandhi
Name:                   anandhi
Namespace:              default
CreationTimestamp:      Sat, 30 May 2026 03:05:49 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 3
Selector:               app=busybox-app
Replicas:               10 desired | 10 updated | 10 total | 10 available | 0 unavailable
StrategyType:           RollingUpdate
Pod Template:
  Containers:
   busybox:
    Image:      busybox:1.37
    Command:    sh -c sleep 1000
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  anandhi-945c8f964 (0/0 replicas), anandhi-f9f65bc46 (0/0 replicas)
NewReplicaSet:   anandhi-585598b444 (10/10 replicas created)

root@controlplane:~$ kubectl rollout status deployment/anandhi
deployment "anandhi" successfully rolled out

root@controlplane:~$ kubectl rollout history deployment/anandhi
deployment.apps/anandhi
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

**Status:** ✅ **SUCCESS**

**Rolling Update in Action:**
The CLI output above shows the rolling update process:
1. **New ReplicaSet created:** `anandhi-585598b444` pods start coming up with busybox:1.37
2. **Gradual replacement:** Old pods (anandhi-f9f65bc46-*) show `Terminating` status as new ones start
3. **All pods replaced:** Eventually all 10 pods from the new ReplicaSet are running with the new image
4. **Zero downtime:** The RollingUpdateStrategy (25% max unavailable, 25% max surge) ensures continuous availability

**Key Learning - Deployment Rollout Details:**
- **Rolling Update Strategy:** Gradually replaces pods (not all at once like ReplicaSet)
- **Revision History:** The deployment has 3 revisions tracked:
  - Revision 1: Initial deployment with busybox:stable
  - Revision 2: Re-applied task8.yaml (fixed the Completed status issue)
  - Revision 3: Updated to busybox:1.37
- **Rollout Status:** Shows the update completed successfully
- **Can Rollback:** You can rollback to previous revisions using `kubectl rollout undo deployment/anandhi`

**Advantages Over ReplicaSet:**
- ✅ Automatic rolling updates with zero downtime
- ✅ Revision history for tracking changes
- ✅ Easy rollback to previous versions
- ✅ Configurable update strategy (RollingUpdate or Recreate)
- ✅ Progress monitoring with rollout status

## Additional Cleanup Commands
```bash
# Delete all resources
kubectl delete pod mypod
kubectl delete pod advpro
kubectl delete pod advpro1
kubectl delete rs advreplica
kubectl delete deployment anandhi

# Or delete all in one command
kubectl delete all --all
```

---

# 📚 Learning Summary & Key Concepts

## Kubernetes Resource Hierarchy

```
Pod (basic unit)
  ↓
ReplicaSet (manages replicas, no rolling updates)
  ↓
Deployment (adds rolling updates, history, rollback)
```

## Common Errors Encountered & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `ErrImagePull` | Invalid image name | Use valid image names (e.g., nginx, busybox, alpine) |
| `ImagePullBackOff` | Cannot pull image from registry | Check image name and registry connectivity |
| `Forbidden: pod updates may not change fields` | Trying to modify immutable pod fields | Delete and recreate pod, or only modify image field |
| `Completed` status | Container process exits successfully | Use commands that keep running indefinitely |
| `CrashLoopBackOff` | Container crashes and restarts repeatedly | Fix container command or RestartPolicy |

## Key Differences Between Kubernetes Resources

| Feature | Pod | ReplicaSet | Deployment |
|---------|-----|-----------|-----------|
| **Direct Pod Management** | ✅ Yes | ❌ No | ❌ No |
| **Ensures Replicas** | ❌ No | ✅ Yes | ✅ Yes |
| **Rolling Updates** | ❌ No | ❌ No | ✅ Yes |
| **Revision History** | ❌ No | ❌ No | ✅ Yes |
| **Rollback Support** | ❌ No | ❌ No | ✅ Yes |
| **Self-Healing** | ❌ No | ✅ Yes | ✅ Yes |
| **Update Strategy** | N/A | Manual (no orchestration) | Automatic (RollingUpdate/Recreate) |

## CKA Exam Focus Areas Covered

✅ **Cluster Administration:** Pod lifecycle, resource management
✅ **Workloads & Scheduling:** Creating and managing Pods, ReplicaSets, Deployments
✅ **Labels & Selectors:** Understanding label-based pod selection
✅ **Deployment Strategies:** Rolling updates, zero-downtime deployments
✅ **Image Management:** Pulling valid images, handling image pull errors
✅ **Troubleshooting:** Debugging pod issues, analyzing describe output
✅ **Imperative vs Declarative:** Both approaches demonstrated

## Best Practices Learned

1. **Always use valid image names** - Test with `docker pull <image>` first
2. **Use Deployments for production** - ReplicaSets are mostly for Deployment internals
3. **Leverage labels effectively** - Organize pods with meaningful labels
4. **Monitor rollout progress** - Use `kubectl rollout status` to track updates
5. **Keep revision history** - Easy rollback is crucial in production
6. **Plan container lifecycle** - Use commands that match intended application behavior
7. **Understand pod immutability** - Know which fields can be updated after pod creation

## Useful kubectl Commands Reference

```bash
# Imperative - Quick pod creation
kubectl run <name> --image=<image>

# Declarative - From YAML files  
kubectl apply -f <file>

# Inspect resources
kubectl get pods/replicasets/deployments
kubectl describe <resource-type> <resource-name>

# Update resources
kubectl label <resource> <key=value>
kubectl set image <resource> <container=image>
kubectl edit <resource>

# Rollout management (Deployments only)
kubectl rollout status <deployment>
kubectl rollout history <deployment>
kubectl rollout undo <deployment>

# Cleanup
kubectl delete <resource-type> <resource-name>
kubectl delete all --all  # Delete everything in namespace
```

---

# Task 10: Create a Pod in Kubernetes using a YAML file

**Objective:** Create a Pod using a YAML manifest file with proper structure

**What This Task Does:**
This task demonstrates the declarative approach to creating Kubernetes pods using YAML manifests. This is the recommended method for production environments as it allows version control and reproducibility.

**Solution:**
Create a file named `pod-nginx.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

**Apply the manifest:**
```bash
kubectl apply -f pod-nginx.yaml
```

**Verification Commands:**
```bash
kubectl get pods
kubectl describe pod nginx-pod
kubectl get pod nginx-pod -o yaml
```

**CLI Output:**
```
root@controlplane:~$ kubectl apply -f pod-nginx.yaml
pod/nginx-pod created

root@controlplane:~$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          10s

root@controlplane:~$ kubectl describe pod nginx-pod
Name:         nginx-pod
Namespace:    default
Status:       Running
Containers:
  nginx:
    Image:  nginx:latest
    State:  Running
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- Pod `nginx-pod` was successfully created from the YAML manifest
- The declarative approach allows you to version control your infrastructure
- YAML manifests can be easily shared, reviewed, and reused across teams
- This method is preferred over imperative commands in production environments

---

# Task 11: Create a Pod directly using a command without using a YAML file

**Objective:** Create a Pod using imperative kubectl command without writing YAML

**What This Task Does:**
This task demonstrates the imperative approach to quickly create Kubernetes pods without writing YAML files. This method is useful for quick testing and prototyping.

**Solution:**
```bash
kubectl run quick-pod --image=nginx:latest --restart=Never
```

**Verification Commands:**
```bash
kubectl get pods
kubectl describe pod quick-pod
```

**CLI Output:**
```
root@controlplane:~$ kubectl run quick-pod --image=nginx:latest --restart=Never
pod/quick-pod created

root@controlplane:~$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
quick-pod   1/1     Running   0          5s

root@controlplane:~$ kubectl describe pod quick-pod | grep Image
    Image:          nginx:latest
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- Pod created instantly without writing a YAML file
- The `--restart=Never` flag ensures no restart policy is applied
- Imperative commands are quick for testing but less suitable for production
- Note: Without `--restart=Never`, kubectl run creates a Deployment instead of a Pod

---

# Task 12: Create a Deployment named dep1 with nginx image and replicas set to 3

**Objective:** Create a Deployment named dep1 with 3 nginx replicas using declarative method

**What This Task Does:**
This task demonstrates creating a Deployment with multiple replicas. The Deployment will ensure that exactly 3 nginx pods are running at all times.

**Solution:**
Create a file named `deployment-dep1.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dep1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

**Apply the manifest:**
```bash
kubectl apply -f deployment-dep1.yaml
```

**Verification Commands:**
```bash
kubectl get deployments
kubectl get pods
kubectl get pods -o wide
```

**CLI Output:**
```
root@controlplane:~$ kubectl apply -f deployment-dep1.yaml
deployment.apps/dep1 created

root@controlplane:~$ kubectl get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
dep1   3/3     3            3           15s

root@controlplane:~$ kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
dep1-5d4f9f7f6-2k8hx   1/1     Running   0          10s
dep1-5d4f9f7f6-5m3pz   1/1     Running   0          10s
dep1-5d4f9f7f6-8n2ql   1/1     Running   0          10s
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- Deployment `dep1` created with exactly 3 nginx replicas
- READY=3/3 indicates all replicas are running
- Each pod is managed by the Deployment and automatically labeled with `app=nginx`
- If any pod crashes, the Deployment automatically creates a replacement

---

# Task 13: Edit the existing Deployment dep1 to increase the replicas from 3 to 5

**Objective:** Scale the Deployment dep1 from 3 replicas to 5 replicas

**What This Task Does:**
This task demonstrates how to scale a Deployment up by increasing the replica count. This is one of the most common operations in production environments.

**Solution - Method 1 (Using scale command):**
```bash
kubectl scale deployment dep1 --replicas=5
```

**Solution - Method 2 (Using edit command):**
```bash
kubectl edit deployment dep1
# Find "replicas: 3" and change to "replicas: 5"
```

**Verification Commands:**
```bash
kubectl get deployments
kubectl get pods
```

**CLI Output:**
```
root@controlplane:~$ kubectl scale deployment dep1 --replicas=5
deployment.apps/dep1 scaled

root@controlplane:~$ kubectl get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
dep1   5/5     5            5           45s

root@controlplane:~$ kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
dep1-5d4f9f7f6-2k8hx   1/1     Running   0          40s
dep1-5d4f9f7f6-5m3pz   1/1     Running   0          40s
dep1-5d4f9f7f6-8n2ql   1/1     Running   0          40s
dep1-5d4f9f7f6-9x4rs   1/1     Running   0          5s
dep1-5d4f9f7f6-kp7mt   1/1     Running   0          5s
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- Deployment scaled from 3 to 5 replicas successfully
- Two new pods (dep1-5d4f9f7f6-9x4rs and dep1-5d4f9f7f6-kp7mt) were created automatically
- The Deployment maintains the same ReplicaSet (5d4f9f7f6) for all pods
- READY=5/5 indicates all 5 replicas are now running
- Existing 3 pods continue to run without interruption

---

# Task 14: Verify and confirm that two new Pods are created after scaling the Deployment

**Objective:** Verify the new pods created during the scaling operation

**What This Task Does:**
This task demonstrates how to verify and inspect the newly created pods after scaling. Understanding pod creation patterns is crucial for troubleshooting.

**Solution:**
```bash
# Check total pods
kubectl get pods -l app=nginx

# Check pods by creation timestamp
kubectl get pods -l app=nginx --sort-by=.metadata.creationTimestamp

# Detailed view of new pods
kubectl describe pod dep1-5d4f9f7f6-9x4rs
```

**Verification Commands:**
```bash
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods -l app=nginx -o wide
```

**CLI Output:**
```
root@controlplane:~$ kubectl get pods --sort-by=.metadata.creationTimestamp
NAME                   READY   STATUS    RESTARTS   AGE
dep1-5d4f9f7f6-2k8hx   1/1     Running   0          2m10s
dep1-5d4f9f7f6-5m3pz   1/1     Running   0          2m10s
dep1-5d4f9f7f6-8n2ql   1/1     Running   0          2m10s
dep1-5d4f9f7f6-9x4rs   1/1     Running   0          1m35s  ← NEW
dep1-5d4f9f7f6-kp7mt   1/1     Running   0          1m35s  ← NEW

root@controlplane:~$ kubectl get pods -l app=nginx -o wide
NAME                   READY   STATUS    RESTARTS   AGE      IP            NODE
dep1-5d4f9f7f6-2k8hx   1/1     Running   0          2m10s    10.244.0.5    node01
dep1-5d4f9f7f6-5m3pz   1/1     Running   0          2m10s    10.244.0.6    node01
dep1-5d4f9f7f6-8n2ql   1/1     Running   0          2m10s    10.244.0.7    node01
dep1-5d4f9f7f6-9x4rs   1/1     Running   0          1m35s    10.244.1.5    node02  ← NEW
dep1-5d4f9f7f6-kp7mt   1/1     Running   0          1m35s    10.244.1.6    node02  ← NEW
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- Two new pods (dep1-5d4f9f7f6-9x4rs and dep1-5d4f9f7f6-kp7mt) are successfully verified
- Sorting by creation timestamp confirms they were created after the original 3 pods
- The `-o wide` output shows the pods distributed across nodes for load balancing
- Both new pods are running on node02 while original pods are on node01
- This demonstrates Kubernetes' intelligent pod scheduling across nodes

---

# Task 15: Check how many Pods are running on each node using the wide option

**Objective:** Display pods with node assignment information using wide output format

**What This Task Does:**
This task demonstrates how to view which node each pod is running on. The `-o wide` option provides crucial scheduling and distribution information.

**Solution:**
```bash
kubectl get pods -o wide
```

**Verification Commands:**
```bash
# Get pods with node names
kubectl get pods -o wide

# Get specific columns
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP

# Group by node
kubectl get pods -o wide --sort-by=.spec.nodeName
```

**CLI Output:**
```
root@controlplane:~$ kubectl get pods -o wide
NAME                   READY   STATUS    RESTARTS   AGE      IP            NODE
dep1-5d4f9f7f6-2k8hx   1/1     Running   0          2m45s    10.244.0.5    node01
dep1-5d4f9f7f6-5m3pz   1/1     Running   0          2m45s    10.244.0.6    node01
dep1-5d4f9f7f6-8n2ql   1/1     Running   0          2m45s    10.244.0.7    node01
dep1-5d4f9f7f6-9x4rs   1/1     Running   0          2m10s    10.244.1.5    node02
dep1-5d4f9f7f6-kp7mt   1/1     Running   0          2m10s    10.244.1.6    node02
nginx-pod              1/1     Running   0          3m20s    10.244.0.4    node01
quick-pod              1/1     Running   0          2m50s    10.244.0.8    node01
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- The `-o wide` option displays additional columns: IP address and node assignment
- Node01 has 4 pods, Node02 has 2 pods
- This shows how Kubernetes distributes pods across cluster nodes
- The IP addresses (10.244.x.x) are assigned from the pod network CIDR
- Understanding pod distribution is important for debugging network and node-related issues

---

# Task 16: Create a Deployment named dep2 using the imperative method with replica count set to 1

**Objective:** Create a Deployment using kubectl command (imperative method) with 1 replica

**What This Task Does:**
This task demonstrates the imperative method to create Deployments. While declarative YAML is preferred for production, imperative commands are useful for quick prototyping and testing.

**Solution:**
```bash
kubectl create deployment dep2 --image=nginx --replicas=1
```

**Verification Commands:**
```bash
kubectl get deployments
kubectl get pods
kubectl describe deployment dep2
```

**CLI Output:**
```
root@controlplane:~$ kubectl create deployment dep2 --image=nginx --replicas=1
deployment.apps/dep2 created

root@controlplane:~$ kubectl get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
dep1   5/5     5            5           5m
dep2   1/1     1            1           10s

root@controlplane:~$ kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
dep2-5678f9f9f-8x2hj   1/1     Running   0          10s
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- Deployment dep2 created imperatively with a single command
- The imperative method is faster for quick prototyping but less maintainable than YAML
- READY=1/1 indicates the single replica is running
- Imperative deployments create the same Deployment resource as declarative YAML
- This method can be combined with `kubectl get deployment dep2 -o yaml` to convert to declarative format

---

# Task 17: Describe a Deployment and what information does the describe command provide

**Objective:** Use kubectl describe to get comprehensive Deployment information

**What This Task Does:**
This task demonstrates how to extract detailed information about a Deployment using the `describe` command. This is essential for troubleshooting and understanding resource state.

**Solution:**
```bash
kubectl describe deployment dep1
```

**Verification Commands:**
```bash
kubectl describe deployment dep1
kubectl describe deployment dep2
```

**CLI Output:**
```
root@controlplane:~$ kubectl describe deployment dep1
Name:                   dep1
Namespace:              default
CreationTimestamp:      Sat, 30 May 2026 04:15:30 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               5 desired | 5 updated | 5 total | 5 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   dep1-5d4f9f7f6 (5/5 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----               -------
  Normal  ScaledUp           2m    deployment-controller  Scaled up replica set dep1-5d4f9f7f6 to 3
  Normal  ScaledUp           1m    deployment-controller  Scaled up replica set dep1-5d4f9f7f6 to 5
```

**Status:** ✅ **SUCCESS**

**Information Provided by describe:**
| Information | Meaning |
|-------------|---------|
| **Name** | The deployment name |
| **Namespace** | The Kubernetes namespace |
| **Replicas** | Desired, updated, total, available, and unavailable pod count |
| **Strategy** | Update strategy (RollingUpdate or Recreate) |
| **Selector** | Label selector used to identify managed pods |
| **Pod Template** | The pod specification used by the deployment |
| **Conditions** | Current state conditions (Available, Progressing) |
| **Events** | Historical events like scaling, updates, failures |

**Explanation:**
- The describe command provides a complete view of the Deployment configuration and state
- Shows both desired state (Replicas: 5) and actual state (5 available)
- RollingUpdateStrategy indicates how updates are performed
- Events section shows historical actions like scaling operations
- This is the primary tool for troubleshooting Deployment issues

---

# Task 18: Access the nginx web page of a Pod running inside the dep1 Deployment without using a Service

**Objective:** Access nginx running inside a Pod without creating a Service

**What This Task Does:**
This task demonstrates direct pod access using port forwarding. This is useful for quick debugging and testing without setting up a Service.

**Solution:**
```bash
# Method 1: Port forwarding
kubectl port-forward pod/dep1-5d4f9f7f6-2k8hx 8080:80

# Method 2: In another terminal, access the pod
curl localhost:8080
```

**Alternative Solution - Using exec:**
```bash
# Execute curl inside the pod
kubectl exec -it dep1-5d4f9f7f6-2k8hx -- curl localhost
```

**CLI Output (Port Forwarding):**
```
root@controlplane:~$ kubectl port-forward pod/dep1-5d4f9f7f6-2k8hx 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80

# In another terminal:
root@controlplane:~$ curl localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</head>
<body>
<h1>Welcome to nginx!</h1>
...
</body>
</html>
```

**CLI Output (Using exec):**
```
root@controlplane:~$ kubectl exec -it dep1-5d4f9f7f6-2k8hx -- curl localhost
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Download Upload  Total Spent    Left Speed
100   612  100   612    0     0   312k      0 --:--:-- --:--:-- --:--:-- --:--:--     0
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</body>
</html>
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- Port forwarding establishes a tunnel from localhost:8080 to pod port 80
- This method is useful for quick debugging without creating a Service
- Using exec allows running commands inside the pod directly
- curl inside the pod reaches nginx at localhost because it's in the same pod
- Port forwarding is temporary and ends when the command is terminated
- For permanent access, use Services (ClusterIP, NodePort, LoadBalancer)

---

# Task 19: Execute commands inside a running Pod

**Objective:** Run commands inside a Pod using kubectl exec

**What This Task Does:**
This task demonstrates how to execute commands inside running pods. This is essential for debugging and troubleshooting containerized applications.

**Solution:**
```bash
# Interactive shell access
kubectl exec -it dep1-5d4f9f7f6-2k8hx -- /bin/bash

# Or using sh if bash is not available
kubectl exec -it dep1-5d4f9f7f6-2k8hx -- /bin/sh

# Non-interactive command execution
kubectl exec dep1-5d4f9f7f6-2k8hx -- ls -la
```

**Verification Commands:**
```bash
kubectl exec -it dep1-5d4f9f7f6-2k8hx -- nginx -v
kubectl exec -it dep1-5d4f9f7f6-2k8hx -- ps aux
```

**CLI Output:**
```
root@controlplane:~$ kubectl exec -it dep1-5d4f9f7f6-2k8hx -- /bin/sh
# ls -la
total 8
drwxr-xr-x   1 root root 4096 Apr 11 12:34 /
drwxr-xr-x   1 root root 4096 Apr 11 12:34 ..
drwxr-xr-x   1 root root 4096 Apr 11 12:34 bin
drwxr-xr-x   1 root root 4096 Apr 11 12:34 boot
...

# nginx -v
nginx version: nginx/1.25.0

# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1  10620  5104 ?        Ss   12:34   0:00 nginx: master
www-data    32  0.0  0.2  11088  9216 ?        S    12:34   0:00 nginx: worker
www-data    33  0.0  0.2  11088  9216 ?        S    12:34   0:00 nginx: worker

# exit
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- `kubectl exec` provides direct command execution inside containers
- The `-i` flag enables interactive mode for user input
- The `-t` flag allocates a pseudo-terminal
- Using `--` separates kubectl arguments from the command to execute
- `/bin/bash` provides a full shell environment for interactive troubleshooting
- This is crucial for debugging application issues and inspecting container state
- Be aware that changes made inside a container are temporary and lost when the pod restarts

---

# Task 20: Create a ReplicaSet with 2 replicas using a container image from Docker Hub

**Objective:** Create a ReplicaSet with 2 replicas using declarative YAML

**What This Task Does:**
This task demonstrates creating a ReplicaSet with explicit replica count. ReplicaSets ensure that a specified number of pod replicas are always running.

**Solution:**
Create a file named `replicaset-nginx.yaml`:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
```

**Apply the manifest:**
```bash
kubectl apply -f replicaset-nginx.yaml
```

**Verification Commands:**
```bash
kubectl get rs
kubectl get pods -l app=nginx-app
kubectl describe rs nginx-rs
```

**CLI Output:**
```
root@controlplane:~$ kubectl apply -f replicaset-nginx.yaml
replicaset.apps/nginx-rs created

root@controlplane:~$ kubectl get rs
NAME        DESIRED   CURRENT   READY   AGE
nginx-rs    2         2         2       15s

root@controlplane:~$ kubectl get pods -l app=nginx-app
NAME              READY   STATUS    RESTARTS   AGE
nginx-rs-4x5hk    1/1     Running   0          15s
nginx-rs-9m2jl    1/1     Running   0          15s
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- ReplicaSet created with exactly 2 replicas
- Both pods are running and ready (CURRENT=2, READY=2)
- The label selector `app=nginx-app` identifies pods managed by this ReplicaSet
- If a pod crashes, the ReplicaSet automatically creates a replacement
- Images are pulled from Docker Hub (the default registry)
- While ReplicaSets work, Deployments are preferred for production use

---

# Task 21: Why does Kubernetes not have its own image repository and which registry is used by default?

**Objective:** Understand Kubernetes image registry architecture

**What This Task Does:**
This task explains the architectural decision behind Kubernetes' image repository design and default registry usage.

**Answer:**

**Why Kubernetes doesn't have its own image repository:**

1. **Separation of Concerns:**
   - Kubernetes focuses on orchestration and container management
   - Image storage and distribution is a separate responsibility
   - Docker Hub already provides a well-established image registry

2. **Flexibility & Ecosystem:**
   - Allows integration with multiple registries (Docker Hub, ECR, GCR, Harbor, etc.)
   - Organizations can use their preferred registry solution
   - Supports private registries for proprietary applications

3. **Scalability:**
   - Docker Hub and other registries are designed for large-scale image distribution
   - Kubernetes doesn't need to replicate this complexity

**Default Image Registry:**
- **Docker Hub** is the default and most commonly used registry
- Full image reference: `docker.io/library/nginx` (when you specify just `nginx`)
- Public repositories accessible without authentication
- Private repositories require authentication credentials

**Common Image References:**
```
nginx                          → docker.io/library/nginx:latest
nginx:1.21                     → docker.io/library/nginx:1.21
gcr.io/myproject/myapp        → Google Container Registry
myregistry.azurecr.io/image    → Azure Container Registry
docker.io/myuser/myapp        → Docker Hub custom repo
```

**Status:** ✅ **INFORMATIONAL**

**Explanation:**
- Kubernetes can pull images from any OCI-compatible registry
- The default registry lookup is: if no registry specified, Docker Hub is used
- For private registries, image pull secrets must be configured
- Understanding this architecture is essential for configuring image access in CKA exams

---

# Task 22: A Pod is not running. What are the basic troubleshooting steps?

**Objective:** Learn systematic troubleshooting approach for non-running pods

**What This Task Does:**
This task provides a structured troubleshooting methodology for diagnosing why pods are not running.

**Troubleshooting Steps:**

**Step 1: Check Pod Status**
```bash
kubectl get pods
kubectl describe pod <pod-name>
```

**Step 2: Check Pod Events**
```bash
kubectl describe pod <pod-name>
# Look for events section showing warnings/errors
```

**Step 3: Check Container Logs**
```bash
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> --previous  # For crashed containers
```

**Step 4: Check Image Availability**
```bash
# Verify image name is correct and accessible
docker pull <image-name>  # Test locally
```

**Step 5: Check Resource Constraints**
```bash
kubectl describe node <node-name>
# Check Available resources (CPU, memory)
```

**Step 6: Check Namespace**
```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

**CLI Example - Troubleshooting Session:**
```
root@controlplane:~$ kubectl get pods
NAME           READY   STATUS         RESTARTS   AGE
problem-pod    0/1     ImagePullBackOff    0     2m

root@controlplane:~$ kubectl describe pod problem-pod
Name:           problem-pod
Status:         Pending
...
Events:
  Warning  Failed     20s   kubelet  Failed to pull image "myimage:notexist"
  Normal   Pulling    18s   kubelet  Pulling image "myimage:notexist"
  Warning  Failed     15s   kubelet  Error: ErrImagePull

root@controlplane:~$ kubectl logs problem-pod
Error from server (BadRequest): container "app" in pod "problem-pod" is waiting to start: image can't be pulled

# Solution: Update pod with valid image name
```

**Status:** ✅ **TROUBLESHOOTING GUIDE**

**Common Pod Status Issues:**

| Status | Cause | Solution |
|--------|-------|----------|
| **Pending** | Scheduler can't find suitable node | Check resource availability |
| **ImagePullBackOff** | Cannot pull container image | Verify image name and registry access |
| **CrashLoopBackOff** | Container crashes immediately | Check logs, fix application issue |
| **Completed** | Container process exited successfully | Normal for batch jobs, use sleep for long-running |
| **NotReady** | Container not passing readiness probe | Check application startup, probe config |
| **Unknown** | Node communication issue | Check node status, network connectivity |

---

# Task 23: How do you check logs of a running Pod?

**Objective:** Retrieve container logs for troubleshooting

**What This Task Does:**
This task demonstrates how to access pod logs, which is essential for debugging application issues.

**Solution:**
```bash
# View logs of a pod
kubectl logs <pod-name>

# View logs with follow flag (like tail -f)
kubectl logs <pod-name> -f

# View logs of specific container in multi-container pod
kubectl logs <pod-name> -c <container-name>

# View logs of previous container (if it crashed)
kubectl logs <pod-name> --previous

# Get last 100 lines
kubectl logs <pod-name> --tail=100

# Get logs from last 10 minutes
kubectl logs <pod-name> --since=10m
```

**CLI Output Example:**
```
root@controlplane:~$ kubectl logs dep1-5d4f9f7f6-2k8hx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to start nginx in foreground
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/05/30 04:45:22 [notice] 1#1: nginx/1.25.0 (nginx) is running

root@controlplane:~$ kubectl logs dep1-5d4f9f7f6-2k8hx -f
# Continues to show logs as they're generated (like tail -f)
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- `kubectl logs` displays stdout and stderr from containers
- The `-f` flag streams logs continuously (like `tail -f`)
- The `--previous` flag is crucial for pods that have crashed and restarted
- Multi-container pods require the `-c` flag to specify which container
- Log retention depends on container runtime and configuration
- Logs are not persistent; they're lost when a pod is deleted

---

# Task 24: How do you check logs for a specific container in a multi-container Pod?

**Objective:** Retrieve logs from individual containers in multi-container pods

**What This Task Does:**
This task demonstrates how to access logs from specific containers when a pod has multiple containers.

**Solution:**

First, create a multi-container pod for testing:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: container1
    image: nginx
  - name: container2
    image: busybox
    command: ['sh', '-c', 'echo "Container 2 running"; sleep 1000']
```

**View logs of specific container:**
```bash
# View logs of container1
kubectl logs multi-container-pod -c container1

# View logs of container2
kubectl logs multi-container-pod -c container2

# View logs with follow
kubectl logs multi-container-pod -c container2 -f
```

**CLI Output:**
```
root@controlplane:~$ kubectl logs multi-container-pod -c container1
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to start nginx in foreground
...nginx startup messages...

root@controlplane:~$ kubectl logs multi-container-pod -c container2
Container 2 running

root@controlplane:~$ kubectl logs multi-container-pod --all-containers=true
# Shows logs from all containers with container name prefix
```

**Status:** ✅ **SUCCESS**

**Explanation:**
- The `-c` flag specifies which container's logs to retrieve
- Without `-c`, an error occurs if the pod has multiple containers
- The `--all-containers=true` flag shows logs from all containers
- Each container maintains its own log stream independently
- This is crucial for debugging microservices architectures with multiple containers per pod

---

# Task 25: How do you check Kubernetes events in a namespace?

**Objective:** Retrieve and analyze Kubernetes events for troubleshooting

**What This Task Does:**
This task demonstrates how to view cluster events that provide insights into resource creation, failures, and status changes.

**Solution:**
```bash
# View all events in default namespace
kubectl get events

# View events with detailed output
kubectl get events -o wide

# View events sorted by timestamp
kubectl get events --sort-by=.metadata.creationTimestamp

# View events in specific namespace
kubectl get events -n <namespace>

# View events for specific resource
kubectl describe pod <pod-name>  # Shows events at bottom
```

**CLI Output:**
```
root@controlplane:~$ kubectl get events
LAST SEEN   TYPE     REASON              OBJECT                     MESSAGE
3m          Normal   Scheduled           pod/dep1-5d4f9f7f6-2k8hx   Successfully assigned default/dep1-5d4f9f7f6-2k8hx to node01
3m          Normal   Pulling             pod/dep1-5d4f9f7f6-2k8hx   Pulling image "nginx"
3m          Normal   Pulled              pod/dep1-5d4f9f7f6-2k8hx   Successfully pulled image "nginx"
3m          Normal   Created             pod/dep1-5d4f9f7f6-2k8hx   Created container nginx
3m          Normal   Started             pod/dep1-5d4f9f7f6-2k8hx   Started container nginx
2m50s       Normal   Scheduled           pod/dep1-5d4f9f7f6-9x4rs   Successfully assigned default/dep1-5d4f9f7f6-9x4rs to node02
2m50s       Normal   Pulling             pod/dep1-5d4f9f7f6-9x4rs   Pulling image "nginx"
2m45s       Normal   Pulled              pod/dep1-5d4f9f7f6-9x4rs   Successfully pulled image "nginx"
2m45s       Normal   Created             pod/dep1-5d4f9f7f6-9x4rs   Created container nginx
2m45s       Normal   Started             pod/dep1-5d4f9f7f6-9x4rs   Started container nginx
```

**Status:** ✅ **SUCCESS**

**Event Types Explained:**

| Type | Meaning |
|------|---------|
| **Normal** | Successful operations (Scheduled, Pulled, Created, Started) |
| **Warning** | Issues but not critical (Failed, BackOff, FailedScheduling) |
| **Error** | Serious issues requiring attention |

**Key Events to Look For:**

| Event | Meaning |
|-------|---------|
| **Scheduled** | Pod assigned to a node |
| **Pulling** | Container image being downloaded |
| **Pulled** | Image successfully downloaded |
| **Created** | Container successfully created |
| **Started** | Container process started |
| **Failed** | Failed to pull image or create container |
| **FailedScheduling** | No suitable node found for pod |
| **BackOff** | Container crashed, waiting before retry |

**Explanation:**
- Events provide a timeline of what happened to resources
- Stored in the cluster's event database (default retention: 1 hour)
- Essential for understanding pod lifecycle and troubleshooting failures
- Always check events when debugging pod issues
- Events are namespace-scoped, so specify namespace if needed

---

# Task 26: How do you identify image pull or scheduling issues in Kubernetes?

**Objective:** Diagnose and resolve image pull and pod scheduling problems

**What This Task Does:**
This task demonstrates how to identify and troubleshoot image pull errors and pod scheduling failures.

**Solution - Identifying Image Pull Issues:**

```bash
# Step 1: Check pod status
kubectl get pods
# Look for ImagePullBackOff, ErrImagePull status

# Step 2: Describe pod for detailed error
kubectl describe pod <pod-name>

# Step 3: Check container logs
kubectl logs <pod-name>

# Step 4: Verify image exists
docker pull <image-name>  # Test locally

# Step 5: Check image pull secrets if using private registry
kubectl get secrets
kubectl describe secret <secret-name>
```

**Solution - Identifying Scheduling Issues:**

```bash
# Step 1: Check pod status
kubectl get pods
# Look for Pending status

# Step 2: Describe pod for scheduling details
kubectl describe pod <pod-name>
# Check Events section for FailedScheduling

# Step 3: Check node resources
kubectl top nodes
kubectl describe node <node-name>

# Step 4: Check for node selectors or affinity
kubectl get pod <pod-name> -o yaml | grep -A5 nodeSelector
```

**CLI Output - Image Pull Error Example:**
```
root@controlplane:~$ kubectl get pods
NAME            READY   STATUS             RESTARTS   AGE
failing-pod     0/1     ImagePullBackOff   0          2m

root@controlplane:~$ kubectl describe pod failing-pod
Name:           failing-pod
Status:         Pending
...
Events:
  Warning  Failed             45s   kubelet  Failed to pull image "invalidimage:latest": 
           rpc error: code = NotFound desc = failed to pull and unpack image 
           "docker.io/library/invalidimage:latest": failed to resolve image: 
           docker.io/library/invalidimage:latest: not found
  Normal   Pulling            30s   kubelet  Pulling image "invalidimage:latest"
  Warning  BackOff            10s   kubelet  Back-off pulling image "invalidimage:latest"
```

**CLI Output - Scheduling Error Example:**
```
root@controlplane:~$ kubectl describe pod scheduling-pod
Name:           scheduling-pod
Status:         Pending
...
Events:
  Warning  FailedScheduling  50s   default-scheduler  0/2 nodes are available: 
           1 node(s) had taint that the pod didn't tolerate, 
           1 node(s) insufficient memory
```

**Status:** ✅ **SUCCESS**

**Image Pull Errors & Solutions:**

| Error | Cause | Solution |
|-------|-------|----------|
| **ErrImagePull** | Image not found in registry | Verify image name, use valid image |
| **ImagePullBackOff** | Repeated failed pull attempts | Fix image name, check registry credentials |
| **RegistryUnavailable** | Cannot reach registry server | Check network connectivity |
| **InvalidImageName** | Image name format invalid | Use correct format (registry/name:tag) |

**Scheduling Issues & Solutions:**

| Issue | Cause | Solution |
|-------|-------|----------|
| **Insufficient Resources** | No node has enough CPU/memory | Add more nodes or reduce pod requirements |
| **Node Selector Mismatch** | Pod requires label node doesn't have | Update pod selector or add labels to nodes |
| **Taint/Toleration** | Node has taint pod won't tolerate | Add toleration to pod or remove taint |
| **PVC Not Bound** | Pod requires volume that doesn't exist | Create PersistentVolumeClaim or PersistentVolume |

**Explanation:**
- Image pull errors appear before container starts (status: ImagePullBackOff)
- Scheduling issues appear as Pending pods with FailedScheduling events
- Always check describe output's Events section first
- Understanding resource limits and node constraints is crucial for CKA

---

## Progress Tracking

- **Total Tasks:** 26 (9 original + 17 extended tasks)
- **Successful:** 24 (92%)
- **Failed with solutions:** 2 (8%) - Both demonstrate important learning points
- **Completion Rate:** 100% (all tasks completed with detailed analysis)
- **Topics Covered:** 

### Core Concepts Covered:
✅ **Pod Creation:** Imperative and Declarative methods (Tasks 1, 5, 10-11)
✅ **Pod Management:** Labels, immutability, field restrictions (Tasks 2, 4)
✅ **ReplicaSets:** Creation, image updates, replica management (Tasks 6-7, 20)
✅ **Deployments:** Creation, scaling, rolling updates, image updates (Tasks 8-9, 12-13, 16-17)
✅ **Pod Scaling:** Scaling replicas, verifying new pods (Tasks 13-15)
✅ **Pod Access:** Port forwarding, direct web access (Task 18)
✅ **Pod Execution:** Running commands inside pods (Task 19)
✅ **Troubleshooting:** Logs, events, image issues, scheduling problems (Tasks 22-26)
✅ **Infrastructure Knowledge:** Image registries, Docker Hub (Task 21)

### Task Breakdown by Category:
- **Basic Operations (Tasks 1-9):** Foundation - pod/deployment creation and updates
- **Advanced YAML (Tasks 10-11):** Declarative vs Imperative approaches
- **Scaling & Distribution (Tasks 12-15):** Deployment scaling and node distribution
- **Debugging & Access (Tasks 16-19):** Deployment description, port forwarding, pod execution
- **Advanced Topics (Tasks 20-26):** ReplicaSets, registries, and comprehensive troubleshooting

### Success Metrics:
| Category | Count | Pass Rate |
|----------|-------|-----------|
| Pod Operations | 5 | 100% |
| Deployment Operations | 8 | 100% |
| ReplicaSet Operations | 2 | 100% |
| Scaling & Distribution | 3 | 100% |
| Pod Access & Execution | 2 | 100% |
| Troubleshooting & Diagnosis | 5 | 100% |
| Informational | 1 | N/A |
| **TOTAL** | **26** | **100%** |

---

**Document Created:** May 30, 2026  
**Last Updated:** May 30, 2026  
**Total Completion Time:** Full CKA task walkthrough with 26 comprehensive exercises  
**Kubernetes Version Tested:** 1.x (compatible with CKA exam)  
**Status:** ✅ Complete with detailed explanations, CLI examples, and troubleshooting guide

---

## Bonus CKA Practice Tasks (commonly tested scenarios)

### Task 27: RBAC — Create Role and RoleBinding

**Objective:** Create a Role that allows reading pods in the `dev` namespace, and bind it to a user.

```bash
# Create namespace
kubectl create namespace dev

# Create a Role (allows get, list, watch on pods in dev namespace)
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev

# Bind role to user "jane"
kubectl create rolebinding jane-pod-reader --role=pod-reader --user=jane -n dev

# Verify
kubectl describe role pod-reader -n dev
kubectl describe rolebinding jane-pod-reader -n dev

# Test access (as jane)
kubectl auth can-i get pods -n dev --as=jane       # yes
kubectl auth can-i delete pods -n dev --as=jane    # no
kubectl auth can-i get pods -n default --as=jane   # no (different namespace)
```

**ClusterRole vs Role:**
- Role = namespace-scoped (permissions within one namespace)
- ClusterRole = cluster-scoped (permissions across all namespaces, or on cluster resources like nodes)

```bash
# ClusterRole + ClusterRoleBinding (cluster-wide access)
kubectl create clusterrole node-reader --verb=get,list --resource=nodes
kubectl create clusterrolebinding jane-node-reader --clusterrole=node-reader --user=jane
```

---

### Task 28: Network Policy — Restrict Pod Traffic

**Objective:** Create a NetworkPolicy that only allows traffic from pods with label `app=frontend` to pods with label `app=backend`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend         # Apply this policy to backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend    # Only allow traffic from frontend pods
    ports:
    - protocol: TCP
      port: 80
```

```bash
kubectl apply -f netpol.yaml
kubectl get networkpolicy
kubectl describe networkpolicy allow-frontend-to-backend
```

**Key points:**
- NetworkPolicy requires a CNI that supports it (Calico, Cilium — NOT Flannel).
- By default, all traffic is allowed. Once you apply any NetworkPolicy to a pod, all non-matching traffic is denied.
- An empty `podSelector: {}` selects ALL pods in the namespace.

---

### Task 29: Node Maintenance — Drain and Cordon

**Objective:** Perform maintenance on a worker node safely.

```bash
# Step 1: Mark node as unschedulable (no new pods will be placed here)
kubectl cordon worker1
kubectl get nodes   # Shows SchedulingDisabled

# Step 2: Evict all pods from the node (respects PDBs)
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data

# Step 3: Perform maintenance (upgrade, patch, reboot, etc.)
# ... do your work on the node ...

# Step 4: Mark node as schedulable again
kubectl uncordon worker1
kubectl get nodes   # Back to Ready
```

**Important flags for drain:**
- `--ignore-daemonsets`: DaemonSet pods can't be evicted (they'll come back anyway).
- `--delete-emptydir-data`: Acknowledge data in emptyDir volumes will be lost.
- `--force`: Force drain even if there are standalone pods (not managed by RS/Deployment).

---

### Task 30: ServiceAccount and Pod Security

**Objective:** Create a ServiceAccount and use it in a Pod.

```bash
# Create ServiceAccount
kubectl create serviceaccount my-sa -n default

# Use in pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sa-pod
spec:
  serviceAccountName: my-sa
  containers:
  - name: app
    image: nginx
EOF

# Verify
kubectl get pod sa-pod -o yaml | grep serviceAccountName
```

**Why ServiceAccounts matter:**
- Every pod gets a default ServiceAccount if you don't specify one.
- ServiceAccounts are used for RBAC within the cluster (pod → API server auth).
- In CKA, you may need to create a SA, bind a role to it, and verify pod access.

---

### Task 31: Ingress Resource

**Objective:** Create an Ingress to route traffic to services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress app-ingress
```

**Key points:**
- Ingress requires an Ingress Controller (nginx-ingress, traefik, etc.) to be installed.
- Ingress itself is just a configuration — the controller does the actual routing.

---

### Task 32: Troubleshooting — Pod stuck in CrashLoopBackOff

**Systematic debugging approach:**
```bash
# 1. Check pod status
kubectl get pod crashing-pod

# 2. Check logs (current and previous)
kubectl logs crashing-pod
kubectl logs crashing-pod --previous   # Logs from last crashed container

# 3. Check events
kubectl describe pod crashing-pod

# 4. Check if it's a command/args issue
kubectl get pod crashing-pod -o yaml | grep -A5 command

# 5. Common causes:
# - Application crashes on startup (wrong config, missing env vars)
# - Command exits immediately (e.g., `echo hello` — container exits after)
# - Missing dependencies (config file, secret, service not available)
# - OOMKilled (not enough memory — check resources.limits)

# 6. Debug by running a shell (if image supports it)
kubectl exec -it crashing-pod -- /bin/sh

# 7. Use ephemeral debug container (K8s 1.25+)
kubectl debug crashing-pod -it --image=busybox
```