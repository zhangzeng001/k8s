# cfssl 生成ca证书

https://github.com/coreos/docs/blob/master/os/generate-self-signed-certificates.md

## 安装

```shell
mkdir ~/bin
curl -s -L -o ~/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -s -L -o ~/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
curl -s -L -o ~/bin/cfsslcertinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x ~/bin/*
export PATH=$PATH:~/bin
```

## 初始化证书颁发机构

```shell
mkdir ~/cfssl
cd ~/cfssl
cfssl print-defaults config > ca-config.json
cfssl print-defaults csr > ca-csr.json
```

## 创建CA

Now we can configure signing options inside `ca-config.json` config file. Default options contain following preconfigured fields:

- profiles: **www** with `server auth` (TLS Web Server Authentication) X509 V3 extension and **client** with `client auth` (TLS Web Client Authentication) X509 V3 extension.
- expiry: with `8760h` default value (or 365 days)

CA 配置文件用于配置根证书的使用场景 (profile) 和具体参数 (usage，过期时间、服务端认证、客户端认证、加密等)，后续在签名其它证书时需要指定特定场景。 

`vim ca-config.json`

<font color=FF0000>这里指定了3个profile，所以要分别生成3个证书</font>

```json
{
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {
            "etcd": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
```

其中：

`ca-config.json`：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；此实例只有一个etcd模板，也可以命名为其他。
`signing`：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE；
`server auth`： 表示client可以用该 CA 对server提供的证书进行验证；
`client auth`：  表示server可以用该CA对client提供的证书进行验证；
注意标点符号，最后一个字段一般是没有都好的。

## 创建证书签名请求文件 

`ca-csr.json`

```json
{
    "CN": "etcd",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            # country
            "C": "CN",
            # city
            "L": "BeiJing",
            # organization
            "O": "My Company Name",
            # state
            "ST": "BeiJing",
            # organization unit
            "OU": "etcd"
        }
    ]
}
```

国家  省    组织   市  组织机构

CN：Common Name ，kube-apiserver 从证书中提取该字段作为请求的用户名(User Name)，浏览器使用该字段验证网站是否合法；
O： Organization ，kube-apiserver 从证书中提取该字段作为请求用户所属的组(Group)；
kube-apiserver 将提取的 User、Group 作为 RBAC 授权的用户标识； 

## 生成ca

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```

 您将获得以下文件： 

```
ca-key.pem
ca.csr
ca.pem
```

- 请保持`ca-key.pem`文件安全。使用此密钥可以在您的CA中创建任何种类的证书。
- *** .csr**文件未在我们的示例中使用。

## 验证

```shell
openssl x509 -in etcd-root-ca.pem -text -noout
```



## 生成客户证书

```
cfssl print-defaults csr > server.json
```

```
...
    "CN": "coreos1",
    "hosts": [
        "192.168.122.68",
        "ext.example.com",
        "coreos1.local",
        "coreos1"
    ],
...
```

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server.json | cfssljson -bare server
```

 Or without CSR json file: 

```
echo '{"CN":"coreos1","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server -hostname="192.168.122.68,ext.example.com,coreos1.local,coreos1" - | cfssljson -bare server
```

 You'll get following files: 

```
server-key.pem
server.csr
server.pem
```

## 生成对等证书

```
cfssl print-defaults csr > member1.json
```

```
...
    "CN": "member1",
    "hosts": [
        "192.168.122.101",
        "ext.example.com",
        "member1.local",
        "member1"
    ],
...
```

 Now we are ready to generate member1 certificate and private key: 

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer member1.json | cfssljson -bare member1
```

```
member1-key.pem
member1.csr
member1.pem
```

## 生成客户证书

```
cfssl print-defaults csr > client.json
```

```
...
    "CN": "client",
    "hosts": [""],
...
```

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare client
```

```
client-key.pem
client.csr
client.pem
```



# 配置etcd集群

http://play.etcd.io/install     工作流生成执行命令

https://etcd.io/docs/current/op-guide/security/     TLS验证注意事项

https://etcd.io/docs/current/op-guide/clustering/   集群配置

## 创建ca

`cat > ca-config.json <<EOF`

```json
{
  "signing": {
    "default": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "87600h"
    }
  }
}
EOF
```

## 创建证书签名请求文件

`ca-csr.json`

```json
{
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "etcd",
      "OU": "etcd Security",
      "L": "XiAn",
      "ST": "Shanxi",
      "C": "CN"
    }
  ],
  "CN": "etcd-root-ca"
}
EOF
```

## 生成ca

```shell
cfssl gencert --initca=true ca-csr.json | cfssljson --bare etcd-root-ca
```

结果：

```shell
[root@master1 cfssl]# ll
total 20
-rw-r--r-- 1 root root  844 Feb 23 22:02 ca-config.json
-rw-r--r-- 1 root root  256 Feb 23 22:21 ca-csr.json
# 一下3个
-rw-r--r-- 1 root root  989 Feb 23 23:05 etcd-root-ca.csr
-rw------- 1 root root 1679 Feb 23 23:05 etcd-root-ca-key.pem
-rw-r--r-- 1 root root 1338 Feb 23 23:05 etcd-root-ca.pem
```

## 验证

```shell
openssl x509 -in etcd-root-ca.pem -text -noout
```

## 使用私钥生成本地颁发的证书 

这里不适用官网每个机器都颁发证书(扩容新节点可能需要新证书，因为证书hosts未指定新节点)

`vim etcd-csr.json`

```shell
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "10.0.0.10",
    "10.0.0.11",
    "10.0.0.12"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
        "C": "CN",
        "L": "BeiJing",
        "O": "xxx",
        "ST": "BeiJing",
        "OU": "etcd"
    }
  ]
}
```

