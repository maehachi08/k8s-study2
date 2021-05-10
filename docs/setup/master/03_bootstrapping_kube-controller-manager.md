### 手順

1. `Dockerfile_kube-controller-manager.armhf` を作成する
   <details><summary>Dockerfile_kube-controller-manager.armhf</summary>
      ```
      cat << EOF > Dockerfile_kube-controller-manager.armhf
      FROM arm64v8/ubuntu:bionic

      RUN set -ex \
        && apt update \
        && apt install -y wget \
        && apt clean \
        && wget -P /usr/bin/ https://dl.k8s.io/v1.20.1/bin/linux/arm64/kube-controller-manager \
        && chmod +x /usr/bin/kube-controller-manager \
        && install -o root -g root -m 755 -d /var/lib/kubernetes \
        && install -o root -g root -m 755 -d /etc/kubernetes/config

      COPY ca.pem \
           ca-key.pem \
           service-account-key.pem \
           kube-controller-manager.kubeconfig \
           /var/lib/kubernetes/

      ENTRYPOINT ["/usr/bin/kube-controller-manager"]
      EOF
      ```
   </details>

1. image build
   ```
   sudo buildah bud -t k8s-kube-controller-manager --file=Dockerfile_kube-controller-manager.armhf ./
   ```

1. pod manifestsを `/etc/kubelet.d` へ作成する
   - `--allocate-node-cidrs=true`
      - Node resourceの `spec.podCIDR` へCIDRが設定される
         ```
         kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'
         ```

           - `spec.podCIDR` の値が設定されていないnode instanceではCNI Plugin(flannel)が正常動作しなかった

      <details><summary>/etc/kubelet.d/kube-controller-manager.yaml</summary>
         ```
         cat << EOF | sudo tee /etc/kubelet.d/kube-controller-manager.yaml
         ---
         apiVersion: v1
         kind: Pod
         metadata:
           name: kube-controller-manager
           namespace: kube-system
           labels:
             tier: control-plane
             component: kube-controller-manager

         spec:
           # https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
           priorityClassName: system-node-critical
           hostNetwork: true
           containers:
             - name: kube-controller-manager
               image: localhost/k8s-kube-controller-manager:latest
               imagePullPolicy: IfNotPresent
               resources:
                 requests:
                   cpu: "100m"
                   memory: "128Mi"
                 limits:
                   cpu: "300m"
                   memory: "256Mi"
               command:
                 - /usr/bin/kube-controller-manager
                 - --bind-address=0.0.0.0
                 - --cluster-cidr=10.200.0.0/16
                 - --allocate-node-cidrs=true
                 - --node-cidr-mask-size=24
                 - --cluster-name=kubernetes
                 - --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem
                 - --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem
                 - --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig
                 - --leader-elect=true
                 - --root-ca-file=/var/lib/kubernetes/ca.pem
                 - --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem
                 - --service-cluster-ip-range=10.32.0.0/24
                 - --use-service-account-credentials=true
                 - --v=2
         EOF
         ```
      </details>

1. `crictl` でコンテナ起動を確認する
   ```
   $ sudo crictl ps --name kube-controller-manager
   CONTAINER           IMAGE                                                              CREATED             STATE               NAME                      ATTEMPT             POD ID
   a72cec7323686       4ada5d332b2c795b6333b8b6c538491dec96fb80f81b600359615651725b0ccf   20 seconds ago      Running             kube-controller-manager   0                   526d7f2e9d3cb
   ```

## エラー事例

### Client.Timeoutを超えたため、kube-control-managerとkube-schedulerがロックを取得できない
   - 発生したらコンポーネントを再起動することで回復する
   - kube-apiserverに対する負荷が上がると発生し易くなる
      ```
      E0325 11:08:47.205570       1 leaderelection.go:325] error retrieving resource lock kube-system/kube-controller-manager: Get "https://192.168.10.50:6443/apis/coordination.k8s.io/v1/namespaces/kube-
      system/leases/kube-controller-manager?timeout=10s": context deadline exceeded
      I0325 11:08:47.205695       1 leaderelection.go:278] failed to renew lease kube-system/kube-controller-manager: timed out waiting for the condition
      F0325 11:08:47.205929       1 controllermanager.go:294] leaderelection lost
      ```

## 参考文献

- https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/
- node(flannel)の `Error registering network: failed to acquire lease: node "k8s-node1" pod cidr not assigned` エラーに関して
   - https://blog.net.ist.i.kyoto-u.ac.jp/2019/11/06/kubernetes-%E6%97%A5%E8%A8%98-2019-11-05/
   - https://devops.stackexchange.com/questions/5898/how-to-get-kubernetes-pod-network-cidr

