---
layout: post
title:  "Install and access a private GitLab instance on GCP"
date:   2021-006-02 18:45:10 +0800
categories: gcp gitlab
---
# Objectives:

* Create a GCE VM with an internal IP address only
* Install GitLab
* Configure GitLab to have one way access to the Internet
* Access GitLab through IAP

# Steps

## Set your env variables
export PROJECT_ID=MY_PROJECT_ID
export VPC_NAME=MY_VPC_NAME
export SUBNET_NAME=MY_SUBNET_NAME
export SUBNET_RANGE=MY_SUBNET_RANGE
export REGION=MY_REGION

## Create a VPC
```
gcloud compute networks create $VPC_NAME --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional

gcloud compute networks subnets create $SUBNET_NAME --range=$SUBNET_RANGE --network=$VPC_NAME --region=$REGION --enable-private-ip-google-access
```

## Create an internal GCE VM
```
gcloud beta compute instances create gitlab-internal --zone=$ZONE --machine-type=e2-standard-4 \
  --subnet=$SUBNET_NAME --no-address --maintenance-policy=MIGRATE --scopes=cloud-platform \
  --image=ubuntu-2004-focal-v20210510 --image-project=ubuntu-os-cloud --boot-disk-size=10GB \
  --boot-disk-type=pd-balanced --boot-disk-device-name=gitlab-internal --no-shielded-secure-boot \
  --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

## Preparing your project for IAP TCP forwarding
You will use [IAP TCP forwarding](https://cloud.google.com/iap/docs/tcp-forwarding-overview) to enable SSH access to our VM as it does not have a public IP address. IAP TCP forwarding allows you to establish an encrypted tunnel over which you can forward SSH traffic to VM instances.

### Create a firewall rule
You create a firewall rule that allows ingress traffic from the IP range 35.235.240.0/20. This range contains all IP addresses that IAP uses for TCP forwarding.
The rule also allows connections to port 22 where we want SSH to be accessible by using IAP TCP forwarding.

```
gcloud compute firewall-rules create allow-ssh-ingress-from-iap \
  --direction=INGRESS \
  --network=$VPC_NAME \
  --action=allow \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20
```

## SSH into your VM using IAP TCP forwarding
```
gcloud compute ssh gitlab-internal
```

### Verify no outbound connectivity
Test outbound connectivity to the internet. This will fail because your instance has no external IP.
```
ping google.com
```

Enter `Ctrl-c` and type `exit` to exit the SSH session. You will setup [Cloud NAT](https://cloud.google.com/nat/docs/using-nat) to enable outbound internet access.

## Setup Cloud Router
```
gcloud compute routers create $ROUTER_NAME --region=$REGION --network=$VPC_NAME
```

## Setup Cloud NAT
```
gcloud compute routers nats create $NAT_NAME \
    --router=$ROUTER_NAME \
    --region=$REGION \
    --auto-allocate-nat-external-ips \
    --nat-all-subnet-ip-ranges \
    --enable-logging
```

## Create firewall to allow inbound HTTPS access
```
gcloud compute firewall-rules create allow-https \
  --direction=INGRESS \
  --network=$VPC_NAME \
  --action=allow \
  --rules=tcp:443 \
  --source-ranges=0.0.0.0/0
```

## Install GitLab
Now that you have outbound connectivity, you can download and install GitLab.

### Install and configure the necessary dependencies
```
gcloud compute ssh gitlab-internal
sudo apt-get update
sudo apt-get install -y curl openssh-server ca-certificates tzdata perl
```

### Add the GitLab package repository and install the package
Add the GitLab package repository.
```
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
```

### Install the GitLab package
```
export IP_ADDRESS=gcloud compute instances describe gitlab-internal \
                    --zone=$ZONE \
                    --format='get(networkInterfaces[0].networkIP)'

sudo EXTERNAL_URL="https://$IP_ADDRESS" apt-get install gitlab-ee
```

