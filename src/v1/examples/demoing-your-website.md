---
title: Demoing your website
description: Demo your work-in-progress website from localhost with Webhook Relay tunnels
type: examples
order: 1
---

Once in a while there is a need to share your work-in-progress website, 
built on technologies like [hexo](https://hexo.io/), [jekyll](https://jekyllrb.com/), [hugo](https://gohugo.io/) or any other framework, tool, etc..

Sometimes it is way too early in the development cycle to set up automated (or not) deployment just to get some feedback; this is where tunnels come in:

![simple tunnel](/images/examples/relay-rec-simple.gif)

Here, the `connect` command needs only one argument that specifies destination:

```bash
relay connect [DESTINATION]
```

In the next example we demonstrate tunnels functionality with custom subdomains and automated SSL:

![custom subdomain with ssl](/images/examples/relay-rec-4.gif)

To get a custom subdomain and enable SSL you need to specify two flags: `--subdomain` (or `-s`) and `--crypto`:

```bash
relay connect --subdomain [SUBDOMAIN] --crypto flexible [DESTINATION]
```

Crypto flag `flexible` enables both HTTP and HTTPS endpoints.

More details are available in tunnels [documentation page](/v1/guide/http-tunnels.html).
