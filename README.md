\# Kubernetes Project 3 — GPU-Aware Workload Scheduling and Inference Platform on AWS EKS



\## Overview



This project builds a production-style Kubernetes platform for scheduling and operating GPU-oriented workloads on AWS EKS.



The cluster uses a simulated GPU worker node so that GPU scheduling concepts can be tested without requiring an expensive GPU EC2 instance.



The project demonstrates how Kubernetes can:



\- Reserve nodes for specialized GPU workloads

\- Schedule workloads using labels and affinity

\- Protect GPU nodes using taints and tolerations

\- Run batch training workloads using Jobs

\- Run recurring workloads using CronJobs

\- Prioritize critical workloads

\- Preempt low-priority workloads when resources are constrained

\- Manage configuration using ConfigMaps

\- Manage sensitive application data using Secrets

\- Run a highly available inference service

\- Load balance requests across inference replicas

\- Monitor application health using readiness and liveness probes

\- Protect service availability during maintenance

\- Automatically scale inference replicas using HPA

\- Package the entire application using Helm

\- Upgrade and roll back Kubernetes releases



\---



\# Architecture



```text

&#x20;                        AWS EKS Cluster

&#x20;                              |

&#x20;             ---------------------------------

&#x20;             |                               |

&#x20;       General Node                    GPU-designated Node

&#x20;  ip-10-0-150-40                     ip-10-0-131-68

&#x20;             |                               |

&#x20;     Normal workloads                  accelerator=gpu

&#x20;                                     dedicated=gpu:NoSchedule

&#x20;                                             |

&#x20;                           ---------------------------------

&#x20;                           |               |               |

&#x20;                       GPU Jobs       Inference Pods    CronJobs

&#x20;                           |               |

&#x20;                           |        gpu-inference-service

&#x20;                           |               |

&#x20;                           |        -----------------

&#x20;                           |        |               |

&#x20;                           |      Pod A           Pod B

&#x20;                           |        |               |

&#x20;                           |        -------+---------

&#x20;                           |               |

&#x20;                           |              HPA

&#x20;                           |

&#x20;                     PriorityClasses

&#x20;                           |

&#x20;                Kubernetes Preemption

```



\---



\# Environment



The project was built on:



\- AWS EKS

\- Kubernetes `v1.36.3-eks-cb19647`

\- Amazon Linux 2023 worker nodes

\- containerd runtime

\- kubectl

\- Helm

\- Kubernetes Metrics Server



Two EKS worker nodes were used.



The first node was designated as the simulated GPU node:



```text

ip-10-0-131-68.ec2.internal

```



The second node remained available for normal workloads:



```text

ip-10-0-150-40.ec2.internal

```



\---



\# 1. GPU Node Labeling



The first worker node was labeled as GPU-capable:



```bash

kubectl label node ip-10-0-131-68.ec2.internal accelerator=gpu

```



This created:



```text

accelerator=gpu

```



GPU workloads could then explicitly target this node.



\## Concept



A Kubernetes label describes a characteristic of a resource.



For example:



```text

accelerator=gpu

```



means:



> This node is intended for GPU workloads.



\---



\# 2. GPU Scheduling with nodeSelector



A GPU workload used:



```yaml

nodeSelector:

&#x20; accelerator: gpu

```



Kubernetes scheduled the workload onto:



```text

ip-10-0-131-68.ec2.internal

```



while normal workloads remained on the general-purpose node.



This demonstrated basic workload placement.



\---



\# 3. Taints and Tolerations



The GPU node was protected using:



```bash

kubectl taint nodes ip-10-0-131-68.ec2.internal dedicated=gpu:NoSchedule

```



The taint:



```text

dedicated=gpu:NoSchedule

```



prevents normal pods from being scheduled onto the GPU node.



Authorized GPU workloads include:



```yaml

tolerations:

&#x20; - key: "dedicated"

&#x20;   operator: "Equal"

&#x20;   value: "gpu"

&#x20;   effect: "NoSchedule"

```



\## Hotel analogy



A label says:



> This hotel has GPU rooms.



A taint says:



> Regular guests are not allowed into this hotel.



A toleration acts like special authorization allowing the GPU workload to enter.



\---



\# 4. Node Affinity



A GPU workload was also scheduled using Kubernetes Node Affinity.



Example:



```yaml

affinity:

&#x20; nodeAffinity:

&#x20;   requiredDuringSchedulingIgnoredDuringExecution:

&#x20;     nodeSelectorTerms:

&#x20;       - matchExpressions:

&#x20;           - key: accelerator

&#x20;             operator: In

&#x20;             values:

&#x20;               - gpu

```



