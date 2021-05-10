# bootstrapping kube-apiserver

## 手順

1. `Dockerfile_kube-apiserver.armhf` を作成する
   <details><summary>Dockerfile_kube-apiserver.armhf</summary>
      ```
      cat << EOF > Dockerfile_kube-apiserver.armhf
      FROM arm64v8/ubuntu:bionic

      RUN set -ex \
        && apt update \
        && apt install -y wget \
        && apt clean \
        && wget --quiet -P /usr/bin/ https://dl.k8s.io/v1.20.1/bin/linux/arm64/kube-apiserver \
        && chmod +x /usr/bin/kube-apiserver \
        && install -o root -g root -m 755 -d /var/lib/kubernetes \
        && install -o root -g root -m 755 -d /etc/kubernetes/config \
        && install -o root -g root -m 755 -d /etc/kubernetes/webhook

      COPY ca.pem \
           ca-key.pem \
           kubernetes-key.pem \
           kubernetes.pem \
           service-account-key.pem \
           service-account.pem \
           encryption-config.yaml \
           /var/lib/kubernetes/

      COPY authorization-config.yaml /etc/kubernetes/webhook/

      EXPOSE 6443

      ENTRYPOINT ["/usr/bin/kube-apiserver"]
      EOF
      ```
   </details>

1. encryption-provider-config を作成する
   - `--encryption-provider-config` オプションで指定してsecretリソースを作成する際に暗号化するための鍵を定義する
      - https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#encrypting-your-data
      - https://access.redhat.com/documentation/ja-jp/openshift_container_platform/3.11/html/cluster_administration/admin-guide-encrypting-data-at-datastore
         <details><summary>encryption-config.yaml</summary>
            ```
            ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

            cat << EOF > encryption-config.yaml
            kind: EncryptionConfig
            apiVersion: v1
            resources:
              - resources:
                  - secrets
                providers:
                  - aescbc:
                      keys:
                        - name: key1
                          secret: ${ENCRYPTION_KEY}
                  - identity: {}
            EOF
            ```
         </details>

1. webbhook configファイルを作成する
   - `--authorization-webhook-config-file` で指定するファイル
      <details><summary>authorization-config.yaml</summary>
         ```
         KUBE_API_SERVER_ADDRESS=192.168.10.50

         cat << EOF > authorization-config.yaml
         ---
         apiVersion: v1
         # kind of the API object
         kind: Config
         # clusters refers to the remote service.
         clusters:
           - name: kubernetes
             cluster:
               certificate-authority: /var/lib/kubernetes/ca.pem       # CA for verifying the remote service.
               server: https://${KUBE_API_SERVER_ADDRESS}:6443/authenticate # URL of remote service to query. Must use 'https'.

         # users refers to the API server's webhook configuration.
         users:
           - name: api-server-webhook
             user:
               client-certificate: /var/lib/kubernetes/kubernetes.pem  # cert for the webhook plugin to use
               client-key: /var/lib/kubernetes/kubernetes-key.pem      # key matching the cert

         # kubeconfig files require a context. Provide one for the API server.
         current-context: webhook
         contexts:
         - context:
             cluster: kubernetes
             user: api-server-webhook
           name: webhook
         EOF
         ```
      </details>

1. image build
   ```
   sudo buildah bud -t k8s-kube-apiserver --file=Dockerfile_kube-apiserver.armhf ./
   ```

1. pod manifestsを `/etc/kubelet.d` へ作成する
   <details><summary>/etc/kubelet.d/kube-api-server.yaml</summary>
      ```
      cat << EOF | sudo tee /etc/kubelet.d/kube-api-server.yaml
      ---
      apiVersion: v1
      kind: Pod
      metadata:
        name: kube-apiserver
        namespace: kube-system
        annotations:
          seccomp.security.alpha.kubernetes.io/pod: runtime/default
        labels:
          tier: control-plane
          component: kube-apiserver

      spec:
        # https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
        priorityClassName: system-node-critical
        hostNetwork: true
        containers:
          - name: kube-apiserver
            image: localhost/k8s-kube-apiserver:latest
            imagePullPolicy: IfNotPresent
            resources:
              requests:
                cpu: "0.5"
                memory: "256Mi"
              limits:
                cpu: "1"
                memory: "384Mi"
            command:
              - /usr/bin/kube-apiserver
              - --advertise-address=192.168.10.50
              - --allow-privileged=true
              - --anonymous-auth=false
              - --apiserver-count=1
              - --audit-log-maxage=30
              - --audit-log-maxbackup=3
              - --audit-log-maxsize=100
              - --audit-log-path=/var/log/audit.log
              - --authorization-mode=Node,RBAC,Webhook
              - --authorization-webhook-config-file=/etc/kubernetes/webhook/authorization-config.yaml
              - --authentication-token-webhook-cache-ttl=2m
              - --authentication-token-webhook-version=v1
              - --bind-address=0.0.0.0
              - --client-ca-file=/var/lib/kubernetes/ca.pem
              - --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,RuntimeClass,PodSecurityPolicy
              - --etcd-cafile=/var/lib/kubernetes/ca.pem
              - --etcd-certfile=/var/lib/kubernetes/kubernetes.pem
              - --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem
              - --etcd-servers=https://192.168.10.50:2379
              - --event-ttl=1h
              - --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml
              - --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem
              - --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem
              - --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem
              - --kubelet-https=true
              - --runtime-config=authentication.k8s.io/v1beta1=true
              - --feature-gates=APIPriorityAndFairness=false
              - --service-account-key-file=/var/lib/kubernetes/service-account.pem
              - --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem
              - --service-account-issuer=api
              - --service-account-api-audiences=api
              - --service-cluster-ip-range=10.32.0.0/24
              - --service-node-port-range=30000-32767
              - --tls-cert-file=/var/lib/kubernetes/kubernetes.pem
              - --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem
              - --http2-max-streams-per-connection=3000
              - --max-requests-inflight=3000
              - --max-mutating-requests-inflight=1000
              - --v=2
      EOF
      ```
   </details>

