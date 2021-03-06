---
title: Transparent Proxy and Filtering on Kubernetes with Initializers
description: Learn how to run a transparent proxy on Kubernetes to filter and intercept traffic out of your deployments using custom deployment initializers.
author: danisla
tags: Kubernetes, mitmproxy, proxy
date_published: 2017-09-23
---

Dan Isla | Google Cloud Solution Architect | Google

This is a follow-on tutorial to the [Transparent Proxy and Filtering on Kubernetes](https://cloud.google.com/community/tutorials/transparent-proxy-and-filtering-on-k8s) tutorial. It shows how to simplify the application of a transparent proxy for existing deployments using a Deployment Initializer. Initializers are one of the [Dynamic Admission Control](https://kubernetes.io/docs/admin/extensible-admission-controllers/) features of Kubernetes, and are available as an alpha feature in Kubernetes 1.7.

This tutorial uses the [tproxy-initializer](https://github.com/danisla/kubernetes-tproxy/tree/master/cmd/tproxy-initializer) Kubernetes Initializer to inject the sidecar InitContainer, ConfigMap and environment variables into a deployment when the annotation `"initializer.kubernetes.io/tproxy": "true"` is present. This tutorial also demonstrates how to deploy the [tproxy Helm chart](https://github.com/danisla/kubernetes-tproxy/tree/master/charts/tproxy) with the optional [Role Based Access Control](https://kubernetes.io/docs/admin/authorization/rbac/) support.

Just like in the previous tutorial, the purpose of the [tproxy-sidecar](https://github.com/danisla/kubernetes-tproxy/tree/master/sidecar) container is to create firewall rules in the pod network to block egress traffic. The [tproxy-podwatch](https://github.com/danisla/kubernetes-tproxy/tree/master/cmd/tproxy-podwatch) controller watches for pod changes containing the annotation and automatically add/removes the local firewall `REDIRECT` rules to apply the transparent proxy to the pod.

![architecture diagram](https://storage.googleapis.com/gcp-community/tutorials/transparent-proxy-and-filtering-on-k8s-with-initializers/tproxy_initializers_diagram.png)

**Figure 1.** transparent proxy with initializers architecture diagram

## Objectives

- Create a Kubernetes cluster with initializer and RBAC support using Google Kubernetes Engine
- Deploy the tproxy, tproxy-initializer and the tproxy-podwatch pods using Helm
- Deploy example apps with annotations to test external access to a Google Cloud Storage bucket

## Before you begin

This tutorial assumes you already have a Google Cloud Platform (GCP) account and are familiar with the high level concepts of Kubernetes Pods and Deployments.

## Costs

This tutorial uses billable components of GCP, including:

- [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/pricing)

Use the [Pricing Calculator](https://cloud.google.com/products/calculator/#id=f52c2651-4b02-4da3-b8cd-fdbca6ad89a9) to estimate the costs for your environment.

## Checkout the source repository

1. Open [Cloud Shell](https://console.cloud.google.com/cloudshell)

2. Clone the repository containing the code for this tutorial:

        git clone https://github.com/danisla/kubernetes-tproxy
        cd kubernetes-tproxy

    The remainder of this tutorial will be run from the root of the cloned repository directory.

## Create Kubernetes Engine cluster and install Helm

1. Create Kubernetes Engine cluster with alpha features enabled, RBAC support, and a cluster version of at least 1.7 to support initializers:

        gcloud container clusters create tproxy-example \
          --zone us-central1-f \
          --enable-kubernetes-alpha

    This command also automatically configures the kubectl command to use the cluster.

2. Create a service account and cluster role binding for Helm to enable RBAC support:

        kubectl create serviceaccount tiller --namespace kube-system

        kubectl create clusterrolebinding tiller-cluster-rule \
          --clusterrole=cluster-admin \
          --serviceaccount=kube-system:tiller

3. Install the Helm tool locally in your Cloud Shell instance:

        curl -sL https://storage.googleapis.com/kubernetes-helm/helm-v2.5.1-linux-amd64.tar.gz | tar -xvf - && sudo mv linux-amd64/helm /usr/local/bin/ && rm -Rf linux-amd64

4. Initialize Helm with the service account:

        helm init --service-account=tiller

    This installs the server side component of Helm, Tiller,  in the Kubernetes cluster. The Tiller pod may take a minute to start, run the command below to verify it has been deployed:

        helm version

    You should see the Client and Server versions in the output:

        Client: &version.Version{SemVer:"v2.5.1", GitCommit:"7cf31e8d9a026287041bae077b09165be247ae66", GitTreeState:"clean"}
        Server: &version.Version{SemVer:"v2.5.1", GitCommit:"7cf31e8d9a026287041bae077b09165be247ae66", GitTreeState:"clean"}

## Install the Helm chart

Before installing the chart, you must first extract the certificates generated by mitmproxy. The generated CA cert is used in the example pods to trust the proxy when making HTTPS requests.

1. Extract the generated certs using Docker:

        cd charts/tproxy
        docker run --rm -v ${PWD}/certs/:/home/mitmproxy/.mitmproxy mitmproxy/mitmproxy >/dev/null 2>&1

2. Install the chart with the initializer and RBAC enabled:

        helm install -n tproxy --set tproxy.useInitializer=true,tproxy.useRBAC=true .

    The output of this command shows you how to augment your deployments to use the initializer. Example output below:

        Add this metadata annotation to your deployment specs to apply the tproxy initializer:

        metadata:
        annotations:
            "initializer.kubernetes.io/tproxy": "true"

3. Get the status of the DaemonSet pods:

        kubectl get pods -o wide

    Notice in the example output below that there is a tproxy pod for each node:

        NAME                     READY     STATUS    RESTARTS   AGE       IP           NODE
        tproxy-tproxy-2h7lk   2/2       Running   0          21s       10.128.0.8   gke-tproxy-example-default-pool-1e70b38d-xchn
        tproxy-tproxy-4mvtf   2/2       Running   0          21s       10.128.0.7   gke-tproxy-example-default-pool-1e70b38d-hk89
        tproxy-tproxy-ljfq9   2/2       Running   0          21s       10.128.0.6   gke-tproxy-example-default-pool-1e70b38d-jsqd

4. A single instance of the initializer pod should be running in the kube-system namespace:

        kubectl get pods -n kube-system --selector=app=tproxy

    Example output:

        NAME                                           READY     STATUS    RESTARTS   AGE
        tproxy-tproxy-initializer-833731286-9kp84   1/1       Running   0          4m

## Deploy example apps

Deploy the sample apps to demonstrate using and not using the annotation to trigger the initializer.

1. Change directories back to the repository root  and deploy the example apps:

        cd ../../
        kubectl create -f examples/debian-app.yaml
        kubectl create -f examples/debian-app-locked.yaml

    Note that the second deployment is the one that contains the deployment annotation described in the chart post-install notes.

2. Get the logs for the pod without the tproxy annotation:

        kubectl logs --selector=app=debian-app,variant=unlocked --tail=10

    Example output:

        https://www.google.com: 200
        https://storage.googleapis.com/solutions-public-assets/: 200
        PING www.google.com (209.85.200.105): 56 data bytes
        64 bytes from 209.85.200.105: icmp_seq=0 ttl=52 time=0.758 ms

    The output from the example app shows the status codes for the requests and the output of a ping command.

    Notice the following:
    - The request to https://www.google.com succeeds with status code 200.
    - The request to the Cloud Storage  bucket succeeds with status code 200.
    - The the ping to www.google.com succeeds.

3. Get the logs for the pod with the tproxy annotation:

        kubectl logs --selector=app=debian-app,variant=locked --tail=4

    Example output:

        https://www.google.com: 418
        https://storage.googleapis.com/solutions-public-assets/: 200
        PING www.google.com (209.85.200.147): 56 data bytes
        ping: sending packet: Operation not permitted

    Notice the following:
    - The proxy blocks the request to https://www.google.com with status code 418.
    - The proxy allows the request to the Cloud Storage bucket with status code 200.
    - The the ping to www.google.com is rejected.

4. Inspect the logs from the mitmproxy DaemonSet pod to show the intercepted requests and responses. Note that the logs have to be retrieved from the tproxy pod that is running on the same node as the example app.

        kubectl logs $(kubectl get pods -o wide | awk '/tproxy.*'$(kubectl get pods --selector=app=debian-app,variant=locked -o=jsonpath={.items..spec.nodeName})'/ {print $1}') -c tproxy-tproxy-mode --tail=10

    Example output:

        10.12.1.41:37380: clientconnect
        10.12.1.41:37380: GET https://www.google.com/ HTTP/2.0
                    << 418 I'm a teapot 30b
        10.12.1.41:37380: clientdisconnect
        10.12.1.41:36496: clientconnect
        Streaming response from 64.233.191.128
        10.12.1.41:36496: GET https://storage.googleapis.com/solutions-public-assets/adtech/dfp_networkimpressions.py HTTP/2.0
                    << 200  (content missing)
        10.12.1.41:36496: clientdisconnect

    Notice that the proxy blocks the request to https://www.google.com with status code 418.

## Cleanup

1. Delete the sample apps:

        kubectl delete -f examples/debian-app.yaml
        kubectl delete -f examples/debian-app-locked.yaml

2. Delete the tproxy helm release:

        helm delete --purge tproxy

3. Delete the Kubernetes Engine cluster:

        gcloud container clusters delete tproxy-example --zone=us-central1-f

## What's next?

- [Transparent Proxy and Filtering on Kubernetes](https://cloud.google.com/community/tutorials/transparent-proxy-and-filtering-on-k8s) - Original tutorial that works without the initializer alpha feature. Also contains some of the additional chart configuration examples.
- [tproxy helm chart](https://github.com/danisla/kubernetes-tproxy/blob/master/charts/tproxy/README.md) - See all configuration options and deployment methods.
- [Istio](https://isio.io/) - A more broad approach to traffic filtering and network policy.
- [Calico Egress NetworkPolicy](https://docs.projectcalico.org/v2.0/getting-started/kubernetes/tutorials/advanced-policy) - Another way to filter egress traffic at the pod level.
