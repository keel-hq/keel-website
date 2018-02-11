---
title: Quick Start
description: Quick introduction in getting Keel up and running
type: guide
order: 2
---

What's better way to introduce into a system than deploying something?

In this short tutorial we will demonstrate how to run your service with [Github](https://github.com), [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) and Keel. We will deploy a simple application that can be then updated automatically by tagging a new release in Github.

Standard workflow with Keel looks like this:

![Keel Workflow](/images/keel-workflow.png)

## Prerequisites

- Basic knowledge of Kubernetes, containers.
- Google Cloud account, register for a free tier [here](https://cloud.google.com/free/).
- [Provisioned Kubernetes cluster](https://cloud.google.com/kubernetes-engine/).


> You can also use Docker for Mac with Kubernetes support or Minikube for local testing. Feel free to use your own image/application as the only thing that matters is applied label with the update policy.

## Automate your image builds

We would really encourage to use [SemVer](http://semver.org/) for versioning your images. Git is a single source of truth for your software, so when you tag a new release that should be it — no bug fixes/features can go into that version. If you add something then tag minor or patch release.

For an example, you can have a look at these tags https://hub.docker.com/r/keelhq/keel/tags/. Then, if you look at Github releases https://github.com/keel-hq/keel/releases we can see that tags match (except older tags are removed). Easiest way to setup automated builds is with https://cloud.docker.com or [Google Cloud builder](https://cloud.google.com/container-builder/docs/). Docker cloud configuration looks like this:

![Docker cloud config](/images/docker-cloud-config.png)

Our example application will be Keel itself because if a continuous delivery tool cannot upgrade itself then it can’t call itself an automation tool.

## Install Keel

Keel installation is as simple as one deployment file. For our use case, default deployment will be enough:

```bash
kubectl create -f deployment-rbac.yaml
```

You can learn more on installation types [here](/v1/guide/installation.html).

## Specify an update policy for your app

We will be adding two configuration labels for Keel:

```
keel.sh/policy=major # <- update major, minor and patch versions
keel.sh/trigger=poll # <- tells Keel to create an active watcher to the repository which is used in this deployment
```

In this demonstration we will be using example application from here: https://github.com/webhookrelay/webhook-demo and a corresponding Docker image repository [here](https://hub.docker.com/r/karolisr/webhook-demo/tags/). Deployment file can be found [here](https://github.com/webhookrelay/webhook-demo/blob/master/hack/deployment.yml).

Make sure you set one of the old tags:

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata: 
  name: wd
  namespace: default
  labels: 
      name: "wd"
      keel.sh/policy: major
      keel.sh/trigger: poll      
  annotations:
      # keel.sh/pollSchedule: "@every 10m"
spec:
  replicas: 1
  template:
    metadata:
      name: wd
      labels:
        app: wd        
    spec:
      containers:                    
        - image: karolisr/webhook-demo:0.0.8
          imagePullPolicy: Always            
          name: wd
          command: ["/bin/webhook-demo"]
          ports:
            - containerPort: 8090       
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8090
            initialDelaySeconds: 30
            timeoutSeconds: 10
          securityContext:
            privileged: true

```

and create it:

```bash
kubectl create -f deployment.yml
```

Now, sit back and relax, in a minute Keel will update application to the newest version available. Why minute? It’s default polling schedule, you can change it but keep in mind rate limits.

## Wrapping up

In this article we saw how easy it is to get a mini PaaS where on Github release tag using tools like Cloudbuild on docker cloud builder you can get an image with the same version.

We also saw how easy it is to install Keel and mark deployments for automated updates, just specifying few labels. 
