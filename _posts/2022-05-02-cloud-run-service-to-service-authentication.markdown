---
layout: post
title:  "Service to service authentication in Cloud Run with runsd"
date:   2022-05-02 18:45:10 +0800
categories: gcp cloudrun
---

Currently, service discovery in Cloud Run isn't easy -

* You must specify the FQDN (Fully-Qualified Domain Name), which is not deterministic today, resulting in confusion

* To work around the FQDN issues, many try to use a custom domain, which can also be confusing

* You can also use Cloud Load Balancing, but that is an additional component which incurs additional financial and operational cost, and adds an extra hop between service to service calls

Authentication is also not straightforward -
* To restrict access to private services, you can configure the service to require authentication, but that requires [extra code](https://cloud.google.com/run/docs/authenticating/service-to-service#acquire-token) and is additional effort to test locally

* Requiring ingress=internal is also not straightforward. [Ingress controls](https://cloud.google.com/run/docs/securing/ingress) to restrict requests can only come from your own private network, which requires using a [Serverless VPC Access Connector](https://cloud.google.com/vpc/docs/configure-serverless-vpc-access) and configuring all traffic to traverse the VPC, which might result in additional bandwidth costs.  It is also not intuitive and is extra management overhead

Ideally, Cloud Run service `foo` should be able to call Cloud Run service `bar` (in the same region and same project) by using http://bar. Until this funcationality becomes native to Cloud Run, [runsd](https://github.com/ahmetb/runsd) is a nice utility from [Ahmet Balkan](https://www.linkedin.com/in/ahmetalpbalkan/) that helps make service disocvery and service to service authentication easier, similar to what you would expect from a service to service call in k8s.

Using runsd was a very pleasant developer experience, and I documented my steps [here](https://github.com/tzehon/cloud-run-auth-runsd).