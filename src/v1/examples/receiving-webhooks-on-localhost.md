---
title: Receiving webhooks on localhost
description: Receive webhooks on localhost or private networks with Webhook Relay forward command
type: examples
order: 2
---

While developing and testing 3rd party integrations, it is useful to have an ability to receive webhooks on your localhost.

![webhook forward](/images/examples/relay-forward.gif)

Here, the `forward` command takes destination and one optional parameter `--bucket` which to reuse webhooks:

```bash
relay forward --bucket [BUCKET] [DESTINATION]
```

Once agent is running, you can supply provided public URL (format my.webhookrelay.com/v1/webhooks/unique-id) to your 3rd party services.

More details are available on CLI [documentation page](/v1/guide/cli.html#Forward).