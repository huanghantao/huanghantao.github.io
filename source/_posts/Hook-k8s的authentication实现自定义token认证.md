---
title: Hook k8s的authentication实现自定义token认证
date: 2022-03-17 17:13:10
tags:
---

> k8s版本1.20.2

首先，我们需要写一个`webhook`服务：

```golang
package main

import (
    "context"
    "encoding/json"
    "log"
    "net/http"

    "github.com/google/go-github/github"
    "golang.org/x/oauth2"
    authentication "k8s.io/api/authentication/v1beta1"
)

// 处理认证
func handleAuthentication(w http.ResponseWriter, r *http.Request) {
    // 解码认证请求
    var tr authentication.TokenReview
    decoder := json.NewDecoder(r.Body)
    decoder.Decode(&tr)

    // 认证token身份
    user, err := checkToken(tr.Spec.Token)
    if err != nil { // 认证失败了，返回失败的响应给api server
        w.WriteHeader(http.StatusUnauthorized)
        json.NewEncoder(w).Encode(map[string]interface{}{
            "apiVersion": "authentication.k8s.io/v1beta1",
            "kind":       "TokenReview",
            "status": authentication.TokenReviewStatus{
                Authenticated: false,
            },
        })
        return
    }

    // 返回认证成功的响应给api server
    w.WriteHeader(http.StatusOK)
    trs := authentication.TokenReviewStatus{
        Authenticated: true,
        User: authentication.UserInfo{
            Username: user.Username,
            UID:      user.UID,
        },
    }
    json.NewEncoder(w).Encode(map[string]interface{}{
        "apiVersion": "authentication.k8s.io/v1beta1",
        "kind":       "TokenReview",
        "status":     trs,
    })
}

func main() {
    http.HandleFunc("/authentication", func(w http.ResponseWriter, r *http.Request) {
        handleAuthentication(w, r)
    })

    log.Println(http.ListenAndServe(":3000", nil))
}
```

然后创建一个文件，指明我们的这个`webhook server`的地址，文件内容如下：

```json
cat authentication-webhook-config.json
{
    "kind": "Config",
    "apiVersion": "v1",
    "preferences": {},
    "clusters": [
        {
            "name": "custom-authentication",
            "cluster": {
                "server": "http://{{改成你的webhook server的地址}}:3000/authentication"
            }
        }
    ],
    "users": [
        {
            "name": "authentication-apiserver",
            "user": {
                "token": "secret"
            }
        }
    ],
    "contexts": [
        {
            "name": "webhook",
            "context": {
                "cluster": "custom-authentication",
                "user": "authentication-apiserver"
            }
        }
    ],
    "current-context": "webhook"
}
```

接着，修改`/etc/kubernetes/manifests/api-server.yaml`文件，指明我们的`authentication-webhook-config.json`文件路径：

```yaml
- --authentication-token-webhook-config-file=/path/to/authentication-webhook-config.json
```

> 注意，因为api-server是以pod的方式运行的，所以你这个authentication-webhook-config.json文件要挂载到容器里面，挂载方式很基础，就不说了。并且，需要注意的一点就是，如果你在修改`api-server.yaml`文件之前，恰好备份了一个`api-server.yaml`文件放在`/etc/kubernetes/manifests/api-server.yaml.backup`的位置，那么很不幸，你会发现无论你怎么修改`api-server.yaml`，发现`api-server`都没有成功的重启。原因就是`/etc/kubernetes/manifests`这个目录很特殊，具体怎么特殊法，可以看k8s的文档。所以，如果你要备份`api-server.yaml`文件的话，请备份到其他地方，例如`~/api-server.yaml.backup`，**不要备份在`/etc/kubernetes/manifests`目录下面**。

保存`api-server.yaml`文件文件后，`api-server`这个`static pod`会自动重启。验证`webhook`是否配置成功。

先用`ps`查看`api-server`的启动参数，例如：

```bash
ps -ef | grep api-server
```

输出的内容里面，一定要带有你配置的`authentication-token-webhook-config-file`。

然后，使用`kubectl -n kube-system get pods | grep api-server`看`pod`的运行状态是否是`Running`。

最后，再验证请求是否会被转发到你的`webhook server`。测试方法如下：

先配置`.kube/config`，加一段使用`token`认证的配置：

```yaml
vim ~/.kube/config

- name: codinghuang
  user:
    token: 你的token
```

然后请求：

```bash
kubectl get pods --user codinghuang
```

正常情况下，会出现这个用户没有获取`pod`的权限。因为我们当前的`webhook server`仅仅是处理认证，鉴权是没有去`hook`的，`hook`鉴权也很简单，也是需要改`api-server`的配置：

```yaml
- --authorization-mode=Node,RBAC,Webhook
- --runtime-config=authorization.k8s.io/v1beta1=true
- --authorization-webhook-config-file=/etc/kubernetes/config/authorization-webhook-config.json
```

> authorization-webhook-config.json的配置可以抄authentication-webhook-config.json，只需要改一下`server`的`url`，然后加一个处理鉴权的路由，例如/authorization。

如果你不想`hook`鉴权，那么可以直接配置`RoleBinding`，赋予这个用户权限，例如，我给这个用户赋予一个最高的权限：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default-admin
  namespace: default
subjects:
  - kind: User
    name: codinghuang
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

然后这个用户就不会报权限问题了。