Node affinity provides more flexible workload placement than a simple `nodeSelector`.



\---



\# 5. Kubernetes Jobs



A simulated machine-learning training workload was created using a Kubernetes `Job`.



The training job:



\- Ran multiple simulated training epochs

\- Was scheduled only on the GPU-designated node

\- Automatically transitioned to `Completed`



Architecture:



```text

Job

&#x20;↓

Pod

&#x20;↓

GPU-designated Node

```



Jobs are useful for finite workloads such as:



\- Model training

\- Batch processing

\- Dataset transformation

\- Evaluation

\- GPU benchmarking



\---



\# 6. CronJobs



A recurring GPU workload was created using:



```text

CronJob

&#x20;  ↓

Job

&#x20;  ↓

Pod

```



The schedule:



```text

\*/5 \* \* \* \*

```



runs every five minutes.



The CronJob simulated recurring model evaluation.



A manual execution was also tested using:



```bash

kubectl create job --from=cronjob/gpu-model-evaluation gpu-model-evaluation-manual -n project3

```



This demonstrated how production CronJobs can be manually tested without changing their normal schedules.



\---



\# 7. PriorityClasses



Two workload priorities were created.



High priority:



```text

gpu-high-priority = 100000

```



Low priority:



```text

gpu-low-priority = -100

```



Higher numeric values represent higher Kubernetes scheduling priority.



The high-priority workload used:



```yaml

priorityClassName: gpu-high-priority

```



\---



\# 8. Real Kubernetes Preemption Test



The GPU node was deliberately placed under resource pressure.



A low-priority workload was scheduled first.



Then a high-priority GPU workload requiring additional CPU resources was submitted.



Kubernetes generated the event:



```text

Preempted pod/low-priority-gpu-workload

```



The lower-priority workload was terminated so that the more important workload could run.



This demonstrated Kubernetes scheduling preemption.



Conceptually:



```text

GPU Node Full

&#x20;     |

&#x20;     v

High-priority workload arrives

&#x20;     |

&#x20;     v

Scheduler searches for lower-priority workload

&#x20;     |

&#x20;     v

Low-priority pod preempted

&#x20;     |

&#x20;     v

Resources released

&#x20;     |

&#x20;     v

High-priority workload scheduled

```



This pattern can be useful in expensive GPU clusters where critical inference or training workloads must take precedence.



\---



\# 9. ConfigMaps



Application configuration was stored outside the container using a ConfigMap.



Example values:



```text

MODEL\_NAME=llama-demo

BATCH\_SIZE=32

GPU\_MODE=training

LOG\_LEVEL=info

```



These values were injected into containers as environment variables.



This avoids hard-coding application configuration into images.



\---



\# 10. Kubernetes Secrets



Sensitive values were stored using Kubernetes Secrets.



Example keys:



```text

API\_KEY

MODEL\_TOKEN

```



The application loaded the secrets using `secretKeyRef`.



The application verified that the secrets were available without printing the secret values into logs.



This demonstrated an important security practice:



> Credentials should never be exposed in application logs.



\---



\# 11. GPU Inference Deployment



A long-running inference workload was created using a Kubernetes Deployment.



The Deployment maintained:



```text

2 replicas

```



Kubernetes automatically managed the desired replica state.



Architecture:



```text

Deployment

&#x20;   ↓

ReplicaSet

&#x20;   ↓

Inference Pods

```



\---



\# 12. Kubernetes Self-Healing



One inference pod was manually deleted.



Kubernetes immediately detected:



```text

Desired replicas = 2

Actual replicas = 1

```



The ReplicaSet automatically created a replacement pod.



This demonstrated Kubernetes self-healing.



\---



\# 13. GPU Inference Service



A Kubernetes `ClusterIP` Service was created:



```text

gpu-inference-service

```



The Service exposed port:



```text

80

```



and forwarded requests to container port:



```text

8000

```



Architecture:



```text

Client Pod

&#x20;   |

&#x20;   v

gpu-inference-service

&#x20;   |

&#x20;   +-------------------+

&#x20;   |                   |

&#x20;   v                   v

Inference Pod A     Inference Pod B

```



\---



\# 14. Service Load Balancing



A temporary curl pod sent repeated requests through the Service.



The response included the hostname of the inference pod.



Repeated requests produced different pod names.



This confirmed that Kubernetes was distributing requests across multiple inference replicas.



The Service discovers pods using:



```yaml

selector:

&#x20; app: gpu-inference

```



