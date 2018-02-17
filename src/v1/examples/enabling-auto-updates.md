---
title: Enabling automated updates
description: How to automatically update your app
type: examples
order: 1
---

Once in a while (or more often) you need to update your application that's running in a cloud-native fashion inside Kubernetes.

Let's see how easy it is to do it with Keel.

![Keel Quick Start](/images/keel-quick-start.png)

## Install Keel

Installing Keel is the first step, as long as no update policies are defined in your application deployment files or Helm Charts, Keel will ignore them.
You can choose your preferred installation type (kubectl or Helm) to deploy Keel, more details are available [here](/v1/guide/installation.html).

## Specify update policy

Our example app is 'webhook-demo' which pretty much doesn't do anything except registering incoming webhooks and showing them. Deployment file can be found here: https://github.com/webhookrelay/webhook-demo/blob/master/hack/deployment.yml.

While traditional deployment manifest would look like this:

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata: 
  name: wd
  namespace: default
  labels: 
      name: "wd"  
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
```

We need to add Keel policy for updates and optional trigger type.

These settings have to be specified as labels:

```
keel.sh/policy: major
keel.sh/trigger: poll      
```

Here:

- **keel.sh/policy: major** specifies that all - major, minor and patch versions should trigger application update.
- **keel.sh/trigger: poll** informs Keel to create an active watcher for the repository that's specified in the deployment file. For private repositories Keel will use existing secrets that Kubernetes uses to pull the image so no additional configuration required. Polling trigger is useful when webhooks or GCR pubsub is not availabe. If you have PUBSUB enabled for your cluster, then you shouldn't use polling trigger.

Now, our deployment file looks like this:

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


That's it, we update our deployment with new labels, if you already have your app deployed:

```bash
kubectl apply -f your-app-deployment.yaml
```

And then wait for a few minutes till Keel picks up the changes. 