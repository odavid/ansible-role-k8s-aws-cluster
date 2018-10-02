# k8s-aws-cluster
Ansible role for setting up a k8s cluster and vpc using cloudformation and kops

## Prerequisites
Before using this ansible role, the following commandline tools should be installed:

* [kops](https://github.com/kubernetes/kops)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [helm](https://helm.sh/)

_Python Requirements_
* [boto3](https://github.com/boto/boto3)
* [botocore](https://github.com/boto/botocore)

_For mac users:_
```shell
brew update
brew install kops
brew install kubernetes-cli
brew install helm
```

The role expects the following AWS Resources to be created beforehand:
* S3 Bucket used as kops [state store](https://github.com/kubernetes/kops/blob/master/docs/state.md)
* Route53 hosted zone to be served as cluster [dns-zone](https://github.com/kubernetes/kops/blob/master/docs/aws.md#configure-dns)

## Network Configuration
The role is responsible for creating all vpc network topology as cloudformation template, that manages the AWS VPC network as a [shared VPC](https://github.com/kubernetes/kops/blob/master/docs/run_in_existing_vpc.md). The VPC stack also contains a VPN Gateway that uses the [transit VPC](https://aws.amazon.com/blogs/aws/aws-solution-transit-vpc/) Infrastructure, so all the cluster components can reside in the private subnets including the API Load Balancer.

The cluster must reside in an `odd` number of avilability zones i.e. 3 or 5
Each AZ AZ contain 2 subnets:
* `Private` - a private subnet that will be routed to the `NAT Gateway`
* `Utility` - a public subnet that is connected directly to the the `Internet Gateway`

Masters can be deployed in all subnets in `HA Mode` or on a single Availibity Zone
If specficied a `bastion` will created as well in one of the public subnets enabling SSH into the cluster.
> When using Transit VPC topology, a bastion is not required.

## Kubernetes Addons
The role is installing the following addons:
* [kube2iam](https://github.com/jtblin/kube2iam) - Using IAM Roles for different pods
* [cluster-autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) - configured with [auto-discovery](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/aws#auto-discovery-setup)
* [metrics-server](https://kubernetes.io/docs/tasks/debug-application-cluster/core-metrics-pipeline/) - cluster-wide aggregator of resource usage data
* [external-dns](https://github.com/kubernetes-incubator/external-dns) - Register DNS Records based on service annotations and ingress hostnames
* [kubernetes-dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
* [nginx ingress controllers](https://github.com/kubernetes/ingress-nginx) - Ability to install multiple nginx ingress controllers (internal and internet facing)

## IAM Roles and Policies
The role creates the following IAM policies:
* k8s-cluster-autoscaler - Required IAM Permissions for cluster-autoscaler addon
* k8s-external-dns - Required IAM Permissions for external-dns addon
* k8s-aws-alb-ingress - Required IAM Permissions for aws-alb-ingress-controller (This addon is not active at the moment due to a [bug](https://github.com/kubernetes-sigs/aws-alb-ingress-controller/issues/457))
* k8-cloudwatch-logs - Required IAM Permissions for sending logs to cloudwatch

For each of the above IAM profiles a corresponding IAM Role will be created:
* $POLICY_NAME-$CLUSTER_NAME - An IAM Role for the concrete cluster with the Policy attached to it and with sts:AssumeRole for the Nodes and Masters IAM Role - See [kube2iam](https://github.com/jtblin/kube2iam) for details

## Role Configuration Variables
The ansible role expects the following variables to be available:
* `state_store` - The s3 bucket URI. i.e. `s3://<STATE_STORE_BUCKET>`
* `clusterSpec` - A dict representing the cluster specfication in all aspects: (VPC Networking, Cluster, IAM Roles, Addons)
* `state` - `present|absent` - `default = present` - according to the state the cluster and all the resources will be created/updated or deleted

## Role Tags
The following tags can be control different aspects of ansible role execution:
* `vpc` - responsible for execution of tasks related to vpc stack creation/deletion
* `cluster` - responsible for execution of tasks related to cluster creation/deletion
* `generate-files` - responsible for creation of spec file and cloudformation template without actually running the creation/deletion
* `iam-policies` - responsible for creation of the iam policies
* `iam-roles` - responsible for creation/deletion of cluster iam roles
* `tiller` - responsible for initializing [helm](https://github.com/helm/helm#helm-in-a-handbasket) with a dedicated cluster role
* `addons` - responsible for installing addons using helm
* `ingress-controllers` - responsible for installing nginx ingress controllers within the cluster
* `dashboard` - responsible for installing dashboard within the cluster

## clusterSpec

|         Parameter         |           Description             |         Default                | Required                 |
|---------------------------|-----------------------------------|--------------------------------|--------------------------|
| `cluster_name` | The cluster name | | True |
| `aws_region` | The AWS Region to be used | | True |
| `aws_zones` | Comma Delimited AZ | | True |
| `dns_zone` | The DNS Hosted Zone to be used | | True |
| `sshKeyName` | The SSH Keypair name to be used | | True |
| `sshPublicKey` | The SSH Public key content | | True |
| `attach_transit_vpc` | If True a VPN Gateway will be created and tagged as `transitvpc:spoke=true` | | False |
| `networkCIDR` | The VPC Cidr | | True |
| `subnets` | List of subnet dicts. Each one contains the following attributes: `cidr`, `name`, `type` - `Private|Utility`, `zone` - the Availability zone | | True |
| `sgIngressRules` | List of `Security Group Ingress Rules` to be added to the default security group| |  |
| `instanceGroups` | a dict of `instanceGroups` - specifying nodes attributes| | At least one instance group should be defined other than `masters` |
| `nginxIngressControllers` | a dict of `ingress classes` and their configuration| | usually `nginx-internal` and `nginx-external` |
| `kubernetesDashboard` | enabling `kubernetesDashboard` and its configuration| | |

## Example Configuration

```yaml
state_store: s3://kops-state.example.com
clusterSpec:
  aws_region: eu-west-1
  aws_zones: eu-west-1a,eu-west-1b,eu-west-1c
  dns_zone: k8s.example.com
  cluster_name: k8s.example.com
  sshKeyName: awsAdminKey
  sshPublicKey: "ssh-rsa AA..."

  networkCIDR: '10.150.104.0/21'
  subnets:
  - cidr: 10.150.104.0/24
    name: eu-west-1a
    type: Private
    zone: eu-west-1a
  - cidr: 10.150.105.0/24
    name: eu-west-1b
    type: Private
    zone: eu-west-1b
  - cidr: 10.150.106.0/24
    name: eu-west-1c
    type: Private
    zone: eu-west-1c
  - cidr: 10.150.107.0/24
    name: utility-eu-west-1a
    type: Utility
    zone: eu-west-1a
  - cidr: 10.150.108.0/24
    name: utility-eu-west-1b
    type: Utility
    zone: eu-west-1b
  - cidr: 10.150.109.0/24
    name: utility-eu-west-1c
    type: Utility
    zone: eu-west-1c
  sgIngressRules:
    - cidr: '10.153.8.0/21'

  instanceGroups:
    masters:
      machineType: t2.medium
      rootVolumeSize: 100
    nodes:
      machineType: t2.medium
      maxSize: 4
      minSize: 3
      rootVolumeSize: 50

  nginxIngressControllers:
    nginx-internal:
      internal: true
      certificateArn: 'arn://<ACM-ARN>'
    nginx-external:
      internal: false
      certificateArn: 'arn://<ACM-ARN>'

  kubernetesDashboard:
    enabled: true
    ingress:
      ingressClass: nginx-internal
      hostname: k8s-dashboard.example.com
```

## Example playbook

``` yaml
---
- hosts: localhost
  connection: local
  roles:
    - k8s-aws-cluster

```