生成证书：

```shell
cfssl gencert --ca=./etcd-root-ca.pem \
    --ca-key=./etcd-root-ca-key.pem \
    --config=./ca-config.json \
    --profile=default etcd-csr.json | cfssljson -bare etcdcluster
```

**遇到错误:**

多hosts警告：（**可忽略，验证生成证书已经可以看到所有hosts了**）

https://github.com/cloudflare/cfssl/issues/717

-hostname="192.168.122.101,ext.example.com,member1.local,member1"  （未验证）

```shell
[root@master1 cfssl]# cfssl gencert --ca=./etcd-root-ca.pem     --ca-key=./etcd-root-ca-key.pem     --config=./ca-config.json     --profile=default etcd-csr.json | cfssljson -bare etcdcluster
2021/02/24 00:18:44 [INFO] generate received request
2021/02/24 00:18:44 [INFO] received CSR
2021/02/24 00:18:44 [INFO] generating key: rsa-2048
2021/02/24 00:18:44 [INFO] encoded CSR
2021/02/24 00:18:44 [INFO] signed certificate with serial number 242746905542346112869228926327191922294451909970
2021/02/24 00:18:44 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
```



## 验证

```shell
openssl x509 -in etcdcluster.pem --text -noout
```



## 启动集群

https://etcd.io/docs/current/op-guide/clustering/



```shell
cd /root/cfssl/
cp etcdcluster* /opt/ssl/
cp etcd-root-ca.pem /opt/ssl/
```

**master1**

```shell
etcd --name master1 --data-dir /opt/app/etcd/data --initial-advertise-peer-urls https://10.0.0.10:2380  --listen-peer-urls https://10.0.0.10:2380 \
  --listen-client-urls https://10.0.0.10:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.0.10:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster master1=https://10.0.0.10:2380,master2=https://10.0.0.11:2380,master3=https://10.0.0.12:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=/opt/ssl/etcd-root-ca.pem \
  --cert-file=/opt/ssl/etcdcluster.pem --key-file=/opt/ssl/etcdcluster-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=/opt/ssl/etcd-root-ca.pem \
  --peer-cert-file=/opt/ssl/etcdcluster.pem --peer-key-file=/opt/ssl/etcdcluster-key.pem
```

**master2**

```shell
etcd --name master2 --data-dir /opt/app/etcd/data --initial-advertise-peer-urls https://10.0.0.11:2380 \
  --listen-peer-urls https://10.0.0.11:2380 \
  --listen-client-urls https://10.0.0.11:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.0.11:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster master1=https://10.0.0.10:2380,master2=https://10.0.0.11:2380,master3=https://10.0.0.12:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=/opt/ssl/etcd-root-ca.pem \
  --cert-file=/opt/ssl/etcdcluster.pem --key-file=/opt/ssl/etcdcluster-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=/opt/ssl/etcd-root-ca.pem \
  --peer-cert-file=/opt/ssl/etcdcluster.pem --peer-key-file=/opt/ssl/etcdcluster-key.pem
```

**master3**

```shell
etcd --name master3 --data-dir /opt/app/etcd/data --initial-advertise-peer-urls https://10.0.0.12:2380 \
  --listen-peer-urls https://10.0.0.12:2380 \
  --listen-client-urls https://10.0.0.12:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.0.12:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster master1=https://10.0.0.10:2380,master2=https://10.0.0.11:2380,master3=https://10.0.0.12:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=/opt/ssl/etcd-root-ca.pem \
  --cert-file=/opt/ssl/etcdcluster.pem --key-file=/opt/ssl/etcdcluster-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=/opt/ssl/etcd-root-ca.pem \
  --peer-cert-file=/opt/ssl/etcdcluster.pem --peer-key-file=/opt/ssl/etcdcluster-key.pem
```



**健康检查**

```shell
etcdctl --endpoints https://10.0.0.10:2379,https://10.0.0.11:2379,https://10.0.0.12:2379 --cacert /opt/ssl/etcd-root-ca.pem --cert /opt/ssl/etcdcluster-key.pem --key /opt/ssl/etcdcluster-key.pem endpoint health
```

输出：

```shell
https://10.0.0.10:2379 is healthy: successfully committed proposal: took = 13.300639ms
https://10.0.0.12:2379 is healthy: successfully committed proposal: took = 13.830267ms
https://10.0.0.11:2379 is healthy: successfully committed proposal: took = 17.139262ms
```



**简单操作**

```shell
etcdctl --endpoints https://10.0.0.10:2379,https://10.0.0.11:2379,https://10.0.0.12:2379 --cacert /opt/ssl/etcd-root-ca.pem --cert /opt/ssl/etcdcluster.pem --key /opt/ssl/etcdcluster-key.pem put /test/k1 v1
```



## 使用supervisord管理

```shell
yum install supervisor.noarch -y
cat > /etc/supervisord.d/etcd-server.ini << EOF
[program:etcd-server]
directory=/opt/etcd
command=/opt/etcd/etcd-server-startup.sh
autostart=true
autorestart=true
startsecs=10
stdout_logfile=/var/log/etcd.log
stdout_logfile_maxbytes=1MB
stdout_logfile_backups=5
stdout_capture_maxbytes=1MB
stderr_logfile=/var/log/etcd_error.log
stderr_logfile_maxbytes=1MB
stderr_logfile_backups=5
stderr_capture_maxbytes=1MB
EOF


systemctl start supervisord
systemctl enable supervisord
```

## 构建systemd服务

http://play.etcd.io/install





