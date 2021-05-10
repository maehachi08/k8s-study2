# bootstrapping kubelet(master/worker 共通)

## 手順

1. `kubelet` バイナリをダウンロード
   ```
   sudo wget -P /usr/bin/ https://dl.k8s.io/v1.20.1/bin/linux/arm64/kubelet
   sudo chmod +x /usr/bin/kubelet
   ```

1. config, 証明書などを配置
   ```
   # host="k8s-node2"
   # host="k8s-node1"
   host="k8s-master"

   sudo install -o root -g root -m 755 -d /etc/kubelet.d
   sudo install -o root -g root -m 755 -d /var/lib/kubernetes
   sudo install -o root -g root -m 755 -d /var/lib/kubelet
   sudo cp ca.pem /var/lib/kubernetes/
   sudo cp ${host}.pem ${host}-key.pem ${host}.kubeconfig /var/lib/kubelet/
   sudo cp ${host}.kubeconfig /var/lib/kubelet/kubeconfig
   ```

1. `/var/lib/kubelet/kubelet-config.yaml` を作成する
    - `clusterDNS` は kube-dns(core-dns)のClusterIPを指定する
    - `podCIDR` はnodeで起動するPodに割り当てるIPアドレスのCIDRを指定する
      ```
      # host="k8s-node2"
      # host="k8s-node1"
      host="k8s-master"

      cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
      ---
      kind: KubeletConfiguration
      apiVersion: kubelet.config.k8s.io/v1beta1

      # https://kubernetes.io/ja/docs/tasks/configure-pod-container/static-pod/
      staticPodPath: /etc/kubelet.d

      # kubeletの認証方式
      #   - anonymous: false が(コンテナ実行ホストのHardeningとして)推奨される
      #   - webhook.enabled: true の場合はkube-api-server側でも諸処の設定が必要
      authentication:
        anonymous:
          enabled: true
        webhook:
          enabled: false
          cacheTTL: "2m"
        x509:
          clientCAFile: "/var/lib/kubernetes/ca.pem"

      # kubeletの認可設定
      #   - authorization.mode のdefault動作は AlwaysAllow
      #   - authorization.mode: Webhook の場合は kube-api-serverで authorization.k8s.io/v1beta1 の有効設定が必要
      authorization:
        mode: AlwaysAllow

      clusterDomain: "cluster.local"
      clusterDNS:
        - "10.32.0.10"
      podCIDR: "10.200.0.0/24"
      resolvConf: "/etc/resolv.conf"
      runtimeRequestTimeout: "15m"
      tlsCertFile: "/var/lib/kubelet/${host}.pem"
      tlsPrivateKeyFile: "/var/lib/kubelet/${host}-key.pem"

      # Reserve Compute Resources for System Daemons
      # https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/
      #
      # Podに配置可能なリソースは "Node resource - system-reserved - kube-reserved - eviction-threshold" らしい
      #
      # system-reserved
      #   - OS system daemons(ssh, udev, etc) 用にリソースを確保する
      #
      # kube-reserved
      #   - k8s system daemons(kubelet, container runtime, node problem detector) 用にリソースを確保する
      enforceNodeAllocatable: ["pods","kube-reserved","system-reserved"]
      cgroupsPerQOS: true
      cgroupDriver: systemd
      cgroupRoot: /
      systemCgroups: /system.slice
      systemReservedCgroup: /system.slice
      systemReserved:
        cpu: 500m
        memory: 300Mi
      runtimeCgroups: /kube.slice/crio.service
      kubeletCgroups: /kube.slice/kubelet.service
      kubeReservedCgroup: /kube.slice
      kubeReserved:
        cpu: 100m
        memory: 100Mi
      EOF
      ```

