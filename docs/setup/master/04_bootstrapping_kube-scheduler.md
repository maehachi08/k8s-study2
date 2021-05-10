### 手順

1. `Dockerfile_kube-scheduler.armhf` を作成する
   <details><summary>Dockerfile_kube-scheduler.armhf</summary>
      ```
      cat << EOF > Dockerfile_kube-scheduler.armhf
      FROM arm64v8/ubuntu:bionic

      RUN set -ex \
        && apt update \
        && apt install -y wget \
        && apt clean \
        && wget -P /usr/bin/ https://dl.k8s.io/v1.20.1/bin/linux/arm64/kube-scheduler \
        && chmod +x /usr/bin/kube-scheduler \
        && install -o root -g root -m 755 -d /var/lib/kubernetes \
        && install -o root -g root -m 755 -d /etc/kubernetes/config

      COPY kube-scheduler.yaml /etc/kubernetes/config/
      COPY kube-scheduler.kubeconfig /var/lib/kubernetes/

      ENTRYPOINT ["/usr/bin/kube-scheduler"]
      EOF
      ```
   </details>

1. kube-schedulerのconfig生成
   - [k8s 1.19.0](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.19.md#v1190) で `KubeSchedulerConfiguration` が betaにupdateされています
      - https://qiita.com/everpeace/items/7dbf14773db82e765370
         <details><summary>kube-scheduler.yaml</summary>
            ```
            cat << EOF > kube-scheduler.yaml
            ---
            apiVersion: kubescheduler.config.k8s.io/v1beta1
            kind: KubeSchedulerConfiguration
            clientConnection:
              kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
            leaderElection:
              leaderElect: true
            EOF
            ```
         </details>

1. image build
   ```
   sudo buildah bud -t k8s-kube-scheduler --file=Dockerfile_kube-scheduler.armhf ./
   ```

1. pod manifestsを `/etc/kubelet.d` へ作成する
   <details><summary>/etc/kubelet.d/kube-scheduler.yaml</summary>
      ```
      cat << EOF | sudo tee /etc/kubelet.d/kube-scheduler.yaml
      ---
      apiVersion: v1
      kind: Pod
      metadata:
        name: kube-scheduler
        namespace: kube-system
        labels:
          tier: control-plane
          component: kube-scheduler

      spec:
        # https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
        priorityClassName: system-node-critical
        hostNetwork: true
        containers:
          - name: kube-scheduler
            image: localhost/k8s-kube-scheduler:latest
            imagePullPolicy: IfNotPresent
            resources:
              requests:
                cpu: "100m"
                memory: "128Mi"
              limits:
                cpu: "300m"
                memory: "256Mi"
            command:
              - /usr/bin/kube-scheduler
              - --config=/etc/kubernetes/config/kube-scheduler.yaml
              - --v=2
      EOF
      ```
   </details>

1. `crictl` でコンテナ起動を確認する
   ```
   $ sudo crictl ps --name kube-scheduler
   CONTAINER           IMAGE                                                              CREATED             STATE               NAME                ATTEMPT             POD ID
   a19648dec2d54       70e852515b3c74175bb3ad4855287cb81101921b2b1f5a890fa4ebd0eeeee684   15 seconds ago      Running             kube-scheduler      0                   da1d0572bc2b1
   ```

### 参考文献

- https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/

