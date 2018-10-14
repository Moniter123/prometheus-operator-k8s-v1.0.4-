
# 使用YAML方式安装

## 1. 下载最新版、解压
* wget https://github.com/coreos/prometheus-operator/archive/v0.23.2.tar.gz


## 2. 部署
  官方提示默认在default命名空间下创建，这里我们修改一下
  Note: make sure to adapt the namespace in the ClusterRoleBinding if deploying in another namespace than the default namespace.

### 2.1 编辑prometheus-operator-0.23.2目录下的bundle.yaml
  注意：上文有三处namespace需要修改

### 2.2 -> 执行创建
  kubectl create -f bundle.yaml 

### 2.3 -> 部署kube-prometheus
  kubectl create -f prometheus-operator-0.23.2/contrib/kube-prometheus/manifests

### 2.4 修改访问方式（集群外部访问）
  把svc的访问方式改为NodePort模式,使用kubectl edit svc [svcname] -n monitoring方式修改 , 
  如： # kubectl edit svc prometheus-k8s -n monitoring

### 查看
  kubectl get pods,svc,deployment,job,daemonset,ingress,configmap,servicemonitor,customresourcedefinitions --namespace=monitoring

-------------------------------------------

# 使用helm方式安装

### 条件: 需要安装helm 和 titler(helm server)
安装好 Helm 后，通过键入如下命令，在 Kubernetes 群集上安装 Tiller：
helm init --upgrade


在缺省配置下， Helm 会利用 "gcr.io/kubernetes-helm/tiller" 镜像在Kubernetes集群上安装配置 Tiller；并且利用 "https://kubernetes-charts.storage.googleapis.com" 作为缺省的 stable repository 的地址。由于在国内可能无法访问 "gcr.io", "storage.googleapis.com" 等域名，阿里云容器服务为此提供了镜像站点。

【重点解决问题】请执行如下命令利用阿里云的镜像来配置 Helm
helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.5.1 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

安装成功完成后，将看到如下输出：
$ helm init --upgrade
$HELM_HOME has been configured at /Users/test/.helm.

Tiller (the helm server side component) has been installed into your Kubernetes Cluster.
Happy Helming!


【重点解决问题】 创建了tiller的 ServceAccount 后还没完，因为我们的 Tiller 之前已经就部署成功了，而且是没有指定 ServiceAccount 的，所以我们需要给 Tiller 打上一个 ServiceAccount 的补丁：
$ kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'


### 部署步骤

git clone https://github.com/coreos/prometheus-operator.git
cd prometheus-operator

1. 创建namespace
kubectl create namespace monitoring

2. 安装 Prometheus Operator Deployment
helm install --name prometheus-operator --set rbacEnable=true --namespace=monitoring helm/prometheus-operator
(注意： 配置都在helm/prometheus-operator目录下)

3. 安装 Prometheus
helm install --name prometheus --set serviceMonitorsSelector.app=prometheus --set ruleSelector.app=prometheus --namespace=monitoring helm/prometheus

4. 安装Alertmanager
helm install --name alertmanager --namespace=monitoring helm/alertmanager

5.安装Grafana
helm install --name grafana --namespace=monitoring helm/grafana

6. 安装 kube-prometheus
helm install --name kube-prometheus --namespace=monitoring helm/kube-prometheus
(kube-prometheus 是一个 Helm Chart，打包了监控 Kubernetes 需要的所有 Exporter 和 ServiceMonitor)

7. 安装Alert 规则
部分文件存在替换配置的操作，暂时忽略；(如需，参考：http://www.sohu.com/a/235023288_766170)
kubectl apply -n monitoring -f contrib/kube-prometheus/manifests/prometheus/prometheus-k8s-rules.yaml

8. 安装 Grafana Dashboard
部分文件存在替换配置的操作，暂时忽略；(如上)
kubectl apply -n monitoring -f contrib/kube-prometheus/manifests/grafana/grafana-dashboards.yaml

9. 其他
helm/webhook
