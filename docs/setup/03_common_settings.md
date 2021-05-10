### 1. swapを無効にする

1. swapが使用されていることを確認
   ```
   $ free -h
                 total        used        free      shared  buff/cache   available
   Mem:          1.8Gi        54Mi       1.5Gi       8.0Mi       244Mi       1.7Gi
   Swap:          99Mi          0B        99Mi
   ```

1. swapを無効に設定する
   ```
   sudo swapoff --all
   sudo systemctl stop dphys-swapfile
   sudo systemctl disable dphys-swapfile
   ```

1. swapが無効であることを確認する
   ```
   $ free -h
                 total        used        free      shared  buff/cache   available
   Mem:          1.8Gi        57Mi       1.5Gi       8.0Mi       251Mi       1.7Gi
   Swap:            0B          0B          0B

   $ systemctl status dphys-swapfile
   ● dphys-swapfile.service - dphys-swapfile - set up, mount/unmount, and delete a swap file
      Loaded: loaded (/lib/systemd/system/dphys-swapfile.service; disabled; vendor preset: enabled)
      Active: inactive (dead)
        Docs: man:dphys-swapfile(8)

   12月 30 20:48:54 k8s-master1 systemd[1]: Starting dphys-swapfile - set up, mount/unmount, and delete a swap file...
   12月 30 20:48:55 k8s-master1 dphys-swapfile[330]: want /var/swap=100MByte, checking existing: keeping it
   12月 30 20:48:55 k8s-master1 systemd[1]: Started dphys-swapfile - set up, mount/unmount, and delete a swap file.
   12月 31 06:57:57 k8s-master1 systemd[1]: Stopping dphys-swapfile - set up, mount/unmount, and delete a swap file...
   12月 31 06:57:57 k8s-master1 systemd[1]: dphys-swapfile.service: Succeeded.
   12月 31 06:57:57 k8s-master1 systemd[1]: Stopped dphys-swapfile - set up, mount/unmount, and delete a swap file.
   ```

### 2. cgroupfs のmemoryを有効にする

1. kernelのboot optionに `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory` を追記する
   ```
   sudo vim /boot/firmware/cmdline.txt
   ```

     - `cmdline.txt` の有効行の確認

        ```
        $ sudo sed -e 's/\s/\n/g' /boot/firmware/cmdline.txt
        console=serial0,115200
        console=tty1
        root=PARTUUID=fb7271c3-02
        rootfstype=ext4
        elevator=deadline
        fsck.repair=yes
        rootwait
        quiet
        splash
        plymouth.ignore-serial-consoles
        cgroup_enable=cpuset
        cgroup_memory=1
        cgroup_enable=memory

        $ sudo reboot

        $ cat /proc/cgroups
        #subsys_name    hierarchy       num_cgroups     enabled
        cpuset  9       1       1
        cpu     5       34      1
        cpuacct 5       34      1
        blkio   10      34      1
        memory  8       80      1
        devices 4       34      1
        freezer 7       1       1
        net_cls 2       1       1
        perf_event      6       1       1
        net_prio        2       1       1
        pids    3       39      1
        ```

### 3. CRI-O インストール

https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o


1. kernel moduleのload
    - overlayファイルシステムを利用するためのkernel module `overlay`
    - iptablesがbridgeを通過するパケットを処理するためのkernel module `br_netfilter`
      ```
      cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
      overlay
      br_netfilter
      EOF

      sudo modprobe overlay
      sudo modprobe br_netfilter
      ```

1. kernel parameterのset
    - iptablesがbridgeを通過するパケットを処理するための設定
      ```
      cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf

      # https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o
      net.bridge.bridge-nf-call-iptables  = 1
      net.ipv4.ip_forward                 = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      EOF

      sudo sysctl --system
      ```

1. system起動時に kernel parameter を再読み込みさせる
   - kube-proxyにて必要なkernel parameter設定(kubelet設定手順にて後述) がiptables起動時のkernel module loadで上書きされるため
      - 利用環境が `/etc/sysconfig/iptables-config` を利用可能なら `IPTABLES_MODULES_UNLOAD="no"` を設定することで本手順は不要です
        ```
        egrep  "sysctl\s+--system" /etc/rc.local > /dev/null || sudo bash -c "echo \"sysctl --system\" >> /etc/rc.local"
        egrep  "sysctl\s+--system" /etc/rc.local
        ```

1. CRI-Oインストール
    - https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.20/xUbuntu_20.04/arm64/
      ```
      VERSION=1.20
      OS=xUbuntu_20.04

      sudo bash -c "echo \"deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /\" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list"
      sudo bash -c "echo \"deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /\" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list"

      curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | sudo apt-key add -
      curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key add -

      sudo apt update
      sudo apt install -y cri-o cri-o-runc

      sudo apt install -y conntrack
      ```

