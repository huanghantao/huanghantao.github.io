---
title: 如何让k8s中的kaniko缓存项目依赖
date: 2022-01-18 16:34:59
tags:
---

如果我们想要在kaniko缓存`composer vendor`或者`golang mod`等程序依赖，我们需要把这些依赖存起来，一个简单的思路是使用`pvc`来解决。

假设我们使用的`pipeline`组件是`tekton`，那么，我们可以这么写我们的流水线：

```yaml
apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  name: build-push-image
spec:
  workspaces:
  - name: output # 声明workspace
    description: 传递工作区
  params:
  - name: workspace
    description: git代码路径
    type: string
  - name: image
    description: 镜像名字
    type: string
  - name: library-cache
    description: 用于缓存依赖库的pvc
    type: string
  steps:
  - name: build-push-image
    image: hub.code-galaxy.net/library/kaniko-executor:v1.6.0
    imagePullPolicy: IfNotPresent
    args:
    - --dockerfile=/workspace/output/$(params.workspace)/Dockerfile
    - --context=/workspace/output/$(params.workspace)
    - --destination=$(params.image)
    - --insecure-pull # 如果不开启这个，则pull镜像的时候会访问443端口
    - --insecure # 如果不开启这个，则push镜像的时候会访问443端口
    - --skip-tls-verify
    - --log-timestamp=true
    - --push-retry=5
    volumeMounts:
    - name: library-cache
      mountPath: /kaniko/library-cache
  volumes:
  - name: library-cache
    persistentVolumeClaim:
      claimName: $(params.library-cache)
      readOnly: false
```

可以看到，我们把一个`pvc`挂载到了`kaniko`容器的`/kaniko/library-cache`目录里面。那么，我们就可以在`Dockerfile`里面利用这个持久化的目录：

```Dockerfile
FROM hyperf/hyperf:7.4-alpine-v3.11-swoole
LABEL maintainer="Hyperf Developers <group@hyperf.io>" version="1.0" license="MIT" app.name="Hyperf"

WORKDIR /opt/www

RUN rm -rf /root/.composer
RUN ln -s /kaniko/library-cache /root/.composer

COPY . /opt/www
RUN composer install --no-dev -o && php bin/hyperf.php

EXPOSE 9501

ENTRYPOINT ["php", "/opt/www/bin/hyperf.php", "start"]
```

我们只需要把这个持久化的目录软链接到`/root/.composer`目录即可。这样，当`composer install`的时候，下载的依赖就会存在`pvc`里面，并且，下次我们可以接着使用它们。
