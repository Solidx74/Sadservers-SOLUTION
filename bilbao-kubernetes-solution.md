# SadServers – Bilbao: Basic Kubernetes Problems

## Challenge Overview

- **Challenge:** Bilbao – Basic Kubernetes Problems
- **Difficulty:** Easy
- **Operating system:** Debian 13
- **Kubernetes distribution:** K3s
- **Goal:** Restore an Nginx Deployment and make it reachable through its LoadBalancer Service.
- **Required test:** `curl 10.43.216.196` must return the default Nginx Welcome page.
- **Privilege constraint:** No root / sudo access required.

## Initial Symptoms

The Kubernetes cluster contained one node (`node1`), an Nginx Deployment, and a LoadBalancer Service.

I started by checking the resources:

```bash
k get pods
k get deployments
k get services
```

The Nginx Pod was stuck in `Pending`, and the Deployment had no available replicas:

```text
NAME                                READY   STATUS
nginx-deployment-7c4b9dc968-m8d29   0/1     Pending
```

The Service already existed:

```text
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
nginx-service   LoadBalancer   10.43.216.196   10.1.13.151   80:30422/TCP
```

## Investigation

I inspected the Pod and its scheduling events:

```bash
k describe pods
```

The first scheduling error was:

```text
0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector.
```

I checked the node labels:

```bash
k get nodes --show-labels
```

The only node, `node1`, did not have the label `disk=ssd`.

I then inspected the manifest:

```bash
cat manifest.yml
```

The Deployment contained this selector:

```yaml
nodeSelector:
  disk: ssd
```

This forced Kubernetes to schedule the Pod only on a node with the `disk=ssd` label. Since no available node had that label, the Pod could not be scheduled.

## Fix 1 – Remove the invalid node selector

I edited `manifest.yml` and removed:

```yaml
nodeSelector:
  disk: ssd
```

The Deployment's Pod template no longer required a node label that did not exist in the cluster.

I applied the updated manifest:

```bash
k apply -f manifest.yml
```

A new Pod was created, and its scheduling events showed that the node-selector problem was resolved. However, the new Pod was still `Pending`.

## Fix 2 – Reduce the excessive memory request

I inspected the new Pod:

```bash
k describe pod <new-pod-name>
```

Its Events section showed a different error:

```text
0/1 nodes are available: 1 Insufficient memory.
```

This revealed a second, independent scheduling problem.

The original container resource configuration requested 2000 MiB of memory:

```yaml
resources:
  limits:
    memory: 2000Mi
    cpu: 100m
  requests:
    cpu: 100m
    memory: 2000Mi
```

The node did not have enough allocatable memory available to satisfy that request.

I reduced the Nginx container's resource settings to:

```yaml
resources:
  limits:
    memory: 256Mi
    cpu: 100m
  requests:
    cpu: 50m
    memory: 128Mi
```

This lowered the requested memory while retaining resource limits for the container.

> These values are the configuration used for this challenge, not universal production recommendations. Resource requests and limits should be chosen based on actual workload requirements and available capacity.

## Final Deployment

After both changes, I applied the manifest again:

```bash
k apply -f manifest.yml
```

I checked the Deployment rollout:

```bash
k rollout status deployment/nginx-deployment --timeout=90s
```

Then inspected the Pods and Service:

```bash
k get pods
k get deployments
k get services
k get endpoints
```

The expected healthy state was:

- Nginx Pod: `1/1 Running`
- Deployment: `1/1 Available`
- Service: `LoadBalancer`
- Service endpoints: a ready Nginx Pod on port 80

## Verification

The challenge's required test was:

```bash
curl 10.43.216.196
```

A successful response should contain the default Nginx Welcome page HTML, including:

```html
<title>Welcome to nginx!</title>
```

After confirming the response, I used the **Check My Solution** button on SadServers to validate the challenge.

## Root Causes and Lessons Learned

There were two separate causes of the scheduling failure:

1. **Invalid `nodeSelector`:** The Deployment required `disk=ssd`, but the only available node did not have that label.
2. **Excessive memory request:** After removing the selector, Kubernetes could consider the node, but it could not schedule the Pod because the 2000 MiB memory request exceeded the available allocatable capacity.

### Useful diagnostic commands

| Command | Purpose |
|---|---|
| `k get pods` | Check Pod status |
| `k get deployments` | Check Deployment availability |
| `k get services` | Inspect Services and cluster IPs |
| `k get nodes --show-labels` | Inspect node labels |
| `k describe pod <pod-name>` | Diagnose Pod scheduling and runtime issues |
| `k describe node node1` | Inspect node conditions and resource allocation |
| `k get events --sort-by=.metadata.creationTimestamp` | Review recent cluster events |
| `k apply -f manifest.yml` | Apply manifest changes |
| `k rollout status deployment/nginx-deployment` | Monitor Deployment rollout |
| `curl 10.43.216.196` | Verify Nginx through the ClusterIP |

## Key Takeaway

A Pod in `Pending` state is not necessarily failing because of its container image or application. Kubernetes scheduling events are an important first diagnostic step.

In this challenge, removing the invalid node selector exposed the second issue: insufficient memory. Fixing both the scheduling constraint and the resource request allowed the Nginx Deployment to become schedulable and serve traffic through its LoadBalancer Service.
