---
title: Introduction
description: Introduction to Keel, quick start, getting started
type: guide
order: 0
---

## What is Keel?

Keel aims to be a simple, robust, **background** service that automatically updates Kubernetes workloads so users can focus on important things like writing code, testing and admiring their creation.

While [Container Builder](https://cloud.google.com/container-builder/docs/) and [Google Container Engine (Kubernetes)](https://cloud.google.com/container-engine/) make a great pair and building images and running your workloads - there is a missing gap: who/what updates deployments when new images are available? maybe it is you:

1. update image tag in `deployment.yaml`
2. run `kubectl apply -f deployment.yaml`

These updates can be repetitive, lacking control (user needs access to the cluster) and simply not necessary. This is what Keel solves: pluggable trigger system (webhooks, pubsub, polling) and pluggable provider system (Kubernetes, Helm).

## Overview

![Keel Overview](/images/keel-overview.png)

Keel acts as a native Kubernetes service, main features:

- **Runs silently** and doesn’t<sup>**1**</sup> require direct interactions from the user (users label deployments that are eligible for updates)
- **Automatically** creates topic & subscriptions for your GCR images (GCR uses pubsub instead of webhooks to notify regarding push/delete events in registry) so you don’t have to
- ** Accepts webhooks** from DockerHub, Quay, JFrog (because not everyone runs on Google Cloud)
- Schedules regular image **SHA digest** checks if you don’t have access to webhooks (ie: repository is not yours) and scans for new tags

So, once you have deployed Keel in your Kubernetes cluster, the workflow looks like this:

1. Tag a release in GitHub (this triggers CI to build a new image).
2. Cloudbuild/{insert your favorite builder here} starts building an image and pushes to the image registry.
3. Keel gets new image event, looks for impacted deployments marked with keel update policy and starts rolling update.

Easy! Sounds like a PaaS, right?

Minimal Keel configuration where it actively checks for new images:

![minimal configuration](/images/keel-minimal-configuration.png)


<sup>**1**</sup> users can specify that certain deployments have to be approved before proceeding with an update.