1. `/etc/systemd/system/kubelet.service` を配置
   ```
   cat <<'EOF' | sudo tee /etc/systemd/system/kubelet.service
   [Unit]
   Description=Kubernetes Kubelet
   Documentation=https://github.com/kubernetes/kubernetes
   After=crio.service
   Requires=crio.service

   [Service]
   Restart=on-failure
   RestartSec=5

   ExecStartPre=/usr/bin/mkdir -p \
     /sys/fs/cgroup/systemd/kube.slice \
     /sys/fs/cgroup/cpuset/kube.slice \
     /sys/fs/cgroup/cpuset/system.slice \
     /sys/fs/cgroup/pids/kube.slice \
     /sys/fs/cgroup/pids/system.slice \
     /sys/fs/cgroup/memory/kube.slice \
     /sys/fs/cgroup/memory/system.slice \
     /sys/fs/cgroup/cpu,cpuacct/kube.slice \
     /sys/fs/cgroup/cpu,cpuacct/kube.slice

   ExecStart=/usr/bin/kubelet \
     --config=/var/lib/kubelet/kubelet-config.yaml \
     --kubeconfig=/var/lib/kubelet/kubeconfig \
     --network-plugin=cni \
     --container-runtime=remote \
     --container-runtime-endpoint=/var/run/crio/crio.sock \
     --register-node=true \
     --v=2

   [Install]
   WantedBy=multi-user.target
   EOF
   ```

1. `kubelet.service` を起動
   ```
   sudo systemctl enable kubelet.service
   sudo systemctl start kubelet.service
   ```

## エラー事例

### cgroupディレクトリが未作成の場合

```
kubelet.go:1347] Failed to start ContainerManager Failed to enforce Kube Reserved Cgroup Limits on "/kube.slice": ["kube"] cgroup does not exist
```

   - `kubelet` 起動オプションで詳細なログを出すことでPathがわかった( `--v 10` )

      ```
      cgroup_manager_linux.go:294] The Cgroup [kube] has some missing paths: [/sys/fs/cgroup/pids/kube.slice /sys/fs/cgroup/memory/kube.slice]
      ```

### cgroupで確保するsystemReserved memory sizeが小さい場合に発生

- 原因などは未調査、systemReserved memoryを大きくしたら発生しなくなった
   ```
   kubelet.go:1347] Failed to start ContainerManager Failed to enforce System Reserved Cgroup Limits on "/system.slice": failed to set supported cgroup subsystems for cgroup [system]: failed to set config for supported subsystems : failed to write "104857600" to "/sys/fs/cgroup/memory/system.slice/memory.limit_in_bytes": write /sys/fs/cgroup/memory/system.slice/memory.limit_in_bytes: device or resource busy
   ```


### kubeconfig の証明書の `CN` がnode ホスト名と異なる


```
360163 kubelet_node_status.go:93] Unable to register node "k8s-master" with API server: nodes "k8s-master" is forbidden: node "k8s-node1" is not allowed to modify node "k8s-master"
```

   - kubeconfigのclient-certificate-dataのCNを確認する
      ```
      sudo cat k8s-master.kubeconfig | grep client-certificate-data | awk '{print $2;}' | base64 -d | openssl x509 -text | grep Subject:
      ```
       - `k8s-master` が正しいのに `CN = system:node:k8s-node1` となっていた
           ```
           root@k8s-master:~# cat /var/lib/kubelet/kubeconfig | grep client-certificate-data | awk '{print $2;}' | base64 -d | openssl x509 -text | grep Subject:
                Subject: C = JP, ST = Sample, L = Tokyo, O = system:nodes, OU = Kubernetes The HardWay, CN = system:node:k8s-master
           ```

### `Node` リソースの `spec.podCIDR` にCIDRが設定されない

- 以下コマンドでnodeに設定したpodCIDRが表示されない
    - flannnelが起動しない原因がここにあった...
     ```
     kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'
     ```

- `kube-controller-manager` のログ
    - `Set node k8s-node1 PodCIDR to [10.200.0.0/24]` が出ることがポイント
        - `kube-controller-manager` の起動オプションに `--allocate-node-cidrs=true` が必要ってお話...
        ```
        actual_state_of_world.go:506] Failed to update statusUpdateNeeded field in actual state of world: Failed to set statusUpdateNeeded to needed true, because nodeName="k8s-node1" does not exist
        range_allocator.go:373] Set node k8s-node1 PodCIDR to [10.200.0.0/24]
        ttl_controller.go:276] "Changed ttl annotation" node="k8s-node1" new_ttl="0s"
        controller.go:708] Detected change in list of current cluster nodes. New node set: map[k8s-node1:{}]
        controller.go:716] Successfully updated 0 out of 0 load balancers to direct traffic to the updated set of nodes
        node_lifecycle_controller.go:773] Controller observed a new Node: "k8s-node1"
        controller_utils.go:172] Recording Registered Node k8s-node1 in Controller event message for node k8s-node1
        node_lifecycle_controller.go:1429] Initializing eviction metric for zone:
        node_lifecycle_controller.go:1044] Missing timestamp for Node k8s-node1. Assuming now as a timestamp.
        event.go:291] "Event occurred" object="k8s-node1" kind="Node" apiVersion="v1" type="Normal" reason="RegisteredNode" message="Node k8s-node1 event: Registered Node k8s-node1 in Controller"
        node_lifecycle_controller.go:1245] Controller detected that zone  is now in state Normal.
        ```

