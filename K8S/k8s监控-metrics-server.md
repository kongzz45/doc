# K8S监控-metrics-server

- 安装

```sh
git clone https://github.com/kubernetes-incubator/metrics-server
cd metrics-server
kubectl create -f deploy/1.8+/
```

- 错误1

```sh
[root@node1 1.8+]# kubectl logs metrics-server-b75d497f-bsl4r -n kube-system
I1022 04:20:53.569253       1 serving.go:312] Generated self-signed cert (apiserver.local.config/certificates/apiserver.crt, apiserver.local.config/certificates/apiserver.key)
I1022 04:20:54.377287       1 secure_serving.go:116] Serving securely on [::]:443
E1022 04:21:36.118674       1 reststorage.go:135] unable to fetch node metrics for node "node1": no metrics known for node
E1022 04:21:36.118709       1 reststorage.go:135] unable to fetch node metrics for node "node2": no metrics known for node
E1022 04:21:37.132706       1 reststorage.go:135] unable to fetch node metrics for node "node1": no metrics known for node
E1022 04:21:37.132742       1 reststorage.go:135] unable to fetch node metrics for node "node2": no metrics known for node
```

- 修改

```yaml
[root@node1 ~]# cat metrics-server/deploy/1.8+/metrics-server-deployment.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      volumes:
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.6
        command:
        - /metrics-server
        - metrics-resolution=30s
        - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
        - --kubelet-insecure-tls
        imagePullPolicy: Always
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
```

- 错误2

```sh
[root@node1 metrics-server]# kubectl logs metrics-server-7578984995-fsk7h -n kube-system
I1022 03:42:33.758718       1 serving.go:312] Generated self-signed cert (apiserver.local.config/certificates/apiserver.crt, apiserver.local.config/certificates/apiserver.key)
I1022 03:42:34.715647       1 secure_serving.go:116] Serving securely on [::]:443
E1022 03:43:34.728203       1 manager.go:111] unable to fully collect metrics: [unable to fully scrape metrics from source kubelet_summary:node2: unable to fetch metrics from Kubelet node2 (node2): Get https://node2:10250/stats/summary?only_cpu_and_memory=true: x509: certificate signed by unknown authority, unable to fully scrape metrics from source kubelet_summary:node1: unable to fetch metrics from Kubelet node1 (node1): Get https://node1:10250/stats/summary?only_cpu_and_memory=true: x509: certificate signed by unknown authority]
E1022 03:44:34.732381       1 manager.go:111] unable to fully collect metrics: [unable to fully scrape metrics from source kubelet_summary:node1: unable to fetch metrics from Kubelet node1 (node1): Get https://node1:10250/stats/summary?only_cpu_and_memory=true: x509: certificate signed by unknown authority, unable to fully scrape metrics from source kubelet_summary:node2: unable to fetch metrics from Kubelet node2 (node2): Get https://node2:10250/stats/summary?only_cpu_and_memory=true: x509: certificate signed by unknown authority]
E1022 03:44:58.677063       1 reststorage.go:135] unable to fetch node metrics for node "node2": no metrics known for node
E1022 03:44:58.677109       1 reststorage.go:135] unable to fetch node metrics for node "node1": no metrics known for node
```

- 修改

```sh
[root@node1 ~]# kubectl edit configmap coredns -n kube-system
Edit cancelled, no changes made.
[root@node1 ~]# kubectl describe configmap coredns -n kube-system
Name:         coredns
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
Corefile:
----
.:53 {
    errors
    health
    hosts {
       10.6.176.217 node1
       10.6.176.200 node2
       fallthrough
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}

Events:  <none>
```

- 安装完毕正常情况如下

```sh
[root@node1 ~]# kubectl top nodes
NAME    CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node1   95m          2%     1309Mi          11%
node2   27m          0%     791Mi           6%
[root@node1 ~]# kubectl top pods
NAME                       CPU(cores)   MEMORY(bytes)
my-nginx-f97c96f6d-24rff   0m           1Mi
```

```sh
[root@node1 ~]# kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes | jq .
{
  "kind": "NodeMetricsList",
  "apiVersion": "metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes"
  },
  "items": [
    {
      "metadata": {
        "name": "node1",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/node1",
        "creationTimestamp": "2019-10-22T06:37:42Z"
      },
      "timestamp": "2019-10-22T06:36:45Z",
      "window": "30s",
      "usage": {
        "cpu": "74203270n",
        "memory": "1324144Ki"
      }
    },
    {
      "metadata": {
        "name": "node2",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/node2",
        "creationTimestamp": "2019-10-22T06:37:42Z"
      },
      "timestamp": "2019-10-22T06:36:51Z",
      "window": "30s",
      "usage": {
        "cpu": "25407228n",
        "memory": "793240Ki"
      }
    }
  ]
}
```
