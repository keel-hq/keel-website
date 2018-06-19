---
title: Setting up "push to deploy" workflow
description: Tutorial on how to setup a push to deploy workflow for your apps
type: examples
order: 2
---

In this tutorial we will configure several tools to enable automated Kubernetes updates on Git push. This workflow is mostly useful when developing apps for Kubernetes. For production we recommend tag approach where a tagged release would trigger an image build and Keel update policies would increase the version.

![Keel Force Workflow](/images/examples/force-workflow.png)

Once workflow is ready, any push to the master branch (or merge requests from develop/feature branches) will update your app running in Kubernetes.

In this tutorial we will use:

* [Minikube](https://github.com/kubernetes/minikube#what-is-minikube) - our local development Kubernetes environment. Mac users are free to use Docker for Mac with Kubernetes support, works fine!
* GitHub - we will store our code here
* DockerHub - our Docker images will be built and stored here
* Webhook Relay - will relay public webhooks to our internal Kubernetes environment so we don't have to expose Keel to the public internet

## Set up GitHub repository

First, let's set up our versioning control system. Let's create a local repo of our example app and push it to our GitHub repository.

Our example app will be a really really simple one:

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Welcome to my website! Version 0")
	})
	log.Fatal(http.ListenAndServe(":8500", nil))
}
```

## Configure DockerHub (code repository + webhook)

Now, we need to tell DockerHub to build a new image on every GitHub push to the master branch.

## Configure Webhook Relay forwarding

In this step we will configure Webhook Relay to forward DockerHub webhooks our internal Kubernetes environment. This is especially useful when developing on a local Kubernetes cluster as receiving webhooks from public services can be slightly more complicated.

Let's prepare configuration:

```bash
relay forward -b keel-push --no-agent http://localhost:9300 # <- using localhost as webhookrelayd agent will be running as a sidecar
Forwarding configuration created: 
https://my.webhookrelay.com/v1/webhooks/690cbe33-988a-49a4-bf46-5523e2f63061 -> http://localhost:9300  # <- got our public endpoint!
In order to start receiving webhooks - start an agent: 'relay forward'
```

## Deploy services

First, we need to deploy Keel with Webhook Relay sidecar. This is a one-off thing after which when you add more applications to your Kubernetes environment you don't need to repeat this step.

### Keel with Webhook Relay sidecar

```bash
# Create a new authentication key & secret pair:
relay token create
Access key: [KEY] secret: [SECRET]
# use token key and secret to create 'webhookrelay-credentials' Kubernetes secret
kubectl create secret generic webhookrelay-credentials --from-literal=key=[KEY] --from-literal=secret=[SECRET]
```

if RBAC is enabled, use:

```bash
kubectl create -f https://raw.githubusercontent.com/keel-hq/keel/master/deployment/deployment-rbac-whr-sidecar.yaml
```

if there's no RBAC in your cluster, use:

```bash
kubectl create -f https://raw.githubusercontent.com/keel-hq/keel/master/deployment/deployment-norbac-whr-sidecar.yaml
```

## Push to update
