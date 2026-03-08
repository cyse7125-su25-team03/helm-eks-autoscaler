# helm-eks-autoscaler

A Helm chart for deploying the [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/charts/cluster-autoscaler) on Amazon EKS. The Cluster Autoscaler automatically adjusts the size of your EKS node groups based on pod scheduling demand.

---

## Prerequisites

- [kubectl](https://kubernetes.io/docs/tasks/tools/) configured to talk to your EKS cluster
- [Helm](https://helm.sh/docs/intro/install/) v3+
- [Terraform](https://developer.hashicorp.com/terraform/install) v1.0+
- An AWS account with an existing EKS cluster
- EKS node groups tagged for auto-discovery:
  - `k8s.io/cluster-autoscaler/enabled: "true"`
  - `k8s.io/cluster-autoscaler/<CLUSTER_NAME>: "owned"`
- An IAM role for the Cluster Autoscaler with IRSA configured (see [IAM Setup](#iam-setup))
- OIDC provider enabled on your EKS cluster

---

## Repository Structure

```
helm-eks-autoscaler/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── serviceaccount.yaml
│   ├── clusterrole.yaml
│   ├── clusterrolebinding.yaml
│   ├── limitrange.yaml
│   ├── networkpolicy.yaml
│   └── pdb.yaml
└── README.md
```

---

## IAM Setup

The Cluster Autoscaler needs IAM permissions to describe and modify Auto Scaling Groups. This is done via **IRSA (IAM Roles for Service Accounts)** — no hardcoded credentials are required.

### IAM Policy

Attach the following permissions to your IAM role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:DescribeInstanceTypes"
      ],
      "Resource": "*"
    }
  ]
}
```

### Trust Policy

The trust policy must be scoped to the Cluster Autoscaler service account:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:cluster-autoscaler:cluster-autoscaler",
          "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

---

## Build and Deploy

### 1. Add the Helm repository

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update
```

### 2. Create the namespace

```bash
kubectl create namespace cluster-autoscaler
```

### 3. Configure values

Edit `values.yaml` and set the following required fields:

```yaml
autoDiscovery:
  clusterName: "<YOUR_CLUSTER_NAME>"

awsRegion: "<YOUR_AWS_REGION>"

rbac:
  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: "<YOUR_IRSA_ROLE_ARN>"
```

### 4. Install the chart

```bash
helm install cluster-autoscaler cluster-autoscaler/. \
  --namespace cluster-autoscaler \
  --values values.yaml
```

### 5. Verify the deployment

```bash
# Check pod is running
kubectl get pods -n cluster-autoscaler

# Check logs
kubectl logs -n cluster-autoscaler \
  deployment/cluster-autoscaler -f
```

---

## Deploy with Terraform

If you prefer to deploy via Terraform using the `helm_release` resource:

```hcl
resource "helm_release" "cluster_autoscaler" {
  name       = "cluster-autoscaler"
  namespace  = "cluster-autoscaler"
  repository = "https://kubernetes.github.io/autoscaler"
  chart      = "cluster-autoscaler"

  set {
    name  = "autoDiscovery.clusterName"
    value = aws_eks_cluster.main.name
  }

  set {
    name  = "awsRegion"
    value = var.aws_region
  }

  set {
    name  = "rbac.serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
    value = aws_iam_role.cluster_autoscaler.arn
  }
}
```

---

## Upgrade

```bash
helm upgrade cluster-autoscaler cluster-autoscaler/. \
  --namespace cluster-autoscaler \
  --values values.yaml
```

---

## Uninstall

```bash
helm uninstall cluster-autoscaler --namespace cluster-autoscaler
kubectl delete namespace cluster-autoscaler
```

---

## Configuration Reference

| Parameter | Description | Default |
|---|---|---|
| `autoDiscovery.clusterName` | EKS cluster name for ASG discovery | `""` |
| `awsRegion` | AWS region of your EKS cluster | `us-east-1` |
| `replicaCount` | Number of autoscaler replicas | `2` |
| `image.repository` | Container image repository | `<dockerhub-org>/cluster-autoscaler` |
| `image.tag` | Container image tag | `v1.30.0` |
| `resources.requests.cpu` | CPU request | `100m` |
| `resources.requests.memory` | Memory request | `600Mi` |
| `resources.limits.cpu` | CPU limit | `200m` |
| `resources.limits.memory` | Memory limit | `1Gi` |
| `rbac.serviceAccount.annotations` | IRSA role ARN annotation | `{}` |
| `extraArgs.scale-down-delay-after-add` | Delay before scaling down after a scale up | `10m` |
| `extraArgs.scale-down-unneeded-time` | Time a node must be underutilized before removal | `10m` |

---

## Container Image Mirroring

All container images used by this chart are mirrored to our DockerHub organization. To update the mirrored image:

```bash
docker pull registry.k8s.io/autoscaling/cluster-autoscaler:<TAG>
docker tag registry.k8s.io/autoscaling/cluster-autoscaler:<TAG> <dockerhub-org>/cluster-autoscaler:<TAG>
docker push <dockerhub-org>/cluster-autoscaler:<TAG>
```

Update `image.repository` and `image.tag` in `values.yaml` accordingly.

---

## Contributing

All changes must be submitted via Pull Request. Direct pushes to `main` are not allowed.

- Commits must follow [Conventional Commits](https://www.conventionalcommits.org/) specification
- All status checks must pass before merging
- Releases are managed automatically via `semantic-release`