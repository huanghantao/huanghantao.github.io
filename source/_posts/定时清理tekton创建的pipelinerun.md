---
title: 定时清理tekton创建的pipelinerun
date: 2022-03-03 15:40:26
tags:
---

`k8s`版本是`1.20.2`，`tekton pipeline`的版本是`0.26.0`。目前它无法自动进行清理，每次手动清理很麻烦，所以需要搞一个定时清理。

参考了[这个人的](https://gist.github.com/raelga/e75e6de4fd04be60f267128e985bde6d)，但是它只能清理`default`命名空间下的，并且会删除掉正在运行中的`pipelinerun`，所以这里优化了下。

需求如下：

每**30分钟**清理一次已经**跑完了**的`pipelinerun`，每条`pipeline`只保留**3个**`pipelinerun`：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: tekton-pipelines
  name: tekton-pipelinerun-cleaner
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tekton-pipelinerun-cleaner-clusterrole
rules:
- apiGroups:
  - "tekton.dev"
  resources:
  - pipelineruns
  - pipelines
  verbs:
  - get
  - list
  - watch
  - delete
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tekton-pipelinerun-cleaner-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tekton-pipelinerun-cleaner-clusterrole
subjects:
- kind: ServiceAccount
  name: tekton-pipelinerun-cleaner
  namespace: tekton-pipelines
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  namespace: tekton-pipelines
  name: tekton-pipelinerun-cleaner
  labels:
    app: tekton-pipelinerun-cleaner
spec:
  schedule: "*/30 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccount: tekton-pipelinerun-cleaner
          containers:
            - name: kubectl
              image: hub.code-galaxy.net/bitnami/kubectl:1.20.15
              env:
                - name: NUM_TO_KEEP
                  value: "3"
              command:
                - /bin/bash
                - -c
                - >
                  while read -r PIPELINE; do
                      while read -r NAMESPACE; do
                          PIPELINERUN_NUM=$(kubectl -n ${NAMESPACE} get pipelinerun -l tekton.dev/pipeline=${PIPELINE} -o name | wc -l)
                          REMOVED_NUM=$(expr ${PIPELINERUN_NUM} - ${NUM_TO_KEEP})
                          if [ ${REMOVED_NUM} -le 0 ]
                          then
                          echo "$(date -Is) NAMESPACE=${NAMESPACE} PIPELINE=${PIPELINE} has ${PIPELINERUN_NUM} pipelineruns, le ${NUM_TO_KEEP}, skip"
                          continue
                          else
                          echo "$(date -Is) NAMESPACE=${NAMESPACE} PIPELINE=${PIPELINE} has ${PIPELINERUN_NUM} pipelineruns, gt ${NUM_TO_KEEP}, delete ${REMOVED_NUM}"
                          fi
                          while read -r PIPELINERUN_TO_REMOVE; do
                              test -n "${PIPELINERUN_TO_REMOVE}" || continue;
                              kubectl -n ${NAMESPACE} delete pipelinerun ${PIPELINERUN_TO_REMOVE} \
                              && echo "$(date -Is) PipelineRun ${PIPELINERUN_TO_REMOVE} deleted." \
                              || echo "$(date -Is) Unable to delete PipelineRun ${PIPELINERUN_TO_REMOVE}.";
                          done < <(kubectl -n ${NAMESPACE} get pipelinerun -l tekton.dev/pipeline=${PIPELINE} --sort-by=.status.startTime -o go-template='{{range .items}}{{if .status.completionTime}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | head -n ${REMOVED_NUM})
                          echo -e "\n"
                      done < <(kubectl get pipeline -A --field-selector=metadata.name=${PIPELINE} -o go-template='{{range .items}}{{.metadata.namespace}}{{"\n"}}{{end}}')
                  done < <(kubectl get pipeline -A -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
              resources:
                requests:
                  cpu: 50m
                  memory: 32Mi
                limits:
                  cpu: 100m
                  memory: 64Mi
```