### Webhook Authenticationの設定が正しくない

```
I0214 07:03:56.822586       1 dynamic_cafile_content.go:129] Loaded a new CA Bundle and Verifier for "client-ca-bundle::/var/lib/kubernetes/ca.pem"
F0214 07:03:56.822637       1 server.go:269] failed to run Kubelet: no client provided, cannot use webhook authentication
goroutine 1 [running]:
```

  - https://kubernetes.io/docs/reference/access-authn-authz/webhook/
  - https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication


### CNI Pluginを `/etc/cni/net.d` でCNI Pluginが見つからない

```
kubelet.go:2163] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
cni.go:239] Unable to update cni config: no networks found in /etc/cni/net.d
kubelet.go:2163] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

   - CNI Pluginを `/etc/cni/net.d` へ置くことで解決する
      - https://github.com/containernetworking/plugins/releases

### Kubelet cannot determine CPU online state

```
sysinfo.go:203] Nodes topology is not available, providing CPU topology
sysfs.go:348] unable to read /sys/devices/system/cpu/cpu0/online: open /sys/devices/system/cpu/cpu0/online: no such file or directory
sysfs.go:348] unable to read /sys/devices/system/cpu/cpu1/online: open /sys/devices/system/cpu/cpu1/online: no such file or directory
sysfs.go:348] unable to read /sys/devices/system/cpu/cpu2/online: open /sys/devices/system/cpu/cpu2/online: no such file or directory
sysfs.go:348] unable to read /sys/devices/system/cpu/cpu3/online: open /sys/devices/system/cpu/cpu3/online: no such file or directory
gce.go:44] Error while reading product_name: open /sys/class/dmi/id/product_name: no such file or directory
machine.go:253] Cannot determine CPU /sys/bus/cpu/devices/cpu0 online state, skipping
machine.go:253] Cannot determine CPU /sys/bus/cpu/devices/cpu1 online state, skipping
machine.go:253] Cannot determine CPU /sys/bus/cpu/devices/cpu2 online state, skipping
machine.go:253] Cannot determine CPU /sys/bus/cpu/devices/cpu3 online state, skipping
machine.go:72] Cannot read number of physical cores correctly, number of cores set to 0
machine.go:253] Cannot determine CPU /sys/bus/cpu/devices/cpu0 online state, skipping
machine.go:253] Cannot determine CPU /sys/bus/cpu/devices/cpu1 online state, skipping
machine.go:253] Cannot determine CPU /sys/bus/cpu/devices/cpu2 online state, skipping
machine.go:253] Cannot determine CPU /sys/bus/cpu/devices/cpu3 online state, skipping
machine.go:86] Cannot read number of sockets correctly, number of sockets set to 0
container_manager_linux.go:490] [ContainerManager]: Discovered runtime cgroups name:
```

   - 既知らしい
      - https://github.com/kubernetes/kubernetes/issues/95039

## 参考

- https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/
- https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/kubelet/config/v1beta1/types.go
- https://cyberagent.ai/blog/tech/4036/
    - kubelet の設定を変更して runtime に cri-o を指定する
- https://downloadkubernetes.com/
- https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/

- Node Authorization
    - https://qiita.com/tkusumi/items/f6a4f9150aa77d8f9822
    - https://kubernetes.io/docs/reference/access-authn-authz/node/
    - https://kubernetes.io/ja/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/

- static pod
    - https://kubernetes.io/ja/docs/tasks/configure-pod-container/static-pod/
    - https://kubernetes.io/docs/concepts/policy/pod-security-policy/
    - https://hakengineer.xyz/2019/07/04/post-1997/#03_master1kube-schedulerkube-controller-managerkube-apiserver
    - `PodSecurityPolicy` を参照した元ネタ(`false` になっているのは `true` に直す)
       - https://github.com/kubernetes/kubernetes/issues/70952
