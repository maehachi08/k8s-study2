# 認証局の設定とTLS証明書の作成

## 手順

### cfssl インストール

- 証明書を作成するための [cfssl](https://github.com/cloudflare/cfssl) をインストールする
    - https://qiita.com/iaoiui/items/fc2ea829498402d4a8e3
    - https://coreos.com/os/docs/latest/generate-self-signed-certificates.html
      ```
      sudo apt install golang-cfssl
      ```

### CA(認証局) 作成

```
cat > ca-config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "8760h"
        },
        "profiles": {
            "kubernetes": {
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ],
                "expiry": "8760h"
            }
        }
    }
}
EOF

cat > ca-csr.json <<EOF
{
    "CN": "Kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "JP",
            "L": "Tokyo",
            "O": "Kubernetes",
            "OU": "CA",
            "ST": "Sample"
        }
    ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

- 以下ファイルが生成されていることを確認
    - `ca-key.pem`
    - `ca.pem`

### 証明書の作成
#### 管理者ユーザ 証明書

```
cat > admin-csr.json <<EOF
{
    "CN": "admin",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "JP",
            "L": "Tokyo",
            "O": "system:masters",
            "OU": "Kubernetes The HardWay",
            "ST": "Sample"
        }
    ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

- 以下ファイルが生成されていることを確認
     - `admin-key.pem`
     - `admin.pem`

#### kubeletのクライアント証明書

- `EXTERNAL_IP`
    - masterサーバのIPアドレス
    - masterが複数サーバ構成の場合は上位のLB IP

```
for instance in k8s-master k8s-node1 k8s-node2; do
cat > ${instance}-csr.json <<EOF
{
   "CN": "system:node:${instance}",
   "key": {
       "algo": "rsa",
       "size": 2048
   },
   "names": [
       {
           "C": "JP",
           "L": "Tokyo",
           "O": "system:nodes",
           "OU": "Kubernetes The HardWay",
           "ST": "Sample"
       }
   ]
}
EOF

EXTERNAL_IP=192.168.10.50

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

- 以下ファイルが生成されていることを確認
    - `k8s-node1-key.pem`
    - `k8s-node1.pem`
    - `k8s-node2-key.pem`
    - `k8s-node2.pem`

#### kube-proxyのクライアント証明書

```
cat > kube-proxy-csr.json <<EOF
{
    "CN": "system:kube-proxy",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "JP",
            "L": "Tokyo",
            "O": "system:node-proxier",
            "OU": "Kubernetes The Hard Way",
            "ST": "Sample"
        }
    ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

- 以下ファイルが生成されていることを確認
    - `kube-proxy-key.pem`
    - `kube-proxy.pem`

#### kube-controller-manageのクライアント証明書

```
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Sample"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

- 以下ファイルが生成されていることを確認
    - `kube-controller-manager-key.pem`
    - `kube-controller-manager.pem`

#### kube-schedulerのクライアント証明書

```
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Sample"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

- 以下ファイルが生成されていることを確認
    - `kube-scheduler-key.pem`
    - `kube-scheduler.pem`

#### kube-apiserverのサーバー証明書

- `10.32.0.1`
    - Cluster IP

```
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
    "CN": "Kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "JP",
            "L": "Tokyo",
            "O": "Kubernetes",
            "OU": "Kubernetes The Hard Way",
            "ST": "Sample"
        }
    ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,192.168.10.50,192.168.10.51,192.168.10.52,127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

- 以下ファイルが生成されていることを確認
    - `kubernetes-key.pem`
    - `kubernetes.pem`

#### service-accountの証明書

```
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "JP",
      "L": "Tokyo",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Sample"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

- 以下ファイルが生成されていることを確認
    - `service-account-key.pem`
    - `service-account.pem`

### 証明書をmaster/nodeへコピーする

- master
- node

## 参考文献
   - https://kubernetes.io/ja/docs/setup/best-practices/certificates/
   - https://kubernetes.io/ja/docs/concepts/cluster-administration/certificates/
   - https://docs.oracle.com/cd/F34086_01/kubernetes-on-oci_jp.pdf

