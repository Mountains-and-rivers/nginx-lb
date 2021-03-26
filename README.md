## K8S自建LoadBalancer

```
一般只有云平台支持LoadBalancer，如果脱离云平台，自己搭建的K8S集群，Service的类型使用LoadBalancer是没有任何效果的。为了让私有网络中的K8S集群也能体验到LoadBalabcer，Metallb成为了解决方案。

Metallb运行在K8S集群中，监视集群内LoadBalancer类型的服务，然后从配置的IP池中为其分配一个可用IP，以ARP/NDP或BGP的方式将其广播出去，这个可用IP成为了LoadBalancer的Url，可供集群外访问。
```

## Metallb搭建过程

创建命名空间 metallb-system：

```shell
vim metallb-namespace.yaml
```

写入文件内容：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
```

下载metallb.yaml文件

```shell
wget https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml -O metallb.yaml --no-check-certificate
```

定义LoadBalancer的IP池，先创建configmap



```shell
vim metallb-configMap.yaml
```

写入文件内容：



```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.31.140-192.168.31.199
```

> 注意：IP池的网络需要和K8S集群的IP处于同一网段，我的K8S集群网络是192.168.115.13x，这里IP池则是给到192.168.115.140-192.168.115.199的范围。

执行命令：



```shell
kubectl apply -f metallb-namespace.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
kubectl apply -f metallb.yaml
kubectl apply -f metallb-configMap.yaml
```

## LoadBalancer测试
nginx镜像见: https://github.com/Mountains-and-rivers/nginx_static_test/tree/main/centos-images

```
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-LoadBalancer.yaml
```

查看结果

```
[root@node 234]#  kubectl get svc -n metallb-system
NAME    TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
nginx   LoadBalancer   10.97.201.98   192.168.31.140   9876:31779/TCP   7s
[root@node 234]#  kubectl get pod -n metallb-system
NAME                        READY   STATUS    RESTARTS   AGE
controller-fb659dc8-dvcfg   1/1     Running   0          13m
nginx-68f94fc449-52djz      1/1     Running   0          4m28s
speaker-4k574               1/1     Running   0          13m
speaker-q5k4l               1/1     Running   0          13m
speaker-xb6nh               1/1     Running   0          13m
```

访问验证
![image](https://github.com/Mountains-and-rivers/nginx-lb/blob/main/image/test.png)