1. `crictl` でコンテナ起動を確認する
   ```
   $ sudo crictl ps --name kube-apiserver
   CONTAINER           IMAGE                                                              CREATED             STATE               NAME                ATTEMPT             POD ID
   82c371fd9d99e       83e685a0b921ef5dd91eb3cdf208ba70690c1dd7decfc39bb3903be6ede752e6   24 seconds ago      Running             kube-apiserver      0                   6af4d1b99fa37
   ```

1. defaultのPodSecurityPolicy(PSP)を作成する
    - `staticPod` を作成する際にkubeletからmirror pod作成リクエストが拒否されないようにします ([参考](https://kubernetes.io/ja/docs/tasks/configure-pod-container/static-pod/))
      <details><summary>PSP / ClusterRole / ClusterRoleBinding</summary>
         ```
         cat << EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
         apiVersion: policy/v1beta1
         kind: PodSecurityPolicy
         metadata:
           annotations:
             apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
             apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
             seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default'
             seccomp.security.alpha.kubernetes.io/defaultProfileName:  'docker/default'
           name: default
         spec:
           allowedCapabilities: []  # default set of capabilities are implicitly allowed
           allowPrivilegeEscalation: false
           fsGroup:
             rule: 'MustRunAs'
             ranges:
               # Forbid adding the root group.
               - min: 1
                 max: 65535
           hostIPC: true
           hostNetwork: true
           hostPID: true
           privileged: true
           readOnlyRootFilesystem: true
           runAsUser:
             rule: 'MustRunAsNonRoot'
           seLinux:
             rule: 'RunAsNonRoot'
           supplementalGroups:
             rule: 'RunAsNonRoot'
             ranges:
               # Forbid adding the root group.
               - min: 1
                 max: 65535
           volumes:
           - 'configMap'
           - 'downwardAPI'
           - 'emptyDir'
           - 'persistentVolumeClaim'
           - 'projected'
           - 'secret'
           hostNetwork: true
           runAsUser:
             rule: 'RunAsAny'
           seLinux:
             rule: 'RunAsAny'
           supplementalGroups:
             rule: 'RunAsAny'
           fsGroup:
             rule: 'RunAsAny'

         ---

         # Cluster role which grants access to the default pod security policy
         apiVersion: rbac.authorization.k8s.io/v1
         kind: ClusterRole
         metadata:
           name: default-psp
         rules:
         - apiGroups:
           - policy
           resourceNames:
           - default
           resources:
           - podsecuritypolicies
           verbs:
           - use

         ---

         # Cluster role binding for default pod security policy granting all authenticated users access
         apiVersion: rbac.authorization.k8s.io/v1
         kind: ClusterRoleBinding
         metadata:
           name: default-psp
         roleRef:
           apiGroup: rbac.authorization.k8s.io
           kind: ClusterRole
           name: default-psp
         subjects:
         - apiGroup: rbac.authorization.k8s.io
           kind: Group
           name: system:authenticated
         EOF
         ```
      </details>

         ```
         $ cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -

           <省略>

         podsecuritypolicy.policy/default created
         clusterrole.rbac.authorization.k8s.io/default-psp created
         clusterrolebinding.rbac.authorization.k8s.io/default-psp created
         ```

1. master nodeにPodがscheduleされないようにする
   - taintを設定する
      - https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/
      - https://kubernetes.io/ja/docs/concepts/scheduling-eviction/taint-and-toleration/
         ```
         kubectl taint nodes k8s-master node-role.kubernetes.io/master:NoSchedule
         ```

         ```
         $ kubectl get node k8s-master -o=jsonpath='{.spec.taints}'
         [{"effect":"NoSchedule","key":"node-role.kubernetes.io/master"}]
         ```


## エラー事例

### 1.20のバグ?

```
failed creating mandatory flowcontrol settings: failed getting mandatory FlowSchema exempt due to the server was unable to return a response in the time allotted, but may still be processing the request
```

- 解決方法(2021/04/17時点)
   - https://github.com/kubernetes/kubernetes/issues/97525#issuecomment-753022219
      - kube-apiserver起動オプションに以下2つを付加する
         - `--feature-gates=APIPriorityAndFairness=false`
         - `--runtime-config=flowcontrol.apiserver.k8s.io/v1beta1=false`


### kubeletがWebhook認証を期待しているのにkube-api-serverでWebhook認証が有効でない場合

```
failed to run Kubelet: no client provided, cannot use webhook authentication
```

- 解決方法
   - Webhook認証を有効にする
   - https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/
   - https://kubernetes.io/docs/reference/access-authn-authz/webhook/
      - https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication

## 参考文献

- https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/
- https://kubernetes.io/docs/reference/access-authn-authz/webhook/

