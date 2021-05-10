# health check

## 手順

### 各コンポーネントの起動確認

   ```
   kubectl get pods -n kube-system
   ```
   <details><summary>実行例</summary>
      ```
      $ kubectl get pods -n kube-system
      NAME                                 READY   STATUS    RESTARTS   AGE
      etcd-k8s-master                      1/1     Running   0          5m56s
      kube-apiserver-k8s-master            1/1     Running   0          6m7s
      kube-controller-manager-k8s-master   1/1     Running   0          4m2s
      kube-scheduler-k8s-master            1/1     Running   0          2m48s
      ```
   </details>

### master node上のresource確認

   ```
   kubectl get nodes
   kubectl describe node <pod_name>
   ```
   <details><summary>実行例</summary>
      ```
      $ kubectl get nodes
      NAME         STATUS   ROLES    AGE     VERSION
      k8s-master   Ready    <none>   7m57s   v1.20.1

      $ kubectl describe node k8s-master
      Name:               k8s-master
      Roles:              <none>
      Labels:             beta.kubernetes.io/arch=arm64
                          beta.kubernetes.io/os=linux
                          kubernetes.io/arch=arm64
                          kubernetes.io/hostname=k8s-master
                          kubernetes.io/os=linux
      Annotations:        node.alpha.kubernetes.io/ttl: 0
                          volumes.kubernetes.io/controller-managed-attach-detach: true
      CreationTimestamp:  Sat, 17 Apr 2021 15:13:42 +0000
      Taints:             node-role.kubernetes.io/master:NoSchedule
      Unschedulable:      false
      Lease:
        HolderIdentity:  k8s-master
        AcquireTime:     <unset>
        RenewTime:       Sat, 17 Apr 2021 16:34:29 +0000
      Conditions:
        Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
        ----             ------  -----------------                 ------------------                ------                       -------
        MemoryPressure   False   Sat, 17 Apr 2021 16:34:09 +0000   Sat, 17 Apr 2021 15:13:41 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
        DiskPressure     False   Sat, 17 Apr 2021 16:34:09 +0000   Sat, 17 Apr 2021 15:13:41 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
        PIDPressure      False   Sat, 17 Apr 2021 16:34:09 +0000   Sat, 17 Apr 2021 15:13:41 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
        Ready            True    Sat, 17 Apr 2021 16:34:09 +0000   Sat, 17 Apr 2021 15:13:52 +0000   KubeletReady                 kubelet is posting ready status. AppArmor enabled
      Addresses:
        InternalIP:  192.168.10.50
        Hostname:    k8s-master
      Capacity:
        cpu:                4
        ephemeral-storage:  30459624Ki
        memory:             1892528Ki
        pods:               110
      Allocatable:
        cpu:                3400m
        ephemeral-storage:  28071589432
        memory:             1380528Ki
        pods:               110
      System Info:
        Machine ID:                 58f6de70444c4198b56b30122b6c77dc
        System UUID:                58f6de70444c4198b56b30122b6c77dc
        Boot ID:                    79af3428-cf70-4189-a447-0b917a035a42
        Kernel Version:             5.4.0-1032-raspi
        OS Image:                   Ubuntu 20.04.2 LTS
        Operating System:           linux
        Architecture:               arm64
        Container Runtime Version:  cri-o://1.20.2
        Kubelet Version:            v1.20.1
        Kube-Proxy Version:         v1.20.1
      PodCIDR:                      10.200.0.0/24
      PodCIDRs:                     10.200.0.0/24
      Non-terminated Pods:          (4 in total)
        Namespace                   Name                                  CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
        ---------                   ----                                  ------------  ----------  ---------------  -------------  ---
        kube-system                 etcd-k8s-master                       500m (14%)    1 (29%)     256Mi (18%)      384Mi (28%)    78m
        kube-system                 kube-apiserver-k8s-master             500m (14%)    1 (29%)     256Mi (18%)      384Mi (28%)    78m
        kube-system                 kube-controller-manager-k8s-master    100m (2%)     300m (8%)   128Mi (9%)       256Mi (18%)    76m
        kube-system                 kube-scheduler-k8s-master             100m (2%)     300m (8%)   128Mi (9%)       256Mi (18%)    75m
      Allocated resources:
        (Total limits may be over 100 percent, i.e., overcommitted.)
        Resource           Requests     Limits
        --------           --------     ------
        cpu                1200m (35%)  2600m (76%)
        memory             768Mi (56%)  1280Mi (94%)
        ephemeral-storage  0 (0%)       0 (0%)
      Events:              <none>
      ```
   </details>

### health checks

kube-apiserverの起動オプションで `--anonymous-auth=false` を付加しているため `https://localhost:6443` へのanonymousアカウントでの確認は行わずに `kubectl` で確認する

#### API endpoints for health

```
kubectl get --raw='/readyz?verbose'
```

#### Individual health checks

```
kubectl get --raw='/livez/etcd'
```

## 参考資料

- https://kubernetes.io/docs/reference/using-api/health-checks/

