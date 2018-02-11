---
title: Installation
description: "CLI, daemon or Kubernetes deployment installation instructions"
type: guide
order: 1
vue_version: 2.5.1
dev_size: "271.12"
min_size: "83.13"
gz_size: "30.33"
ro_gz_size: "21.04"
---

Keel doesn't need a database. Keel doesn't need persistent disk. It gets all required information from your cluster. This makes it truly cloud-native and easy to deploy.

## Deploying with kubectl

Keel can run anywhere and will do its job as long as it can connect to your Kubernetes environment. Since you have (or plan to have) Kubernetes environment, this guide will show you how to deploy Keel inside your Kubernetes cluster.

### Prerequisites

- **Kubernetes environment** (easiest way to get Kubernetes up and running is probably [Google Container Engine](https://cloud.google.com/container-engine/) or [Docker for Mac with Kubernetes](https://docs.docker.com/docker-for-mac/kubernetes/)
- **[kubectl]((https://kubernetes.io/docs/user-guide/kubectl-overview/)**: Kubernetes client

<p class="tip">We assume that your kubectl can access Kubernetes environment. If you have multiple environments, you should use <strong>kubectl config use-context [your cluster]</strong> command.</p>

Configuration sample files are available in Keel repository on GitHub [here](https://github.com/rusenask/keel/tree/master/hack).

### Installing

You can find sample deployments in https://github.com/keel-hq/keel repository under `deployments` directory. You can either clone
whole repository or just download that file. Edit settings (depending on your environment whether you want to use [Google Container Registry](https://cloud.google.com/container-registry/) PUBSUB) or notifications and create it. All configuration is done through environment variables.

Some available configuration options:

* __POLL__ - removing this environment variable will disable poll support.
* __PUBSUB__  and __PROJECT_ID__ - environment variables to enable automated integration with [Google Container Registry](https://cloud.google.com/container-registry/). Set PROJECT_ID to your google cloud project ID. 
* __WEBHOOK_ENDPOINT__ - provide an endpoint where Keel should send notifications.
* __SLACK_TOKEN__ - provide Slack token where Keel should send notifications, more info on getting token in the [docs](https://get.slack.help/hc/en-us/articles/215770388-Create-and-regenerate-API-tokens).
* __HELM_PROVIDER__ - enable [Helm](https://helm.sh/) provider. 


Once you are happy with environment variables, create it:

```bash
kubectl create -f deployment-rbac.yaml
```

That's it, to check whether it successfully started - check pods:

```bash
kubectl -n keel get pods
```

You should see something like this:

```bash
$ kubectl -n keel get pods
NAME                    READY     STATUS    RESTARTS   AGE
keel-2732121452-k7sjc   1/1       Running   0          14s
```

### Uninstalling Keel

To remove Keel from your system, simply use existing deployment file:

```bash
kubectl delete -f deployment-rbac.yaml
```

Or delete the namespace.


## Deploying with Helm

Helm chart is available on [KubeApps](https://kubeapps.com/charts/stable/keel) stable charts and   [Github here](https://github.com/keel-hq/keel/tree/master/chart/keel).


### Installing the Chart with Kubernetes provider support

Docker image _polling_ and _Kubernetes_ provider are set by default, then Kubernetes _deployments_ can be upgraded when new Docker image is available:

```bash
helm repo up
helm upgrade --install keel stable/keel
```

If you want to install the Chart with Helm provider support:

Docker image _polling_ is set by default, but we need to enable _Helm provider_ support, then Helm _releases_ can be upgraded when new Docker image is available:

```bash
helm upgrade --install keel stable/keel --set helmProvider.enabled="true"
```

### Setting up Helm release to be automatically updated by Keel

Add the following to your app's `values.yaml` file and do `helm upgrade ...`:

```yaml
keel:
  # keel policy (all/major/minor/patch/force)
  policy: all
  # trigger type, defaults to events such as pubsub, webhooks
  trigger: poll
  # polling schedule
  pollSchedule: "@every 3m"
  # images to track and update
  images:
    - repository: image.repository # it must be the same names as your app's values
      tag: image.tag # it must be the same names as your app's values
```

The same can be applied with `--set` flag without using `values.yaml` file:

```bash
helm upgrade --install whd webhookdemo --reuse-values \
  --set keel.policy="all",keel.trigger="poll",keel.pollSchedule="@every 3m" \
  --set keel.images[0].repository="image.repository" \
  --set keel.images[0].tag="image.tag"
```

You can read in more details about supported policies, triggers and etc in the [docs](/v1/guide/documentation.html).

Also you should check the [Webhooh demo app](https://github.com/webhookrelay/webhook-demo) and it's chart to have more clear
idea how to set automatic updates.


### Uninstalling the Chart

To uninstall/delete the `keel` deployment:

```bash
helm delete keel
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

### Configuration

The following table lists has the main configurable parameters (polling, triggers, notifications, service) of the _Keel_ chart and they apply to both Kubernetes and Helm providers:

| Parameter                         | Description                            | Default                                                   |
| --------------------------------- | -------------------------------------- | --------------------------------------------------------- |
| polling.enabled                   | Docker registries polling              | `true`                                                    |
| helmProvider.enabled              | Enable/disable Helm provider           | `false`                                                   |
| gcr.enabled                       | Enable/disable GCR Registry            | `false`                                                   |
| gcr.projectID                     | GCP Project ID GCR belongs to          |                                                           |
| gcr.pubsub.enabled                | Enable/disable GCP Pub/Sub trigger     | `false`                                                   |
| webhook.enabled                   | Enable/disable Webhook Notification    | `false`                                                   |
| webhook.endpoint                  | Remote webhook endpoint                |                                                           |
| slack.enabled                     | Enable/disable Slack Notification      | `false`                                                   |
| slack.token                       | Slack token                            |                                                           |
| slack.channel                     | Slack channel                          |                                                           |
| service.enable                    | Enable/disable Keel service            | `false`                                                   |
| service.type                      | Keel service type                      | `LoadBalancer`                                            |
| service.externalPort              | Keel service port                      | `9300`                                                    |
| webhookRelay.enabled              | Enable/disable WebhookRelay integration| `false`                                                   |
| webhookRelay.key                  | WebhookRelay key                       |                                                           |
| webhookRelay.secret               | WebhookRelay secret                    |                                                           |
| webhookRelay.bucket               | WebhookRelay bucket                    |                                                           |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.

Alternatively, a YAML file that specifies the values for the above parameters can be provided while installing the chart. For example,

```bash
helm install --name keel -f values.yaml stable/keel
```
> **Tip**: You can use the default *values.yaml*.
