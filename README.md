# WordPress on Amazon EKS with Helm and Horizontal Pod Autoscaling

Deploying a containerized WordPress site and its MySQL database to Amazon EKS using a custom Helm chart, then load testing it to watch Kubernetes scale the application automatically.

## Overview

| | |
|---|---|
| **Cluster** | Amazon EKS, 3 × t3.medium managed nodes, us-east-1, built with eksctl |
| **Packaging** | Custom Helm chart (`my-microservice`) generated with `helm create` and modified |
| **Application** | WordPress 6 (Apache) and MySQL 8.4 |
| **Scaling** | HorizontalPodAutoscaler (autoscaling/v2), 1 to 5 replicas, 50% CPU target |
| **Load testing** | Siege, run as a pod inside the cluster |
| **Tools** | AWS CLI v2, eksctl, kubectl, Helm 4, Git Bash in VS Code |

## Architecture

```
                 Internet
                    │
          AWS Classic Load Balancer  (Service type: LoadBalancer)
                    │
     ┌──────────────┴──────────────┐
     │   WordPress pods (1 to 5)   │ ◄── HPA scales on CPU
     └──────────────┬──────────────┘
                    │  my-microservice-mysql:3306  (ClusterIP, internal only)
             ┌──────┴──────┐
             │ MySQL pod   │ ◄── password from Kubernetes Secret
             └─────────────┘
```

## Repository structure

```
.
├── my-microservice/
│   ├── Chart.yaml
│   ├── values.yaml              # image, service, resources, probes, autoscaling, MySQL settings
│   └── templates/
│       ├── deployment.yaml      # WordPress, with database environment variables added
│       ├── mysql.yaml           # MySQL Deployment and Service (added)
│       ├── service.yaml
│       ├── hpa.yaml
│       └── ...
├── .gitignore
└── .gitattributes
```

## Changes from the original project guide

The guide this project was based on would not have produced a working deployment as written. These are the problems I found and how I fixed them:

| Problem | Fix |
|---|---|
| WordPress was deployed with no database, so it could only return a connection error | Added a MySQL Deployment and ClusterIP Service to the chart and connected WordPress through environment variables |
| The separate `hpa.yaml` targeted a Deployment named `wordpress`, which did not exist | Enabled the chart's built-in HPA template, which builds the target name from the same helper as the Deployment |
| kubectl was never installed | Verified kubectl was available before creating the cluster |
| Default 1-second probe timeouts would restart pods under load | Raised probe timeouts and the liveness failure threshold, and added a startup delay |
| `image: wordpress:latest` | Pinned to `wordpress:6-apache` so every pod runs the same major version |
| `persistence` values were not read by any template | Removed them and documented storage as a production improvement |
| Siege command could pass its arguments to the image's entrypoint | Used `kubectl run --command` and the Service's internal DNS name |
| Helm install script targets Linux and macOS | Installed Helm on Windows with winget |

## Deploying

**Prerequisites:** AWS CLI v2 configured with an IAM user, eksctl, kubectl, and Helm.

**1. Create the cluster** (about 15 minutes)

```bash
eksctl create cluster --name enterprise-cluster --region us-east-1 --nodegroup-name standard-workers --node-type t3.medium --nodes 3 --nodes-min 3 --nodes-max 6 --managed
```

**2. Create the database Secret.** Passwords are generated randomly and never stored in this repository.

```bash
kubectl create secret generic wordpress-db --from-literal=mysql-password="$(openssl rand -hex 16)" --from-literal=mysql-root-password="$(openssl rand -hex 16)"
```

**3. Install the chart**

```bash
helm install my-microservice ./my-microservice
kubectl get pods -w
kubectl get svc my-microservice
```

Open the `EXTERNAL-IP` hostname in a browser over `http://` to complete the WordPress setup.

## Load testing and results

```bash
kubectl get hpa my-microservice -w
kubectl run siege --image=jstarcher/siege --restart=Never --command -- siege -c 25 -t 5m http://my-microservice
```

| HPA reading | Replicas | Event |
|---|---|---|
| `cpu: 5%/50%` | 1 | Idle |
| `cpu: 166%/50%` | 3 | Single pod at its CPU limit (500m limit ÷ 300m request) |
| `cpu: 135%/50%` | 5 | Scaled to the configured maximum |
| `cpu: 4%/50%` | 5 → 1 | Load stopped; scaled down after the 5-minute stabilization window |

New pods reached Ready in about 3 seconds. All sampled requests returned HTTP 200, and no pods restarted during the test. At 5 replicas CPU remained near 130%, so the HPA was capped by `maxReplicas` rather than by demand.

## Cleanup

```bash
helm uninstall my-microservice        # removes the load balancer
kubectl delete secret wordpress-db
eksctl delete cluster --name enterprise-cluster --region us-east-1
```

Confirm in CloudFormation that both `eksctl-enterprise-cluster-*` stacks are deleted.

## What I would change for production

- **Persistent storage:** MySQL uses `emptyDir`, so data is lost if the pod is rescheduled. Production would use a PersistentVolumeClaim with the EBS CSI driver, or Amazon RDS.
- **Shared WordPress files:** uploaded media is stored per pod. A shared EFS volume or S3 offload would keep every replica consistent.
- **Consistent WordPress security keys:** each pod generates its own authentication keys, which can break admin logins across replicas. These would be supplied from a Secret.
- **Node autoscaling:** the HPA scales pods but not nodes. Cluster Autoscaler or Karpenter would add nodes when pods cannot be scheduled.
- **HTTPS:** an AWS Load Balancer Controller with an ACM certificate in place of the plain HTTP Classic Load Balancer.
- **Secrets management:** AWS Secrets Manager through the Secrets Store CSI driver in place of manually created Kubernetes Secrets.

## Author

Joe Elmore