---
title: Documentation
description: Advanced, in-depth configuration of Keel
type: guide
order: 4
---

While it is extremely easy to set up Keel and just use provided default configuration, sometimes you need additional configuration to better solve your use-case.

## Policies

Use policies to define when you want your application to be updated. Providers can have different mechanisms of getting configuration for your application, but policies are consistent across all of them. Following [semver](http://semver.org/) 
best practices allows you to safely automate application updates.

Available policies:

-  **all**: update whenever there is a version bump
-  **major**: update major & minor & patch versions (same as __all__)
-  **minor**: update only minor & patch versions (ignores major)
-  **patch**: update only patch versions (ignores minor and major versions)
-  **force**: force update even if tag is not semver, ie: `latest`

## Providers

Providers are direct integrations into schedulers or other tools (ie: Helm). Providers are handling events created by triggers. Each provider can handle events in different ways, for example Kubernetes provider identifies impacted deployments and starts rolling update while Helm provider communicates with Tiller, identifies releases by Chart and then starts update. 

Available providers:

- Kubernetes
- Helm

While the goal is often the same, different providers can choose different update strategies.

### Kubernetes

Kubernetes provider was the first, simplest provider added to Keel. Policies and trigger configuration for each application deployment is done through labels. 

Policies are specified through special label:

```
keel.sh/policy=all
```

A policy to update only minor releases:

```
keel.sh/policy=minor
```

### Kubernetes example

Here is an example application `deployment.yaml` where we instruct Keel to update container image whenever there is a new version:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata: 
  name: wd
  namespace: default
  labels: 
      name: "wd"
      keel.sh/policy: all
spec:
  replicas: 1
  template:
    metadata:
      name: wd
      labels:
        app: wd        

    spec:
      containers:                    
        - image: karolisr/webhook-demo:0.0.2
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

If Keel gets an event that `karolisr/webhook-demo:0.0.3` is available - it will update deployment and therefore start a rolling update.

### Polling example

While the deployment above works perfect for both webhook and Google Cloud Pubsub triggers sometimes you can't control these events and the only available solution is to check registry yourself. This is where polling trigger comes to the rescue.

> **Note**: when image with non-semver style tag is supplied (ie: `latest`) Keel will monitor SHA digest. If tag is semver - it will track and notify providers when new versions are available.


Add labels:

```
keel.sh/policy=force # add this to enable updates of non-semver tags
keel.sh/trigger=poll
```

To specify custom polling schedule, check [cron expression format]({{ page.url }}#cron-expression-format)

> **Note** that even if polling trigger is set - webhooks or pubsub events can still trigger updates

Example deployment file for polling:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata: 
  name: wd
  namespace: default
  labels: 
      name: "wd"
      keel.sh/policy: force
      keel.sh/trigger: poll      
  annotations:
      keel.sh/pollSchedule: "@every 10m"
spec:
  replicas: 1
  template:
    metadata:
      name: wd
      labels:
        app: wd        

    spec:
      containers:
        - image: karolisr/webhook-demo:latest # this would start repository digest checks
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

### Helm 


Helm helps you manage Kubernetes applications — Helm Charts helps you define, install, and upgrade even the most complex Kubernetes application. More information can be found on project's website [https://helm.sh/](https://helm.sh/). 

Keel works directly with Tiller (a daemon that is used by Helm CLI) to manage release upgrades when new images are available. 

### Helm example

Keel is configured through your chart's `values.yaml` file.

Here is an example application `values.yaml` file where we instruct Keel to track and update specific values whenever there is a new version:

```yaml
replicaCount: 1
image:
  repository: karolisr/webhook-demo
  tag: "0.0.8"
  pullPolicy: IfNotPresent
service:
  name: webhookdemo
  type: ClusterIP
  externalPort: 8090
  internalPort: 8090

keel:
  # keel policy (all/major/minor/patch/force)
  policy: all
  # images to track and update
  images:
    - repository: image.repository # [1]
      tag: image.tag  # [2]
```

If Keel gets an event that `karolisr/webhook-demo:0.0.9` is available - it will upgrade release values so Helm can start updating your application.

* [1]  resolves during runtime image.repository -> karolisr/webhook-demo
* [2]  resolves during runtime image.tag -> 0.0.8

#### Helm configuration polling example

This example demonstrates Keel configuration for polling.

> **Note** that even if polling trigger is set - webhooks or pubsub events can still trigger updates

```yaml
replicaCount: 1
image:
  repository: karolisr/webhook-demo
  tag: "0.0.8"
  pullPolicy: IfNotPresent
service:
  name: webhookdemo
  type: ClusterIP
  externalPort: 8090
  internalPort: 8090

keel:
  # keel policy (all/major/minor/patch/force)
  policy: all
  # trigger type, defaults to events such as pubsub, webhooks
  trigger: poll
  # polling schedule
  pollSchedule: "@every 2m"
  # images to track and update
  images:
    - repository: image.repository # [1]
      tag: image.tag  # [2]
```


## Triggers

Triggers are entry points into the Keel. Their task is to collect information regarding updated images and send events to providers.

Available triggers:

- Webhooks
  * Native Webhooks
  * DockerHub Webhooks
  * Quay Webhooks
  * Receiving webhooks without public endpoint
- Google Cloud GCR registry
- Polling

### Webhooks

Webhooks are "user-defined HTTP callbacks". They are usually triggered by some event, such as pushing image to a registry.

####  Native Webhooks

Native webhooks (simplified version) are accepted at `/v1/webhooks/native` endpoint with a payload that has __name__ and __tag__ fields: 

```json
{
  "name": "gcr.io/v2-namespace/hello-world", 
  "tag": "1.1.1"
}
```

#### DockerHub Webhooks

DockerHub uses webhooks to inform 3rd party systems about repository related events such as pushed image.

https://docs.docker.com/docker-hub/webhooks - go to your repository on 
`https://hub.docker.com/r/your-namespace/your-repository/~/settings/webhooks/` and point webhooks
to `/v1/webhooks/dockerhub` endpoint. Any number of repositories 
can send events to this endpoint.


#### Quay Webhooks 

Documentation how to setup quay webhooks for __Repository Push__ events is available here: [https://docs.quay.io/guides/notifications.html](https://docs.quay.io/guides/notifications.html). These webhooks should be delivered to `/v1/webhooks/quay` endpoint. Any number of repositories 
can send events to this endpoint.


#### Receiving webhooks without public endpoint

If you don't want to expose your Keel service - recommended solution is [https://webhookrelay.com/](https://webhookrelay.com/) which can deliver webhooks to your internal Keel service through a sidecar container.

Example sidecar container configuration for your `deployments.yaml`:

```yaml
        - image: webhookrelay/webhookrelayd:0.6.0
          name: webhookrelayd          
          env:                         
            - name: KEY
              valueFrom:
                secretKeyRef:
                  name: webhookrelay-credentials
                  key: key                
            - name: SECRET
              valueFrom:
                secretKeyRef:
                  name: webhookrelay-credentials
                  key: secret
            - name: BUCKET
              value: dockerhub      
```

### Google Cloud GCR registry  

If you are using Google Container Engine with Container Registry - search no more, pubsub trigger is for you.

<p class="tip">You will need to create a Google Service Account to use PubSub functionality.</p>

Since Keel requires access for the pubsub in GCE Kubernetes to work - your cluster node pools need to have permissions. If you are creating a new cluster - just enable pubsub from the start. If you have an existing cluster - currently the only way is to create a new node-pool through the gcloud CLI (more info in the [docs](https://cloud.google.com/sdk/gcloud/reference/container/node-pools/create?hl=en_US&_ga=1.2114551.650086469.1487625651)):


#### Create a node pool with enabled pubsub scope

```bash
gcloud container node-pools create new-pool --cluster CLUSTER_NAME --scopes https://www.googleapis.com/auth/pubsub
```

#### Create a service account

Detailed tutorial on creating and configuring service account to access Google services is available here: https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform.

High level steps:

1. Create a service account through Google cloud console with a role: `roles/pubsub.editor` (Keel needs to create topics as well for GCR registries).
2. Furnish a new private key and choose key type as JSON.
3. Import credentials as a secret:
```bash
kubectl create -n <KEEL NAMESPACE> secret generic pubsub-key --from-file=key.json=<PATH-TO-KEY-FILE>.json
```
4. Configure the application with the Secret

#### Update Keel's environment variables

Make sure that in the deployment.yaml you have set environment variables __PUBSUB=1__,  __PROJECT_ID=your-project-id__ and __GOOGLE_APPLICATION_CREDENTIALS__ to your secrets yaml. 


### Polling

Since only the owners of docker registries can control webhooks - it's often convenient to use
polling. Be aware that registries can be rate limited so it's a good practice to set up reasonable polling intervals.
While configuration for each provider can be slightly different - each provider has to accept several polling parameters:

* Explicitly enable polling trigger
* Supply polling schedule (defaults to 1 minute intervals)

#### Cron expression format

Custom polling schedule can be specified as cron format or through predefined schedules (recommended solution). 

Available schedules:


Entry                  | Description                                | Equivalent To
-----                  | -----------                                | -------------
@yearly (or @annually) | Run once a year, midnight, Jan. 1st        | `0 0 0 1 1 *`
@monthly               | Run once a month, midnight, first of month | `0 0 0 1 * *`
@weekly                | Run once a week, midnight on Sunday        | `0 0 0 * * 0`
@daily (or @midnight)  | Run once a day, midnight                   | `0 0 0 * * *`
@hourly                | Run once an hour, beginning of hour        | `0 0 * * * *`


#### Intervals

You may also schedule a job to execute at fixed intervals. This is supported by formatting the cron spec like this:

`@every <duration>`

where _duration_ is a string accepted by [time.ParseDuration](http://golang.org/pkg/time/#ParseDuration).

For example, _@every 1h30m10s_ would indicate a schedule that activates every 1 hour, 30 minutes, 10 seconds.


> **Tip**: If you want to disable polling support for your Keel installation - set environment variable
__POLL=0__.


## Approvals

Users can specify on deployments and Helm charts how many approvals do they have to collect before a resource gets updated. Main features:

* __non-blocking__ - multiple deployments/helm releases can be queued for approvals, the ones without specified approvals will be auto updated.
* __extensible__ - current implementation focuses on Slack but additional approval collection mechanisms are trivial to implement.
* __out of the box Slack integration__ - the only needed Keel configuration is Slack auth token, Keel will start requesting approvals and users will be able to approve.
* __stateful__ - uses [github.com/rusenask/k8s-kv](https://github.com/rusenask/k8s-kv) for persistence so even after updating itself (restarting) it will retain existing info.
* __self cleaning__ - expired approvals will be removed after deadline is exceeded.

Configuration is slightly different for Kubernetes and Helm providers. Details on how to configure your approvals:

### Enabling approvals

Approvals are enabled by default but currently there is only one way to approve/reject updates:
Slack - commands like:

* `keel get approvals` - get all pending/approved/rejected approvals
* `keel approve <identifier>` - approve specified request.
* `keel reject <identifier>` - reject specified request.

Make sure you have set `export SLACK_TOKEN=<your slack token here>` environment variable for Keel deployment.

### Configuring via Kubernetes deployments

The only required configuration for Kubernetes deployment to enable approvals is to add `keel.sh/approvals: "1"` with a number (string! as the underlying type is map[string]string) of required approvals.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata: 
  name: wd
  namespace: default
  labels: 
      name: "wd"
      keel.sh/policy: all
      keel.sh/trigger: poll      
      keel.sh/approvals: "1"
```

### Configuring via Helm charts

To enable approvals for a Helm chart update Keel config section in `values.yaml` with a required number of approvals:

```yaml
replicaCount: 1
image:
  repository: karolisr/webhook-demo
  tag: "0.0.13"
  pullPolicy: IfNotPresent 
service:
  name: webhookdemo
  type: ClusterIP
  externalPort: 8090
  internalPort: 8090

keel:
  # keel policy (all/major/minor/patch/force)
  policy: all
  # trigger type, defaults to events such as pubsub, webhooks
  trigger: poll
  # polling schedule
  pollSchedule: "@every 1m"
  # approvals required to proceed with an update
  approvals: 1
  # images to track and update
  images:
    - repository: image.repository
      tag: image.tag
```

### Approving through Slack example

Keel will send notifications to your Slack group about pending approvals. Approval process is as simple as replying to Keel:

- Approve: `keel approve default/whr:0.4.12`
- Reject it: `keel reject default/whr:0.4.12`

Example conversation:

![Approvals](/images/approvals.png)

### Approving through Hipchat example

Coming soon...

### Managing approvals through HTTP endpoint

For third party integrations it can be useful to approve/reject/delete via HTTP endpoint. You can send an approval request via HTTP endpoint:

**Method**: POST
**Endpoint**: `/v1/approvals`

```json
{
  "identifier": "default/myimage:1.5.5", // <- identifier for the approval request
  "action": "approve", // <- approve/reject/delete, defaults to "approve"
  "voter": "john",  
}
```

### Listing pending approvals through HTTP endpoint

You can also view pending/rejected/approved update request on `/v1/approvals` Keel endpoint (make sure you have service exported). Example response:

**Method**: GET
**Endpoint**: `/v1/approvals`


```json
[
	{
		"provider": "helm",
		"identifier": "default/wd:0.0.15",
		"event": {
			"repository": {
				"host": "",
				"name": "index.docker.io/karolisr/webhook-demo",
				"tag": "0.0.15",
				"digest": ""
			},
			"createdAt": "0001-01-01T00:00:00Z",
			"triggerName": "poll"
		},
		"message": "New image is available for release default/wd (0.0.13 -> 0.0.15).",
		"currentVersion": "0.0.13",
		"newVersion": "0.0.15",
		"votesRequired": 1,
		"deadline": "2017-09-26T09:14:54.979211563+01:00",
		"createdAt": "2017-09-26T09:14:54.980936804+01:00",
		"updatedAt": "2017-09-26T09:14:54.980936824+01:00"
	}
]
```

## Notifications

Keel can send notifications on successful or failed deployment updates.  There are several types of notifications - trusted webhooks or Slack, Hipchat messages.

Notification types:

__Pre-deployment update__ - fired before doing update. Can be used to drain running tasks.

```json
{
	"name":"preparing to update deployment",
	"message":"Preparing to update deployment <your deployment namespace/name> (gcr.io/webhookrelay/webhook-demo:0.0.10)",
	"createdAt":"2017-07-23T23:51:46.478440258+01:00",
	"type":"preparing deployment update",
	"level":"LevelDebug"
}
```

__Successful deployment update__ - fired after successful update.

```json
{
	"name":"deployment update",
	"message":"Successfully updated deployment <your deployment namespace/name> (gcr.io/webhookrelay/webhook-demo:0.0.10)",
	"createdAt":"2017-07-23T23:51:46.478440258+01:00",
	"type":"deployment update",
	"level":"LevelSuccess"
}
```

__Failed deployment update__ - fired after failed event.

```json
{
	"name":"deployment update",
	"message":"Deployment <your deployment namespace/name> (gcr.io/webhookrelay/webhook-demo:0.0.10) update failed, error: <error here> ", 
	"createdAt":"2017-07-23T23:51:46.478440258+01:00",
	"type":"deployment update",
	"level":"LevelError"
}
```

### Webhook notifications

To enabled webhook notifications provide an endpoint via __WEBHOOK_ENDPOINT__ environment variable inside Keel deployment. 

Webhook payload sample:

```json
{
	"name": "update deployment",
	"message": "Successfully updated deployment default/wd (karolisr/webhook-demo:0.0.10)",
	"createdAt": "2017-07-08T10:08:45.226565869+01:00"	
}
```

### Slack notifications

![Slack notifications](/images/slack-notifications.png)

First, get a Slack token, info about that can be found in the [docs](https://get.slack.help/hc/en-us/articles/215770388-Create-and-regenerate-API-tokens). Then, provide token via __SLACK_TOKEN__ environment variable. You should also provide __SLACK_CHANNELS__ environment variable with a comma separated list of channels where these notifications should be delivered to.

Keel will be sending messages when deployment updates succeed or fail.


### Hipchat notifications

Coming soon...


### Mattermost notifications

[Mattermost](https://about.mattermost.com/) is an open source Slack alternative, it's written in Go and React.

If you don't have a Mattermost server, you can set one up by using Docker:

```bash
docker run --name mattermost-preview -d --publish 8065:8065 mattermost/mattermost-preview
```

Server should be reachable on: http://localhost:8065/

Now, enable "incoming webhooks" in your Mattermost server. Documentation can be found [here](https://docs.mattermost.com/developer/webhooks-incoming.html):

![Mattermost webhooks](/images/mattermost-webhooks.png)

Also, you should enable icon and username override so users know that webhooks are coming from Keel:

![Mattermost username and icon override](/images/mattermost-icon-username.png)

Now, set `MATTERMOST_ENDPOINT` environment variable for Keel with your Mattermost webhook endpoint:

![Mattermost configuration](/images/mattermost-configuration.png)

That's it, Keel notifications for Mattermost enabled:

![Mattermost notification](/images/mattermost-notification.png)

> If you want to override bot's username, you can supply `MATTERMOST_USERNAME=somethingelse` environment variable.

### Notification levels

Set notification levels via `NOTIFICATION_LEVEL` environment variable. Available levels: debug, info, success, warn, error, fatal. This setting defaults to `info`.

### Overriding default channels per deployment

Some notification providers such as Slack and Hipchat allow apps to send notifications to multiple channels.

For standard Kubernetes deployments use annotation with channels separated by commas: `keel.sh/notify=channelName1,channelName2`.

![notifications per deployment](/images/notify-per-deployment.png)

For Helm chart please specify `notificationChannels` string list to Keel config section in *values.yaml*:

```yaml
notificationChannels:
  - chan1
  - chan2
```

Your *values.yaml* should look like this:

![notifications per chart](/images/notify-per-chart.png)