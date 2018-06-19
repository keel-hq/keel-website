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

var version = "v0"

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Welcome to my website! Version %s", version)
	})
	fmt.Printf("App is starting, version: %s \n", version)
	log.Fatal(http.ListenAndServe(":8500", nil))
}
```

Commit your code and push to remote:

![Git push](/images/examples/git-push.png)

## Configure Webhook Relay forwarding

In this step we will configure Webhook Relay to forward DockerHub webhooks our internal Kubernetes environment. This is especially useful when developing on a local Kubernetes cluster as receiving webhooks from public services can be slightly more complicated.

Let's prepare configuration:

```bash
# using localhost as webhookrelayd agent will be running as a sidecar
# webhookrelayd sidecar for keel comes with preconfigured bucket name 'dockerhub'
$ relay forward -b dockerhub --no-agent http://localhost:9300
Forwarding configuration created: 
https://my.webhookrelay.com/v1/webhooks/b968afa1-b737-4385-bc0f-473dbc2007b4 -> http://localhost:9300
In order to start receiving webhooks - start an agent: 'relay forward' 
```

We will need that long URL for our next step when configuring DockerHub webhooks.

## Configure DockerHub (code repository + webhook)

Now, we need to tell DockerHub to build a new image on every GitHub push to the master branch. First, go to https://cloud.docker.com, then `Repositories` and click on `Create` button. Once you have created repository, link it to your GitHub account and click on `Configure Automated Builds`:

![configure automated builds](/images/examples/configure-autobuild.png)

Select your GitHub repository and create a trigger that will:

* React to changes on a `master` branch
* Tag image as `latest`

Ensure that `autobuild` is switched on and click on "Save and Build". You will get your first image prepared.

![configure automated builds](/images/examples/docker-build-config.png)

Also, we will need to setup DockerHub webhooks to Keel via Webhook Relay. For some reason that configuration is not available on https://cloud.docker.com and we have to go to https://hub.docker.com:

![dockerhub webhooks](/images/examples/dockerhub-webhook.png)

## Deploy Keel and your app

First, we need to deploy Keel with Webhook Relay sidecar. This is a one-off thing after which when you add more applications to your Kubernetes environment you don't need to repeat this step.

#### Credentials for webhookrelayd sidecar 

Webhook Relay daemon will need authentication details to connect. We can use `relay` CLI to configure and insert secret into our Kubernetes environment: 

```bash
# we need to create a namespace first for the secret
$ push-workflow-example git:(master) kubectl create namespace keel
namespace "keel" created
# by default Keel is deployed in namespace 'keel', therefore we need secret there
$ push-workflow-example git:(master) relay ingress secret --name webhookrelay-credentials --namespace keel
secret "webhookrelay-credentials" created
```

#### Deploying Keel

now, if your cluster has RBAC enabled, use this template (if you are using Minikube then by default it should be enabled):

```bash
kubectl create -f https://raw.githubusercontent.com/keel-hq/keel/master/deployment/deployment-rbac-whr-sidecar.yaml
```

and if there's no RBAC in your cluster, use:

```bash
kubectl create -f https://raw.githubusercontent.com/keel-hq/keel/master/deployment/deployment-norbac-whr-sidecar.yaml
```

> TIP: Feel free to save deployment manifest locally and add things like Slack or other chat provider notifications, approvals and so on. For the sake of simplicy we are omitting those steps in this tutorial.

So, when we create it Kubernetes should complain a bit about already existing namespace but that's expected:

```bash
$ kubectl create -f https://raw.githubusercontent.com/keel-hq/keel/master/deployment/deployment-rbac-whr-sidecar.yaml
serviceaccount "keel" created
clusterrolebinding.rbac.authorization.k8s.io "keel-clusterrole-binding" created
clusterrole.rbac.authorization.k8s.io "keel-clusterrole" created
deployment.extensions "keel" created
Error from server (AlreadyExists): error when creating "https://raw.githubusercontent.com/keel-hq/keel/master/deployment/deployment-rbac-whr-sidecar.yaml": namespaces "keel" already exists
$ kubectl get pods -n keel
NAME                   READY     STATUS    RESTARTS   AGE
keel-f8b5959cc-jrgcd   2/2       Running   0          8s
```

#### Deploy your app

Now, we need to create a deployment file of our app:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: pushwf  
  labels: 
    name: "pushwf"
    # force policy will ensure that deployment is updated
    # even when tag is unchanged (latest remains)
    keel.sh/policy: force
spec:
  replicas: 1
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: pushwf
  template:
    metadata:
      name: pushwf
      labels:
        app: pushwf
    spec:     
      containers:                    
        - image: keelhq/push-workflow-example:latest
          imagePullPolicy: Always # this is required to force pull image     
          name: pushwf
          ports:
            - containerPort: 8500
          livenessProbe:
            httpGet:
              path: /
              port: 8500
            initialDelaySeconds: 10
            timeoutSeconds: 5
```

Save it as `deployment.yaml` and create it via kubectl:

```bash
kubectl create -f deployment.yaml
```

Check whether it's running:

```bash
$ kubectl get pods
NAME                      READY     STATUS    RESTARTS   AGE
pushwf-8476855f97-nw4st   1/1       Running   0          1m
$ kubectl logs pushwf-8476855f97-nw4st
App is starting, version: v0
```

## Push to update

Now, update your Go program's version string to `v1`:


```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

var version = "v1"

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Welcome to my website! Version %s", version)
	})
	fmt.Printf("App is starting, version: %s \n", version)
	log.Fatal(http.ListenAndServe(":8500", nil))
}
```

Commit and push. In a minute or two (depending on how fast DockerHub can build your image) our app should be updated. Since it's using webhooks, update should be pretty much instantaneous. 

If you visit Webhook Relay `dockerhub` bucket's page, it should show relayed webhook:

![dockerhub webhooks](/images/examples/whr-dockerhub-relayed.png)

Let's check our deployments rollout history:

```bash
$ kubectl rollout history deployment/pushwf
deployments "pushwf"
REVISION  CHANGE-CAUSE
1         <none>
2         keel automated update, version latest -> latest
```

And logs, just to be sure that our application is running the latest code:

```bash
$ kubectl get pods
NAME                      READY     STATUS    RESTARTS   AGE
pushwf-74c574f9cf-l6lq2   1/1       Running   0          4m
$ kubectl logs pushwf-74c574f9cf-l6lq2
App is starting, version: v1 
```

## Conclusion

While setting up Keel and Webhook Relay can take several of your precious minutes away, it saves enormous amount of time later. Not only you get an instant update to your applications based on policies but you also ensure that you won't update wrong cluster or environment by mistake. And, of course, you won't even need to use `kubectl` for your application updates. 

Once Keel is setup in your cluster in can manage many (all) of your applications. When you add your next app to the cluster, just specify the policy and point DockerHub webhook to the same Webhook Relay endpoint. Keel will filter out relevant deployments based on webhook payload and update them.

If you have any questions or find parts of this tutorial incorrect, please raise an issue on Keel's repository [here](https://github.com/keel-hq/keel/issues)