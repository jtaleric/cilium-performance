# Cilium Service Mesh Performance Test Plan

## Frequency we will execute this test plan
Before each Cilium release

## Test Plan Description
Each of these tests will have multiple metrics we will collect and compare.

| Workload | Main Metric | Secondary Metric |
|----------|-------------|------------------|
| Nighthawk   | Throughput (rps) | CPU usage  |
| Nighthawk   | Latency (ms) | CPU usage       |

> CPU usage will be for the Cilium pods as well as the overall node CPU usage. We will use Prometheus to collect this information.

> CPU and Memory will also be summarized across nodes as a single metric


## Tooling
### Tools to be deployed on the SUT

- Nighthawk-http-server & Nighthawk client \
*How do we install nighthawk-http-server?*\
First label one of your nodes with `app=workload`
 then `kubectl create -f scripts/service-mesh/sm-perf.yaml`

 - Prometheus \
*How do we install prom?*\
`helm install prometheus prometheus-community/prometheus --namespace prometheus  --namespace prometheus --create-namespace`

   We will use Prom to monitor CPU/Mem/IO of the cilium pods, the test pods and the nodes.

- Grafana \
*How do we install Grafana?*\
`git clone https://github.com/cloud-bulldozer/performance-dashboards; cd performance-dashboards; make; cd dittybopper; ./k8s-deploy.sh`

   **WIP but having a Cilium specific dashboard showing Overall System metrics + Cilium Specific Metrics.  https://github.com/jtaleric/performance-dashboards/commit/1ac3a9b21c33c3dc2516f21428ebd4ea4a6396c1#diff-b9c0833d120841436640df16492f6e79f8e99e25232047aadcb63bd4378193aeR339-R375

- benchmark-comparison \
  *How do install benchmark-comparison* \
  https://github.com/cloud-bulldozer/benchmark-comparison#usage

### Tooling Setup
#### kube-burner
We will use kube-burner to scrape prometheus and store the metrics in ES.

We will need to capture the prometheus token

> $ export token=$(kubectl get secrets -n prometheus prometheus-server-<secret> -o go-template='{{index .data "ca.crt"}}')

We will also need to forward the prometheus server port to the machine running the tests

> kubectl --namespace prometheus port-forward <prom server pod> 9090

We need to create a kube-burner config to define the ES server to store the data. I store this as `burner.yaml`
```yaml
---
global:
  writeToFile: true
  metricsDirectory: collected-metrics
  indexerConfig:
    enabled: true
    esServers: [https://user:pass@es-server:9243]
    insecureSkipVerify: false
    defaultIndex: kube-burner
    type: elastic

```

