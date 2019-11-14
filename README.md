# kube-prometheus

> Note that everything is experimental and may change significantly at any time.

This repository collects Kubernetes manifests, [Grafana](http://grafana.com/) dashboards, and [Prometheus rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/) combined with documentation and scripts to provide easy to operate end-to-end Kubernetes cluster monitoring with [Prometheus](https://prometheus.io/) using the Prometheus Operator.

The content of this project is written in [jsonnet](http://jsonnet.org/). This project could both be described as a package as well as a library.

Components included in this package:

* The [Prometheus Operator](https://github.com/coreos/prometheus-operator)
* 高可用 [Prometheus](https://prometheus.io/)
* 高可用 [Alertmanager](https://github.com/prometheus/alertmanager)
* [Prometheus node-exporter](https://github.com/prometheus/node_exporter)
* [Prometheus Adapter for Kubernetes Metrics APIs](https://github.com/DirectXMan12/k8s-prometheus-adapter)
* [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)
* [Grafana](https://grafana.com/)

This stack is meant for cluster monitoring, so it is pre-configured to collect metrics from all Kubernetes components. In addition to that it delivers a default set of dashboards and alerting rules. Many of the useful dashboards and alerts come from the [kubernetes-mixin project](https://github.com/kubernetes-monitoring/kubernetes-mixin), similar to this project it provides composable jsonnet as a library for users to customize to their needs.

## 源项目
从https://github.com/coreos/kube-prometheus#installing fork的这个项目

修改了如下内容
* 更换prometheus-adapter-deployment.yaml使用的ksonnet版本: ksonnet.beta3-->ksonnet.beta4
* 更换kube-state-metrics-deployment.yaml使用的ksonnet版本: ksonnet.beta3-->ksonnet.beta4
* 更换kube-state-metrics-deployment.yaml使用的docker仓库, 从CoreOS的变成阿里云的
* 更换node-exporter-daemonset.yaml使用的ksonnet版本: ksonnet.beta3-->ksonnet.beta4
* 增加了grafana-service.yaml的服务类型为NodePort, 端口为33000
* 增加了prometheus-service.yaml的服务类型为NodePort, 端口为30090


## 快速开始

> 适用版本是kubernetes v1.16.2, 更早的没测过, 估计也行

> 默认的namespace: monitoring


先执行manifests/setup/里的

```shell
# 先把里面的pod生成成功再弄外面的
kubectl apply -f manifests/setup
```

再执行
```
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
kubectl apply -f manifests/
```

验证是否成功
```cassandraql
[root@k8s-master-1 ~]# kubectl get pods -n monitoring
NAME                                  READY   STATUS             RESTARTS   AGE
alertmanager-main-0                   2/2     Running            0          23h
alertmanager-main-1                   2/2     Running            0          23h
alertmanager-main-2                   2/2     Running            0          23h
grafana-784f5f55df-cskrm              1/1     Running            0          23h
kube-state-metrics-64b545f88-5rzn2    4/4     Running            0          23h
node-exporter-42c2w                   2/2     Running            0          23h
node-exporter-db7ql                   2/2     Running            0          23h
node-exporter-h6wjv                   2/2     Running            0          23h
node-exporter-jpmgp                   2/2     Running            0          23h
node-exporter-k975z                   2/2     Running            0          23h
node-exporter-pv75q                   2/2     Running            0          23h
node-exporter-s9gdr                   2/2     Running            0          23h
prometheus-adapter-6899f8875b-8sjcs   1/1     Running            0          23h
prometheus-k8s-0                      3/3     Running            1          23h
prometheus-k8s-1                      3/3     Running            1          23h
prometheus-operator-99dccdc56-vdq56   1/1     Running            0          23h

```

 * 删除方法:
```shell
kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup
```


### 访问: Grafana
浏览器打开
http://any-kubernetes-node-ip:33000

### 访问prometheus
浏览器打开
http://any-kubernetes-node-ip:30090
