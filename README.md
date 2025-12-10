#GKE L400 labs

[L400 labs](https://explore.qwiklabs.com/course_templates/1441196)

custom metrics adapter..
https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-stackdriver/master/custom-metrics-stackdriver-adapter/deploy/production/adapter_new_resource_model.yaml

GKE Autopilot Clusters:

Yes, the Google Cloud Managed Service for Prometheus (GMP) components (running in the gmp-system namespace) are enabled by default in Autopilot clusters running GKE version 1.25 or greater.
GKE Standard Clusters:

New Clusters: GMP managed collection is also enabled by default when you create a new GKE Standard cluster.
Existing Clusters: For Standard clusters created before this default was put in place, GMP is NOT enabled by default, and the gmp-system pods would only exist if you or an administrator manually enabled Managed Service for Prometheus on that cluster.
So, you will find the gmp-system pods in:

All GKE Autopilot clusters (v1.25+).
Any GKE Standard cluster created after the default enablement was rolled out.
Any older GKE Standard cluster where GMP has been manually enabled.
You will not find them in an older GKE Standard cluster where GMP has not been explicitly enabled.

```
kubectl get pods -n gmp-system
kubectl get daemonset -n gmp-system
kubectl get deployment -n gmp-system
kubectl get statefulset -n gmp-system
```

# gke-loadtesting
This document summarizes the discussion about the `gke-workload-autoscaling` test setup.

### 1. `load-script.sh` Functionality

The `load-script.sh` script is designed to test a web service by generating a "staircase" pattern of HTTP traffic.

- **Operation:** It runs in an infinite `while true` loop, meaning it never stops on its own.
- **Load Generation:** Within the loop, it executes a test cycle composed of several steps (e.g., 3 steps by default).
- **Stepped Load:** It starts with a base number of requests per second (QPS) and concurrency. In each subsequent step of the cycle, it multiplies the QPS and concurrency by a scale factor (e.g., 2x), progressively increasing the load.
- **Reset:** After completing all steps in a cycle, the script automatically resets the traffic to the initial base level and starts the entire test cycle over again.

### 2. Interaction with Kubernetes HPA (Horizontal Pod Autoscaler)

The script does not directly communicate with or adjust to the HPA. The interaction is indirect:

1.  **Script's Role (Cause):** The script sends its increasing traffic to a stable Kubernetes Service endpoint. It is unaware of how many pods are running behind that service.
2.  **Service's Role (Distribution):** The Kubernetes Service receives the traffic and acts as a load balancer, distributing it across all available pods for the deployment.
3.  **HPA's Role (Effect):** The HPA monitors the resource utilization (e.g., CPU) of the pods. As the script increases the traffic, the load on the pods rises. When the average utilization exceeds the HPA's target, the HPA scales up the number of pods to handle the load.

To trigger max scaling, the script's configured QPS, scale factor, and number of cycles must be aggressive enough to keep pod utilization high even as new pods are added.

### 3. Calculating Processed Requests

The script calculates the total number of requests after a full test cycle is complete:

1.  For each step in a cycle, it executes the `hey` load-testing tool and captures its summary output.
2.  It parses the `hey` output to extract the actual "Requests/sec" achieved and the "Total" run duration.
3.  It calculates the requests for that step by multiplying: `Requests/sec * Total duration`.
4.  It adds this number to a running total for the entire cycle, which is printed at the end.
