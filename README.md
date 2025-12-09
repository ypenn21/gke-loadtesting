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
