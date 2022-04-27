---
title: 'k8s创建ingress报错: certificate signed by unknown authority'
date: 2022-04-27 23:15:14
tags:
---

报错如下：

```bash
Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": Post "https://ingress-nginx-
controller-admission.ingress-nginx.svc:443/networking/v1/ingresses?timeout= 10s": x509: certificate signed by
unknown authority(111301)
```

找了一圈网上的做法是这样的：

```bash
kubectl delete ValidatingWebhookConfiguration ingress-nginx-admission
```

但是这样仅仅是避开了问题，不是根本原因。

我遇到的一个原因就是：新旧的`nginx ingress`同时存在导致的。

例如，我配置`nginx ingress`的`ConfigMap`的时候，这样写的：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  proxy-body-size: 0
```

这里的`0`写错了，应该是`string`类型才行。

然后，在`apply yaml`的时候，执行到`ConfigMap`自然就会报错了。

当修复了`ConfigMap`之后，再次`apply`，就导致新旧的`nginx ingress`同时存在，最后导致证书出现了问题。

所以，正确的做法是，完全删除`nginx ingress`后，再次`apply`一遍即可。