1. storage driverを `overlay2` へ変更する
   ```
   sudo vim /etc/containers/storage.conf
   sudo vim /etc/crio/crio.conf
   ```

      -  `/etc/crio/crio.conf` へgraph driver設定を入れる
        - podmanやbuildahでbuildしたlocal imageを参照するため
        - `[crio]` セクションに入れる
           ```
           graphroot = "/var/lib/containers/storage"
           ```
   <details><summary>/etc/containers/storage.conf</summary>
      ```
      [storage]
      driver = "overlay2"
      runroot = "/run/containers/storage"
      graphroot = "/var/lib/containers/storage"
      [storage.options]
      additionalimagestores = [
      ]
      [storage.options.overlay]
      mountopt = "nodev"
      [storage.options.thinpool]
      ```
   </details>
   <details><summary>/etc/crio/crio.conf</summary>
      ```
      [crio]
      storage_driver = "overlay2"
      graphroot = "/var/lib/containers/storage"
      log_dir = "/var/log/crio/pods"
      version_file = "/var/run/crio/version"
      version_file_persist = "/var/lib/crio/version"
      clean_shutdown_file = "/var/lib/crio/clean.shutdown"
      [crio.api]
      listen = "/var/run/crio/crio.sock"
      stream_address = "127.0.0.1"
      stream_port = "0"
      stream_enable_tls = false
      stream_idle_timeout = ""
      stream_tls_cert = ""
      stream_tls_key = ""
      stream_tls_ca = ""
      grpc_max_send_msg_size = 16777216
      grpc_max_recv_msg_size = 16777216
      [crio.runtime]
      no_pivot = false
      decryption_keys_path = "/etc/crio/keys/"
      conmon = ""
      conmon_cgroup = "system.slice"
      conmon_env = [
              "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      ]
      default_env = [
      ]
      seccomp_profile = ""
      seccomp_use_default_when_empty = false
      apparmor_profile = "crio-default"
      irqbalance_config_file = "/etc/sysconfig/irqbalance"
      cgroup_manager = "systemd"
      separate_pull_cgroup = ""
      default_capabilities = [
              "CHOWN",
              "DAC_OVERRIDE",
              "FSETID",
              "FOWNER",
              "SETGID",
              "SETUID",
              "SETPCAP",
              "NET_BIND_SERVICE",
              "KILL",
      ]
      default_sysctls = [
      ]
      additional_devices = [
      ]
      hooks_dir = [
              "/usr/share/containers/oci/hooks.d",
      ]pids_limit = 1024
      log_size_max = -1
      log_to_journald = false
      container_exits_dir = "/var/run/crio/exits"
      container_attach_socket_dir = "/var/run/crio"
      bind_mount_prefix = ""
      read_only = false
      log_level = "info"
      log_filter = ""
      uid_mappings = ""
      gid_mappings = ""
      ctr_stop_timeout = 30
      drop_infra_ctr = false
      infra_ctr_cpuset = ""
      namespaces_dir = "/var/run"
      pinns_path = ""
      default_runtime = "runc"
      [crio.runtime.runtimes.runc]
      runtime_path = ""
      runtime_type = "oci"
      runtime_root = "/run/runc"
      allowed_annotations = [
              "io.containers.trace-syscall",
      ]
      [crio.image]
      default_transport = "docker://"
      global_auth_file = ""
      pause_image = "k8s.gcr.io/pause:3.2"
      pause_image_auth_file = ""
      pause_command = "/pause"
      signature_policy = ""
      image_volumes = "mkdir"
      big_files_temporary_dir = ""
      [crio.network]
      network_dir = "/etc/cni/net.d/"
      plugin_dirs = [
              "/opt/cni/bin/",
      ]
      [crio.metrics]
      enable_metrics = false
      metrics_port = 9090
      metrics_socket = ""
      ```
   </details>

1. crioを再起動する
   ```
   sudo systemctl daemon-reload
   sudo systemctl restart crio
   ```

### 4. CLI TOOL(buildah, cri-tools)

1. buildah
    - https://github.com/containers/buildah/blob/master/install.md
       ```
       sudo apt-get -qq -y install buildah
       ```

1. cri-tools(crictl) インストール
    - https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md
       ```
       VERSION="v1.20.0"
       curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-arm.tar.gz --output crictl-${VERSION}-linux-arm.tar.gz
       sudo tar zxvf crictl-$VERSION-linux-arm.tar.gz -C /usr/local/bin
       rm -f crictl-$VERSION-linux-arm.tar.gz
       ```

1. podman インストール(optional)
    - https://podman.io/getting-started/installation
       ```
       sudo apt-get -y install podman
       sudo rm -f /etc/cni/net.d/87-podman-bridge.conflist
       ```

### 5. CNI Pluginインストール

- https://github.com/containernetworking/plugins
   ```
   CNI_VERSION="v0.9.1"
   ARCH="arm"
   curl -L "https://github.com/containernetworking/plugins/releases/download/${CNI_VERSION}/cni-plugins-linux-${ARCH}-${CNI_VERSION}.tgz" | sudo tar -C /opt/cni/bin -xz

   ls -l /opt/cni/bin/
   ```

### 6. kubectl インストール

- https://kubernetes.io/ja/docs/tasks/tools/install-kubectl/
