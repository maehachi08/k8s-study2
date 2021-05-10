# 認証のためのkubeconfigの作成

- Controll PlaneとNodeの各コンポーネントの `.kubeconfig` を作成する

## 手順

### kubelet

```
KUBERNETES_PUBLIC_ADDRESS=192.168.10.50

for instance in k8s-master k8s-node1 k8s-node2; do
    kubectl config set-cluster kubernetes \
        --certificate-authority=ca.pem \
        --embed-certs=true \
        --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
        --kubeconfig=${instance}.kubeconfig

    kubectl config set-credentials system:node:${instance} \
        --client-certificate=${instance}.pem \
        --client-key=${instance}-key.pem \
        --embed-certs=true \
        --kubeconfig=${instance}.kubeconfig

    kubectl config set-context default \
        --cluster=kubernetes \
        --user=system:node:${instance} \
        --kubeconfig=${instance}.kubeconfig

    kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

### kube-proxy

```
KUBERNETES_PUBLIC_ADDRESS=192.168.10.50

kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

### kube-controller-manager

```
KUBE_API_SERVER_ADDRESS=192.168.10.50

kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBE_API_SERVER_ADDRESS}:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

### kube-scheduler

```
KUBE_API_SERVER_ADDRESS=192.168.10.50

kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBE_API_SERVER_ADDRESS}:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```


### admin

```
KUBERNETES_PUBLIC_ADDRESS=192.168.10.50

kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes \
    --user=admin \
    --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

## 参考資料

- https://github.com/kelseyhightower/kubernetes/blob/master/docs/05-kubernetes-configuration-files.md
- https://docs.oracle.com/cd/F34086_01/kubernetes-on-oci_jp.pdf
- https://h3poteto.hatenablog.com/entry/2020/08/20/180552
- `kubectl config set-cluster`
    - https://jamesdefabia.github.io/docs/user-guide/kubectl/kubectl_config_set-cluster/
    - https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-set-cluster-em-
- `kubectl config set-credentials`
    - https://jamesdefabia.github.io/docs/user-guide/kubectl/kubectl_config_set-credentials/
    - https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-set-credentials-em-
- `kubectl config set-context`
    - https://jamesdefabia.github.io/docs/user-guide/kubectl/kubectl_config_set-context/
    - https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-set-context-em-

