---
layout: post
title:  "Configure HPA to autoscale with Istio metrics"
date:   2022-01-15 18:45:10 +0800
categories: istio
---

- [Objectives ](#objectives-)
- [Prepare your environment](#prepare-your-environment)
- [Create a cluster](#create-a-cluster)
- [Install Istio on your cluster](#install-istio-on-your-cluster)
- [Deploy the sample application](#deploy-the-sample-application)
- [Deploy the Custom Metrics Stackdriver Adapter](#deploy-the-custom-metrics-stackdriver-adapter)
- [Configure Autoscaling](#configure-autoscaling)
  - [Observe autoscale event](#observe-autoscale-event)

## Objectives

-  Create a cluster
-  Install Istio on the cluster
-  Configure Istio to send metrics to Cloud Monitoring
-  Autoscale a workload using Istio metrics

## Prepare your environment

In your home directory in Cloud Shell, clone the `microservices-demo` GitHub repository:

```
cd ~
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
```

Set your Cloud project ID and desired zone:

```
export PROJECT_ID=YOUR_PROJECT_ID
export ZONE=YOUR_ZONE
export CLUSTER_NAME=YOUR_CLUSTER_NAME
```

Replace the following:

   -  `YOUR_PROJECT_ID`: the Cloud project ID for the project that you're using in this tutorial
   -  `YOUR_ZONE`: the zone that you're using for your resources in this tutorial
   -  `YOUR_CLUSTER_NAME`: the name of your GKE cluster that you use in this tutorial

Set the cloud project and enable required APIs:

```
gcloud config set project $PROJECT_ID
gcloud services enable container.googleapis.com \
  monitoring.googleapis.com \
  iamcredentials.googleapis.com
```

## Create a cluster

Create a cluster:

```
gcloud container clusters create "${CLUSTER_NAME}" \
--zone "${ZONE}" \
--machine-type "e2-standard-4" \
--logging=SYSTEM,WORKLOAD \
--monitoring=SYSTEM \
--enable-autoscaling \
         	--min-nodes "0" \
         	--max-nodes "5" \
         	--addons HorizontalPodAutoscaling,HttpLoadBalancing
```

## Install Istio on your cluster

Configure `kubectl` to communicate with the cluster:

```
gcloud container clusters get-credentials ${CLUSTER_NAME} \
--zone ${ZONE}
```

Grant cluster administrator permissions to the current user:

```
kubectl create clusterrolebinding cluster-admin-binding \
--clusterrole=cluster-admin \
--user=$(gcloud config get-value core/account)
```

Configure the Helm repository:

```
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

Create a namespace for Istio components:

```
kubectl create namespace istio-system
```

Install the Istio base chart which contains cluster-wide resources used by the Istio control plane:

```
helm install istio-base istio/base -n istio-system
```

Install the Istio discovery chart which deploys the istiod service and configure Istio to send logging and monitoring information to Cloud Operations:

```
helm install istiod istio/istiod -n istio-system --wait \
--set telemetry.v2.stackdriver.enabled=true \
--set telemetry.v2.stackdriver.logging=true \
--set telemetry.v2.stackdriver.monitoring=true \
--set telemetry.v2.stackdriver.topology=true
```

You can check the generated yaml file that is installed by helm by adding the –dry-run flag:

```
helm install istiod istio/istiod -n istio-system --wait \
--set telemetry.v2.stackdriver.enabled=true \
--set telemetry.v2.stackdriver.logging=true \
--set telemetry.v2.stackdriver.monitoring=true \
--set telemetry.v2.stackdriver.topology=true \
--dry-run \
--debug > generated.yaml
```

Enable Istio sidecar proxy injection in the default Kubernetes namespace:

```
kubectl label namespace default istio-injection=enabled
```

## Deploy the sample application

Deploy the sample application on your cluster:

```
cd ~
cd microservices-demo
kubectl apply -n default -f release/kubernetes-manifests.yaml
```

The application includes a load generator which sends requests continuously to other services running within the application.

## Deploy the Custom Metrics Stackdriver Adapter

Deploy the [Custom Metrics Stackdriver Adapter](https://github.com/GoogleCloudPlatform/k8s-stackdriver/tree/master/custom-metrics-stackdriver-adapter). The adapter enables pod autoscaling based on Cloud Monitoring (formerly know as Stackdriver) external metrics:

```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-stackdriver/master/custom-metrics-stackdriver-adapter/deploy/production/adapter_new_resource_model.yaml
```

The adapter lets us read Istio metrics that have been sent to Cloud Monitoring.

## Configure Autoscaling

Deploy the following [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) (HPA):

```
cat  << EOF > hpa.yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: productcatalogservice
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: productcatalogservice
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: External
    external:
      metric:
        name: istio.io|service|server|request_count
        selector:
          matchLabels:
            "metric.labels.destination_canonical_service_name": productcatalogservice
      target:
        type: AverageValue
        averageValue: 4

EOF
kubectl apply -f hpa.yaml

This HPA makes scaling decisions based on Istio metrics in Cloud Monitoring.
```

### Observe autoscale event

If you have properly configured your cluster to work with HPA via Istio metrics in Cloud Monitoring, you should a ‘SuccessfulRescale' event after a few minutes:

```
$ kubectl -n default describe hpa productcatalogservice

Normal   SuccessfulRescale             9s                  horizontal-pod-autoscaler  New size: 4; reason: external metric istio.io|service|server|request_count(&LabelSelector{MatchLabels:map[string]string{metric.labels.destination_workload_name: productcatalogservice,},MatchExpressions:[],}) above target
```

And observe the pods auto-scaling:

```
$ kubectl -n default get pods

productcatalogservice-6c4f68859b-5h8f2   2/2     Running   0          19s
productcatalogservice-6c4f68859b-7vlwv   2/2     Running   0          27d
productcatalogservice-6c4f68859b-f8bgq   2/2     Running   0          19s
productcatalogservice-6c4f68859b-gsvfk   2/2     Running   0          19s
```