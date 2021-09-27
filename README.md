![Incontro DevOps](https://2021.incontrodevops.it/assets/img/social-1200x630.png)

# Furyctl EKS Demo | IDI2021

This step-by-step tutorial helps you deploy the **Kubernetes Fury Distribution** on an EKS cluster.

This tutorial covers the following steps:

1. Deploy Kubernetes clusters with `furyctl`.
2. Download the latest version of Fury with `furyctl`.
3. Install Fury distribution.
4. Explore some features of the distribution.
5. Teardown the environment.

> ⚠️ **You will be charged** to provision the resources used in this tutorial. You should be charged only a few dollars, but we are not responsible for any charges that may incur.
>
> ❗️ **Remember to stop all the instances by following all the steps listed in the teardown phase.**

## Prerequisites

This tutorial assumes some basic familiarity with Kubernetes. Some experience with Terraform is helpful but not strictly required.

To follow this tutorial, you need:

- **Docker** - a [Docker image]([fury-on-gke-dockerfile]) containing `furyctl` and all the necessary tools is provided.
- **OpenVPN Client** - [Tunnelblick][tunnelblick] (on macOS) or [OpenVPN Connect][openvpn-connect] (for other OS) are recommended.
- **AWS Access Credentials**

### Setup and initialize the environment

1. Open a terminal

2. Run the `fury-getting-started` docker image:

```bash
docker run -ti --rm \
  -v $PWD:/demo \
  registry.sighup.io/delivery/fury-getting-started:0.16
```

3. Clone [this repository][https://github.com/nikever/demo-idi2021-furyctl-eks] containing all the example code used in this tutorial:

```bash
git clone https://github.com/nikever/demo-idi2021-furyctl-eks
```

4. Setup AWS credentials by exporting the following environment variables:

```bash
export AWS_ACCESS_KEY_ID=
export AWS_SECRET_ACCESS_KEY=
export AWS_DEFAULT_REGION=
```

5. Create an S3 Bucket to hold the Terraform state.

```bash
aws s3api create-bucket \
  --bucket fury-idi-2021 \
  --region $AWS_DEFAULT_REGION \
  --create-bucket-configuration LocationConstraint=$AWS_DEFAULT_REGION
```

You are all set ✌️.

## Step 1 - Automatic provisioning of a Kubernetes Clusters

### Boostrap provisioning phase

1. Edit the `bootstrap.yml` template located at `/demo/infrastructure/aws/bootstrap.yml`

2. Execute:

```bash
cd /demo/infrastructure/aws
furyctl bootstrap init
furyctl bootstrap apply
```

3. Inspect the output for VPC ID and Private Subnets ID

> 🚀 The provisioner uses under the hood [this Terraform module](https://github.com/sighupio/fury-eks-installer/tree/master/modules/vpc-and-vpn)

### Cluster provisioning phase

1. Create the `fury-idi-2021-aws.ovpn` OpenVPN credentials file with `furyagent`:

```bash
furyagent configure openvpn-client \
  --client-name fury \
  --config /demo/infrastructure/aws/bootstrap/secrets/furyagent.yml \
  > fury-idi-2021-aws.ovpn
```

2. Check that the `fury` user is now listed:

```bash
furyagent configure openvpn-client \
  --list \
  --config /demo/infrastructure/aws/bootstrap/secrets/furyagent.yml
```

3. Connect to the OpenVPN Server.

4. Edit the `cluster.yml` template located at `/demo/infrastructure/aws/cluster.yml`

5. Execute:

```bash
cd /demo/infrastructure/aws
furyctl cluster init
furyctl cluster apply
```

6. Test the connection:

```bash
export KUBECONFIG=/demo/infrastructure/aws/cluster/secrets/kubeconfig
kubectl get nodes
```

> 🚀 The provisioner uses under the hood [this Terraform module](https://github.com/sighupio/fury-eks-installer/tree/master/modules/eks)

## Step 2 - Download fury modules

1. Download the Fury modules with `furyctl`:

```bash
cd /demo/
furyctl vendor -H
```

2. Inspect the downloaded modules in the `vendor` folder:

```bash
tree -d /demo/vendor -L 3
```

## Step 3 - Installation

```bash
cd /demo/manifests/aws/

make apply
# Due to some chicken-egg 🐓🥚 problem with custom resources you have to apply again
make apply
```

## Step 4 - Explore the distribution

Make sure to be connected with the VPN when you interact with each cluster.

### Setup local DNS

### AWS

1. Get the IP address of the internal load balancer:

```bash
dig $(kubectl get svc ingress-nginx -n ingress-nginx --no-headers | awk '{print $4}')
```

Output:

```bash
...

;; ANSWER SECTION:
xxx.elb.eu-west-1.amazonaws.com. 77 IN A <FIRST_IP>
xxx.elb.eu-west-1.amazonaws.com. 77 IN A <SECOND_IP>
xxx.elb.eu-west-1.amazonaws.com. 77 IN A <THIRD_IP>
...

```

2. Add the following line to your local `/etc/hosts`:

```bash
<FIRST_IP> forecastle.aws.fury cerebro.aws.fury kibana.aws.fury grafana.aws.fury
```

### Play around with the distro

Open the browser and go to <forecastle.aws.fury>

## Step 5 - Teardown

### Destroy clusters

```bash
cd /demo/infrastructure/aws/
furyctl cluster destroy
```

### Destroy loadbalancers and networking resources

1. Delete the target groups and loadbalancer associated with the EKS cluster using AWS CLI:

```bash
loadbalancer=$(aws resourcegroupstaggingapi get-resources  \
               --tag-filters Key=kubernetes.io/cluster/fury-aws-demo,Values=owned \
               | jq -r ".ResourceTagMappingList[] | .ResourceARN" | grep loadbalancer)
for i in $loadbalancer ; do aws elbv2 delete-load-balancer --load-balancer-arn $i ; done

target_groups=$(aws resourcegroupstaggingapi get-resources \
                --tag-filters Key=kubernetes.io/cluster/fury-aws-demo,Values=owned  \
                | jq -r ".ResourceTagMappingList[] | .ResourceARN" | grep targetgroup)
for tg in $target_groups ; do aws elbv2 delete-target-group --target-group-arn $tg ; done
```

2. Delete networking:

```bash
cd /demo/infrastructure/aws/
furyctl bootstrap destroy
```

## Conclusions

Congratulations, you made it! 🥳🥳

We hope you enjoyed this tour of Fury!

### Issues/Feedback

In case your ran into any problems feel free to open a  issue here in GitHub.

### Where to go next?

More on-depth tutorials:

- [Fury on GKE][fury-on-gke]
- [Fury on EKS][fury-on-eks]
- [Fury on Minikube][fury-on-minikube]

More about Fury:

- [Fury Documentation][fury-docs]

[fury-discovery-day-repository]: https://github.com/nikever/demo-fury-discovery-day
[fury-getting-started-repository]: https://github.com/sighupio/fury-getting-started
[fury-getting-started-dockerfile]: https://github.com/sighupio/fury-getting-started/blob/main/utils/docker/Dockerfile
[fury-docs]: https://docs.kubernetesfury.com/

[fury-on-minikube]: https://github.com/sighupio/fury-getting-started/tree/main/fury-on-minikube
[fury-on-eks]: https://github.com/sighupio/fury-getting-started/tree/main/fury-on-eks
[fury-on-gke]: https://github.com/sighupio/fury-getting-started/tree/main/fury-on-gke

[furyagent-repository]: https://github.com/sighupio/furyagent

[provisioner-bootstrap-aws-reference]: https://docs.kubernetesfury.com/docs/cli-reference/furyctl/provisioners/aws-bootstrap/

[tunnelblick]: https://tunnelblick.net/downloads.html
[openvpn-connect]: https://openvpn.net/vpn-client/

[kibana-screenshoot]: ../utils/images/kibana.png
[grafana-screenshoot]: ../utils/images/grafana.png
[cerebro-screenshoot]: ../utils/images/cerebro.png
[forecastle-screenshoot]: ../utils/images/forecastle.png