\---



\# 15. EndpointSlice



The Service backend was inspected using Kubernetes EndpointSlice.



Example:



```bash

kubectl get endpointslice -n project3

```



EndpointSlice contains the IP addresses of pods that can receive traffic.



Modern Kubernetes uses EndpointSlice as the preferred mechanism instead of the older Endpoints API.



\---



\# 16. Readiness Probe



The inference application exposed:



```text

/ready

```



The Deployment configured:



```yaml

readinessProbe:

&#x20; httpGet:

&#x20;   path: /ready

&#x20;   port: 8000

```



A deliberately broken readiness path was then tested.



The container remained:



```text

STATUS = Running

```



but changed to:



```text

READY = 0/1

```



This demonstrated that Kubernetes distinguishes between:



```text

Application process running

```



and:



```text

Application ready to receive traffic

```



If readiness fails, Kubernetes removes the pod from normal Service traffic.



\---



\# 17. Liveness Probe



The inference application also exposed:



```text

/health

```



A liveness probe monitored this endpoint.



A deliberately invalid endpoint:



```text

/broken-health

```



was configured.



Kubernetes detected repeated failures and automatically restarted the container.



This demonstrated:



```text

Readiness failure

→ Stop sending traffic



Liveness failure

→ Restart container

```



\---



\# 18. PodDisruptionBudget



A PodDisruptionBudget was created:



```yaml

minAvailable: 1

```



The GPU node was drained.



Kubernetes produced:



```text

Cannot evict pod as it would violate the pod's disruption budget

```



This demonstrated how a PDB can preserve application availability during voluntary disruptions such as:



\- Node maintenance

\- Cluster upgrades

\- Node draining



\---



\# 19. Horizontal Pod Autoscaler



An HPA was configured with:



```text

Minimum replicas: 2

Maximum replicas: 5

Target CPU utilization: 50%

```



A load-generator Deployment generated sustained HTTP traffic.



As CPU utilization increased, Kubernetes automatically increased the number of inference replicas.



Observed behavior:



```text

Traffic ↑

&#x20;  ↓

CPU utilization ↑

&#x20;  ↓

Metrics Server

&#x20;  ↓

HPA

&#x20;  ↓

Pod replicas ↑

```



After the load generator was removed, CPU utilization dropped and Kubernetes scaled the application back down.



This demonstrated both:



```text

Scale-out

```



and:



```text

Scale-in

```



\---



\# 20. Helm Packaging



The GPU inference application was packaged into a Helm chart.



The Helm chart contains:



```text

gpu-inference/

│

├── Chart.yaml

├── values.yaml

│

└── templates/

&#x20;   ├── deployment.yaml

&#x20;   ├── service.yaml

&#x20;   ├── configmap.yaml

&#x20;   ├── secret.yaml

&#x20;   ├── hpa.yaml

&#x20;   └── pdb.yaml

```



The chart centralizes configuration inside:



```text

values.yaml

```



Example:



```yaml

replicaCount: 2



autoscaling:

&#x20; enabled: true

&#x20; minReplicas: 2

&#x20; maxReplicas: 5

&#x20; targetCPUUtilizationPercentage: 50

```



\---



\# 21. Helm Validation



The Helm chart was validated using:



```bash

helm template gpu-inference-test .

```



and:



```bash

helm lint .

```



Successful validation returned:



```text

1 chart(s) linted, 0 chart(s) failed

```



\---



\# 22. Helm Installation



The entire inference platform could then be deployed using one command:



```bash

helm install gpu-inference-helm . -n project3

```



or safely installed/upgraded using:



```bash

helm upgrade --install gpu-inference-helm . -n project3

```



Instead of manually applying many YAML files, Helm manages the complete application as one release.



\---



\# 23. Helm Upgrade



Configuration changes were made through:



```text

values.yaml

```



and deployed using:



```bash

helm upgrade gpu-inference-helm . -n project3

```



Helm generated updated Kubernetes manifests and Kubernetes performed the required rollout.



\---



\# 24. Helm Rollback



Helm release history was inspected using:



```bash

helm history gpu-inference-helm -n project3

```



A previous revision could be restored using:



```bash

helm rollback gpu-inference-helm 1 -n project3

```



This demonstrated application release versioning and rollback.



\---



\# 25. Helm Packaging



The final chart was packaged using:



```bash

helm package .

```



producing:



```text

gpu-inference-1.0.0.tgz

```



The packaged application could then be deployed into another compatible Kubernetes environment.



\---



\# Kubernetes Concepts Demonstrated



