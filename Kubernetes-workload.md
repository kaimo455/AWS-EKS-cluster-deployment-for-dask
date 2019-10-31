- [**Deployment, Jobs and Scaling**](#deployment-jobs-and-scaling)
  - [Deployment](#deployment)
  - [Updating deployments](#updating-deployments)
  - [Managing deployments](#managing-deployments)
  - [Jobs and CronJobs](#jobs-and-cronjobs)
  - [Controlling pod placement](#controlling-pod-placement)
  - [Getting software into your cluster](#getting-software-into-your-cluster)
- [**Kubernetes Engin Networking**](#kubernetes-engin-networking)
  - [Pod Networking](#pod-networking)
  - [Services](#services)
  - [Service types and Load Balancers](#service-types-and-load-balancers)
  - [Ingress Resource](#ingress-resource)
  - [Container-Native Load Balancing](#container-native-load-balancing)
  - [Network Security](#network-security)
- [**Persistent Data and Storgae**](#persistent-data-and-storgae)
  - [Volumns](#volumns)
  - [StatefulSets](#statefulsets)
  - [ConfigMaps](#configmaps)
  - [Secrect](#secrect)


# **Deployment, Jobs and Scaling**

## Deployment

Deployment describes a desired state of pods.

`.yaml` -> `Deployment object` -> `Deployment controller` -> `Node (Pod1, Pod2, ...)`

- Ways to create deployments

```bash
kubectl apply -f [DEPLOYMENT_FILE]
```

```bash
kubectl run [DEPLOYMENT_NAME] \
--image [IMAGE]:[TAG] \
--replicas 3 \
--labels [KEY]=[VALUE] \
--ports 8080 \
--generator deployment/apps.v1 \
--save-config
```

- Use kubectl to inspect your Deployment

```bash
kubectl get/describe deployment DEPLOYMENT_NAME [-o wide/yaml]
```

- Scaling a Deployment manually

```bash
kubectl scale deployment DEPLOYMENT_NAME -replicas=5
```

- Autoscaling a Deployment

```bash
kubectl autoscale deployment DEPLOYMENT_NAME --min=5 --max=15 --cpu-precent=75
```

## Updating deployments

```bash
kubectl apply -f DEPLOYMENT_FILE

kubectl set image deployment DEPLOYMENT_NAME IMAGE IMAGE:TAG

kubectl edit deployment/DEPLOYMENT_NAME
```

- rolling updates

```yaml
kind: deployment
spec:
  strategy:
    rollingUpdate:
      maxSurge: 5
      maxUnavailable: 30%
```

- blue-green deployments

when updating, a completely new deployment is created. Once new application is ready, all traffic is shifted to new version.

- Canary deployment

Based on blue-green method, but the traffic is gradually shifted to the new version.

- rolling back a Deployment

```bash
kubectl rollout undo deployment DEPLOYMENT_NAME

kubectl rollout undo deployment DEPLOYMENT_NAME --to-version=2

kubectl rollout undo deployemnt DEPLOYMENT_NAME --revision=2
```

## Managing deployments

Three different states: Progressing State, Complete State, Failed State

- Pausing a Deployment

```bash
kubectl rollout pause deployment DEPLOYMENT_NAME
```

- Resuming a Deployment

```bash
kubectl rollout resume deployment DEPLOYMENT_NAME
```

- Monitoring a Deployment

```bash
kubectl roolout status deployment DEPLOYMENT_NAME
```

- Deleting a Deployment

```bash
kubectl delete deployment DEPLOYMENT_NAME
```

## Jobs and CronJobs

`.yaml file` -> `job object` -> `job controller` -> `pod`

- Parallel Jobs

```yaml
kind: Jon
spec:
  completions: 3
  parallelism: 2
```

- Inspecting a Job

```bash
kubectl describe job JOB_NAME

kubectl get pod -l [job-name=my-app-job]
```

- Scaling a Job

```bash
kubectl scale job JOB_NAME --replicas=VALUE
```

- Failing a Job

```yaml
kind: Job
spec:
  backoffLimit: 4
  activeDeadlineSeconds: 300
```

- Deleting a Job

```bash
kubectl delete -f JOB_NAME

kubectl delete job JOB_NAME

kubectl delete job JOB_NAME --cascade false
```

- CronJob

```yaml
kind: CronJob
spec:
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 3600
  concurrencyPolicy: Forbid
  suspend: True
  successfulJobsHistoryLimit: 3
  failedJobHistoryLimit: 1
  jobtemplate: {}
```

- Create a CronJob

```bash
kubectl apply -f FILE
```

- inspect a CronJob

```bash
kubectl describe cronjob NAME
```

- delete a CronJob

```bash
kubectl delete cronjob NAME
```

## Controlling pod placement

- Affinity and anti-affinity

```yaml
kind: Pod
spec:
  affinity:
    nodeAffinity:
      requiredDeringSchedulingIgnoreDuringExecution:
        nodeSelectorTerm:
          - matchExpressions:
            - key: accelerator-type
            operator: In
            values:
              - gpu
              - tpu
```

- Taints and tolerations

```bash
# taint a to repel Pods
kubectl taint nodes node1 key=Value:NoSchedule key2=Value2:NoSchedule
```

```yaml
toleration:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "Noschedule"
```

## Getting software into your cluster

- How to get software

  1. Build it yourself, and supply your own YAML
  2. Use Helm to install software into your cluster

Create -> Version -> Share -> Publish

- Helm architecture

Helm client (helm) -> helm server (Tiller) -> Kubernetes APIserver (inside Kubernetes cluster)

# **Kubernetes Engin Networking**

Service to expose applications that are running within Pods. LoadBalancers to expose Services to external clients.

## Pod Networking

Each pod has unique IP address. Connect to each other use Pod route networking namespace.

## Services

A Kubernetes Service creates endpoints, label selector to select pods then distribute workload.

- Ways to find a Service in GKE

  1. Environment variables
  2. Kubernetes DNS
  3. lstio

## Service types and Load Balancers

- ClusterIP

ClusterIP Service has a static IP address, Pods use this static IP address to communicate the service.

```yaml
kind: Service
spec:
  type: ClusterIP
  selector:
    app: Backend
```

- NodePort

When creating NodePort Service, ClusterIP service is automatically created in the process. The service can be reached from outside of the cluster using the IP of any Node. traffic will be directed to a service on port 80 and further redirected to one the backend pods on target port.

Can be useful to expose a service through an external laodbalancer that setup and manage yourself. Need to deal with node management, making sure there are no pod collisions.

```yaml
kind: Service
spec:
  type: NodePort
  selector:
    app: Backend
  ports:
    - protocol: TCP
      nodePort: 30100
      port: 80
      targetPort: 9376
```

- LoadBalancer

Loadbalancer forwards the traffic onto the nodes for the service.

```yaml
kind: Service
spec:
  type: LoadBalancer
```

- An engineering tradeoff: Double-hop or imbalance

The network LoadBalancer chooses a random node in the cluster and forwards the traffic to it. then in order to balance the traffic, the iptable will randomly select to pods to handle the incoming traffic. And the pod may be or may not be the same node of the iptable. If the pod not in the same node, moreover the pod need to send response back via iptable node, which is double-hop.

```yaml
kind: Service
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local # avoid double-hop - lowest latency
```

## Ingress Resource

```yaml
apiversion: extensions/v1beta1
kind: Ingress
spec:
  backend:
    serviceName: demo
    serviceport: 80
```

- Updating an Ingress

```bash
kubectl edit ingress INGRESS_NAME

kubectl replace -f FILE
```

## Container-Native Load Balancing

## Network Security

# **Persistent Data and Storgae**

## Volumns

## StatefulSets

## ConfigMaps

## Secrect