For the kube-burner metrics, use [rook-node-v2.yaml](https://gist.github.com/jtaleric/c2a9c4974cd82c358ce4e1b5fd2d40f3#file-rook-node-v2-yaml)


## System Under Test (SUT) definition
### Managed Platform testing
Service Mesh Performance testing will be in a single Region and Zone.

We will use `cilium install` which will capture SUT details such as the platform we are running on, to help build the most performant configuration for Cilium to run.

| IaaS | SaaS | Controller Sizes | Node Sizes |  Region |
|------|------|------------------|------------|---------|
| GCP | GKE | n/a | n2-standard-16 | us-central1-a |


## Scenarios

### Baseline - No L7 Policy, Ingress and Hubble
Envoy shouldn't even bee in the picture here, as we haven't created an L7 policy or defined an ingress.

#### HTTP Request
  ```mermaid
    graph LR;
      subgraph NodeA
        LoadPod
      end
      subgraph NodeB
        Service
        WebServer
      end
    Service---WebServer
    LoadPod---Service
 ```

 ### With L7 Policy
 With l7 Policy, Envoy will only be created on the node that has the pod which matches the label for the l7 policy.
 #### HTTP Request
   ```mermaid
    graph LR;
      subgraph NodeA
        LoadPod
      end
      subgraph NodeB
        Envoy
        Service
        WebServer
      end
    Envoy---Service---WebServer
    LoadPod---Envoy
```

 ### With Ingress
Creating an Ingress policy will create Envoy instances across the fleet of nodes in the cluster.
  #### HTTP Request
   ```mermaid
    graph LR;
      subgraph NodeA
        LoadPod
      end
      subgraph NodeB
        Service
        WebServer
      end
    Envoy---Service---WebServer
    LoadPod---Envoy
```

# Test Cases
Test Cases assume you have enabled all the tooling mentioned above. If not, please refer back to the Tooling section.
## 1.0 Latency and Throughput
### 1.1 Determine Latency at fixed RPS
#### Steps to reproduce
1. Deploy Cloud in GKE using (https://github.com/jtaleric/tinker/tree/main/clouds/gke/kernel-swap)
   1. Using parameters
    - export TYPE=n2-standard-16
    - export KERNEL=5.16
    - export NODES=5
    - export BOOTSTRAP=true
2. Once the cluster is up, ensure the kernel version, `kubectl get nodes -o wide`
3. Label a single node with `app=workload` this will be where Nighthawk runs, and we will steer all other pods to other nodes.
4. Follow the Kube-burner & Prometheus setup in the [tooling section](/Tooling-setup)
5. `kubectl create -f scripts/service-mesh/sm-perf.yaml`
6. Exec into the load pod `kubectl exec -it <pod> - sh`
7. Ensure connectivity to the svc ip is working with a simple 10sec test `nighthawk_client --duration 10 --simple-warmup --rps 1000 --connections 1 --concurrency auto -v error http://baseline:9080/`

*Good Output:*
```
Counter                                 Value       Per second
benchmark.http_2xx                      7631        763.10
```
*Bad Output:*
```
Counter                                 Value       Per second
cluster_manager.cluster_added           4           inf
```
   > This will run with a single connection and attempt to achieve 1000rps.
8. Setup your env vars for the test
    ```
    NAME    - Name of the test, ie l7Policy
    STORE   - Store the system metrics to ES w/ kube-burner
    ```
9. Kick off the test script `scripts/service-mesh/run-targeted.sh`

The `run-targeted.sh` script will generate some log artifacts that contain the results.

      > grep benchmark_http_client.latency_2xx -A9 * | grep 0.95

      To capture the 95%tile latency observed during the workload

      > grep benchmark.http_2xx *

      To capture the rps observed during the workload

Details about the results :
- There are 3 tests that run, 160rps, 1600rps and 16000rps.
- There will be 5 samples per tests mentioned above

#### Metrics
- Requests Per Second (rps)

    Total number of requests that Nighthawk was able to achieve.

- Latency (us)

   Response time observed during the workload

- CPU Utilization (% idle)

   While we capture all CPU utilization, we will focus on the idle CPU which will give us an idea of how much available CPU is left for other applications while under load.

- Memory Utilization (% available)
### 1.2 Baseline - No Policy or Ingress defined
#### Steps to reproduce
1. Deploy Cloud in GKE using (https://github.com/jtaleric/tinker/tree/main/clouds/gke/kernel-swap)
   1. Using parameters
    - export TYPE=n2-standard-16
    - export KERNEL=5.16
    - export NODES=5
    - export BOOTSTRAP=true
2. Once the cluster is up, ensure the kernel version, `kubectl get nodes -o wide`
3. Label a single node with `app=workload` this will be where Nighthawk runs, and we will steer all other pods to other nodes.
4. Follow the Kube-burner & Prometheus setup in the [tooling section](/Tooling-setup)
5. `kubectl create -f scripts/service-mesh/sm-perf.yaml`
6. Exec into the load pod `kubectl exec -it <pod> - sh`
7. Ensure connectivity to the svc ip is working with a simple 10sec test `nighthawk_client --duration 10 --simple-warmup --rps 1000 --connections 1 --concurrency auto -v error http://baseline:9080/`

*Good Output:*
```
Counter                                 Value       Per second
benchmark.http_2xx                      7631        763.10
```
*Bad Output:*
```
Counter                                 Value       Per second
cluster_manager.cluster_added           4           inf
```
   > This will run with a single connection and attempt to achieve 1000rps.
8. Setup your env vars for the test
    ```
    NAME    - Name of the test, ie l7Policy
    STORE   - Store the system metrics to ES w/ kube-burner
    ```
9. Kick off the test script `scripts/service-mesh/run.sh`

The `run.sh` script will generate some log artifacts that contain the results.

      > grep benchmark_http_client.latency_2xx -A9 * | grep 0.95

      To capture the 95%tile latency observed during the workload

      > grep benchmark.http_2xx *

      To capture the rps observed during the workload

#### Metrics
- Requests Per Second (rps)

    Total number of requests that Nighthawk was able to achieve.

- Latency (us)

   Response time observed during the workload

- CPU Utilization (% idle)


   While we capture all CPU utilization, we will focus on the idle CPU which will give us an idea of how much available CPU is left for other applications while under load.

- Memory Utilization (% available)
### 1.3 Baseline - l7 Policy
#### Steps to reproduce
1. Deploy Cloud in GKE using (https://github.com/jtaleric/tinker/tree/main/clouds/gke/kernel-swap)
   1. Using parameters
    - export TYPE=n2-standard-16
    - export KERNEL=5.16
    - export NODES=5
    - export BOOTSTRAP=true
2. Once the cluster is up, ensure the kernel version, `kubectl get nodes -o wide`
3. Label a single node with `app=workload` this will be where Nighthawk runs, and we will steer all other pods to other nodes.
4. Follow the Kube-burner & Prometheus setup in the [tooling section](/Tooling-setup)
5. `kubectl create -f scripts/service-mesh/sm-perf.yaml`
6. Exec into the load pod `kubectl exec -it <pod> - sh`
7. Apply the l7 policy (Example below)
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l7rule"
spec:
  description: "Allow HTTP GET / from app=workload to app=baseline"
  endpointSelector:
    matchLabels:
      app: baseline
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: workload
    toPorts:
    - ports:
      - port: "9080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/"
   ```
8. Ensure connectivity to the svc ip is working with a simple 10sec test `nighthawk_client --duration 10 --simple-warmup --rps 1000 --connections 1 --concurrency auto -v error http://baseline:9080/`

*Good Output:*
```
Counter                                 Value       Per second
benchmark.http_2xx                      7631        763.10
```
*Bad Output:*
```
Counter                                 Value       Per second
cluster_manager.cluster_added           4           inf
```
   > This will run with a single connection and attempt to achieve 1000rps.
8. Setup your env vars for the test
    ```
    NAME    - Name of the test, ie l7Policy
    STORE   - Store the system metrics to ES w/ kube-burner
    ```
9. Kick off the test script `scripts/service-mesh/run.sh`

The `run.sh` script will generate some log artifacts that contain the results.

      > grep benchmark_http_client.latency_2xx -A9 * | grep 0.95

      To capture the 95%tile latency observed during the workload

      > grep benchmark.http_2xx *

      To capture the rps observed during the workload

#### Metrics
- Requests Per Second (rps)

    Total number of requests that Nighthawk was able to achieve.

- Latency (us)

   Response time observed during the workload

- CPU Utilization (% idle)


   While we capture all CPU utilization, we will focus on the idle CPU which will give us an idea of how much available CPU is left for other applications while under load.

- Memory Utilization (% available)

## 2.0 Control Plane Performance
### 2.1 Naked Pod Creation Latency
#### Steps to reproduce
1. Deploy Cloud in GKE using (https://github.com/jtaleric/tinker/tree/main/clouds/gke/kernel-swap)
   1. Using parameters
    - export TYPE=n2-standard-16
    - export KERNEL=5.16
    - export NODES=5
    - export BOOTSTRAP=true

2. Follow the kube-burner & Prometheus setup in the [tooling section](/Tooling-setup)
3. To deploy the naked-pod workload, refer to the `scripts/service-mesh/naked-density-tmpl.yml`
   1. Update the parameters (such as the User/Password for your ES Server)
4. Execute the benchmark by running `kube-burner index -c naked-density-tmpl.yml -m rook-node-v2.yaml -t $token -u http://127.0.0.1:9090`
   1. Where token is the prometheus token captured in step 2.
   2. `http://127.0.0.1 ` is the prometheus forward for kube-burner to scrape prometheus.
   3. `rook-node-v2.yaml` is the prometheus scraping config which kube-burner will use to capture and label the metrics.
#### Metrics
This workload, we are strictly interested in looking at the podLatency, which is the time it takes a pod to become "Ready".
1. Install benchmark-comparison (https://github.com/cloud-bulldozer/benchmark-comparison#usage)
2. Capture the UUIDs you want to inspect the podLatency for.
3. `touchstone_compare -url https://user:password@es:9243 -u <uuid-1> <uuid-2> --config scripts/service-mesh/podlatency.yaml`
   1. This will query the provided ES server, and look for the two uuids referenced. Specifically querying for podlatency.
   2. example output
```
+--------------------------------+-----------------+----------+----------------------------+---------------------------+
|           metricName           |  quantileName   |  metric  | 1650993041-cilium-100-run1 | 1650997776-istio-100-run1 |
+--------------------------------+-----------------+----------+----------------------------+---------------------------+
| podLatencyQuantilesMeasurement | ContainersReady | avg(P99) |           5703.0           |          92274.0          |
| podLatencyQuantilesMeasurement |   Initialized   | avg(P99) |            77.0            |          73599.0          |
| podLatencyQuantilesMeasurement |  PodScheduled   | avg(P99) |            31.0            |           40.0            |
| podLatencyQuantilesMeasurement |      Ready      | avg(P99) |           5703.0           |          92274.0          |
+--------------------------------+-----------------+----------+----------------------------+---------------------------+
```