This project covers:



\- AWS EKS

\- Kubernetes Pods

\- Deployments

\- ReplicaSets

\- Services

\- ClusterIP networking

\- EndpointSlice

\- Jobs

\- CronJobs

\- ConfigMaps

\- Secrets

\- Labels

\- Node selectors

\- Node affinity

\- Taints

\- Tolerations

\- Resource requests

\- Resource limits

\- PriorityClasses

\- Pod preemption

\- Readiness probes

\- Liveness probes

\- PodDisruptionBudget

\- Horizontal Pod Autoscaler

\- Metrics Server

\- Node draining

\- Self-healing

\- Internal load balancing

\- Helm charts

\- Helm values

\- Helm upgrades

\- Helm rollback

\- Helm packaging



\---



\# Hotel Analogy



A hotel analogy helps explain the scheduling architecture.



\## Kubernetes cluster



The entire hotel company.



\## Node



One hotel building.



\## GPU node



A specialized hotel containing expensive GPU rooms.



\## Pod



A guest staying in the hotel.



\## Label



A sign identifying what facilities the hotel contains.



```text

accelerator=gpu

```



means:



> This building contains GPU facilities.



\## Taint



A restriction placed on the hotel:



> Normal guests cannot book rooms here.



\## Toleration



Special authorization allowing an eligible guest into the restricted hotel.



\## Scheduler



The central reservation desk deciding which hotel a guest should stay in.



\## PriorityClass



VIP status.



High-priority guests may receive resources ahead of lower-priority guests.



\## Service



The hotel's main phone number.



Customers call one number without needing to know which individual employee handles their request.



\## HPA



Management automatically adding more front-desk staff when customer demand increases.



\## Readiness probe



Checking:



> Is this employee ready to serve customers?



\## Liveness probe



Checking:



> Is this employee still functioning?



\## PodDisruptionBudget



Management policy saying:



> Never take every front-desk employee offline simultaneously during scheduled maintenance.



\## Helm



A complete hotel operations package containing the configuration required to reproduce the hotel system elsewhere.



\---



\# Key Outcome



The final platform demonstrates a simplified version of how Kubernetes can operate specialized AI infrastructure.



```text

AI workload submitted

&#x20;       ↓

Kubernetes Scheduler

&#x20;       ↓

GPU eligibility checked

&#x20;       ↓

Taints / tolerations checked

&#x20;       ↓

Priority evaluated

&#x20;       ↓

GPU node selected

&#x20;       ↓

Inference workload started

&#x20;       ↓

Health monitored

&#x20;       ↓

Service distributes traffic

&#x20;       ↓

CPU demand monitored

&#x20;       ↓

HPA adjusts replicas

&#x20;       ↓

PDB protects availability

&#x20;       ↓

Helm manages deployment lifecycle

```



Although this project uses a simulated GPU node, the scheduling and orchestration concepts closely mirror the mechanisms used in production GPU Kubernetes environments.



\---



\# Repository Structure



```text

K8s-Project3/

│

├── README.md

│

├── manifests/

│   ├── gpu-pod.yaml

│   ├── gpu-affinity-pod.yaml

│   ├── gpu-job.yaml

│   ├── gpu-cronjob.yaml

│   ├── gpu-configmap.yaml

│   ├── gpu-secret.yaml

│   ├── gpu-inference-deployment.yaml

│   ├── gpu-inference-service.yaml

│   ├── gpu-inference-hpa.yaml

│   └── gpu-inference-pdb.yaml

│

└── helm/

&#x20;   └── gpu-inference/

&#x20;       ├── Chart.yaml

&#x20;       ├── values.yaml

&#x20;       └── templates/

&#x20;           ├── deployment.yaml

&#x20;           ├── service.yaml

&#x20;           ├── configmap.yaml

&#x20;           ├── secret.yaml

&#x20;           ├── hpa.yaml

&#x20;           └── pdb.yaml

```



\---



\# Skills Demonstrated



This project demonstrates hands-on experience with:



\- Kubernetes workload orchestration

\- GPU-aware scheduling concepts

\- EKS cluster operations

\- Distributed workload placement

\- Resource-aware scheduling

\- Kubernetes production resiliency

\- Service discovery

\- Load balancing

\- Autoscaling

\- Health monitoring

\- Workload prioritization

\- Infrastructure packaging

\- Helm release management



\---



\# Project Status



\*\*Completed\*\*



Kubernetes Project 3 successfully demonstrates a GPU-aware inference and workload scheduling platform running on AWS EKS.

