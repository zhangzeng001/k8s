## 故障现象

https://blog.csdn.net/ywq935/article/details/88355832

使用kubeadm部署的集群，在运行了一年之后今天，出现k8s api无法调取的现象，使用kubectl命令获取资源均返回如下报错:

```
Unable to connect to the server: x509: certificate has expired or is not yet valid
1
```

## 故障排查

查看apiserver.crt证书的签署日期和过期日期：

```
root@9027:/etc/etcd/ssl# openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text |grep ' Not '
            Not Before: Mar  8 03:49:29 2018 GMT
            Not After : Mar  7 05:44:27 2020 GMT
123
```

发现恰好刚到过期日期，去github查询才得知kubeadm默认的证书签署过期时间为1年，大坑！google找到方法说可以通过修改kubeadm源代码调整证书签署的过期时间然后重新编译，即可预防这个问题。但是现在证书过期的问题已经发生了，只能通过更换证书的办法解决，在github上有详细的说明，issue链接如下:
https://github.com/kubernetes/kubeadm/issues/581

#### 开始替换apiserver证书

进入master节点

```
cd /etc/kubernetes
# 备份证书和配置
mkdir ./pki_bak
mkdir ./conf_bak
mv pki/apiserver* ./pki_bak/
mv pki/front-proxy-client.* ./pki_bak/
mv ./admin.conf ./conf_bak/
mv ./kubelet.conf ./conf_bak/
mv ./controller-manager.conf ./conf_bak/
mv ./scheduler.conf ./conf_bak/

# 创建证书
kubeadm alpha phase certs apiserver --apiserver-advertise-address ${MASTER_API_SERVER_IP}
kubeadm alpha phase certs apiserver-kubelet-client
kubeadm alpha phase certs front-proxy-client

# 生成新配置文件
kubeadm alpha phase kubeconfig all --apiserver-advertise-address ${MASTER_API_SERVER_IP}

# 将新生成的admin配置文件覆盖掉原本的admin文件
mv $HOME/.kube/config $HOME/.kube/config.old
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
sudo chmod 777 $HOME/.kube/config

12345678910111213141516171819202122232425
```

完成上方操作后，docker restart重启kube-apiserver,kube-controller,kube-scheduler这3个容器

查看证书的过期时间:

```
root@009027:~# openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text |grep ' Not '
            Not Before: Mar  8 03:49:29 2018 GMT
            Not After : Mar  7 05:44:27 2020 GMT
123
```

可以看到证书有效期延长了一年

如果有多台master节点，先仿照上方将证书文件和配置文件进行备份，然后将这一台配置完成的master上的证书和配置scp过去:

```
scp pki/* root@${other_master}:/etc/kubernetes/pki/
scp admin.conf kubelet.conf controller-manager.conf scheduler.conf root@${other_master}:/etc/kubernetes/
12
```

scp完成后记得重启相应master上的3个kubernetes组件容器。

### 验证

kubectl命令发现还是无法查看资源，检查apiserver的日志: kubectl logs ${api-server_container_id}

```
I0308 06:26:52.956024       1 serve.go:89] Serving securely on [::]:6443
I0308 06:26:52.956168       1 available_controller.go:262] Starting AvailableConditionController
I0308 06:26:52.956180       1 apiservice_controller.go:112] Starting APIServiceRegistrationController
I0308 06:26:52.956244       1 cache.go:32] Waiting for caches to sync for APIServiceRegistrationController controller
E0308 06:26:52.956279       1 cache.go:35] Unable to sync caches for APIServiceRegistrationController controller
I0308 06:26:52.956291       1 apiservice_controller.go:116] Shutting down APIServiceRegistrationController
I0308 06:26:52.956325       1 crd_finalizer.go:242] Starting CRDFinalizer
I0308 06:26:52.956195       1 cache.go:32] Waiting for caches to sync for AvailableConditionController controller
I0308 06:26:52.956342       1 crd_finalizer.go:246] Shutting down CRDFinalizer
I0308 06:26:52.956363       1 customresource_discovery_controller.go:152] Starting DiscoveryController
E0308 06:26:52.956376       1 customresource_discovery_controller.go:155] timed out waiting for caches to sync
I0308 06:26:52.956392       1 naming_controller.go:274] Starting NamingConditionController
E0308 06:26:52.956399       1 cache.go:35] Unable to sync caches for AvailableConditionController controller
I0308 06:26:52.956403       1 naming_controller.go:278] Shutting down NamingConditionController
I0308 06:26:52.956435       1 controller.go:84] Starting OpenAPI AggregationController
I0308 06:26:52.956457       1 controller.go:90] Shutting down OpenAPI AggregationController
I0308 06:26:52.956476       1 crdregistration_controller.go:110] Starting crd-autoregister controller
I0308 06:26:52.956487       1 controller_utils.go:1019] Waiting for caches to sync for crd-autoregister controller
E0308 06:26:52.956504       1 controller_utils.go:1022] Unable to sync caches for crd-autoregister controller
I0308 06:26:52.957364       1 customresource_discovery_controller.go:156] Shutting down DiscoveryController
I0308 06:26:52.958437       1 available_controller.go:266] Shutting down AvailableConditionController
I0308 06:26:52.959508       1 crdregistration_controller.go:115] Shutting down crd-autoregister controller

1234567891011121314151617181920212223
```

发现kube-apiserver容器无法正常启动，怀疑是证书配置过程中出错，重复了如上步骤多次，发现问题依旧。etcd和kubeadm是同一时间部署的，这个时候考虑etcd是否有问题

## etcd证书过期处理

检查etcd的证书，发现etcd同样也过期了：

```
root@yksp009027:/etc/etcd/ssl# openssl x509 -in etcd.pem -noout -text |grep ' Not '
            Not Before: Mar  7 00:59:00 2018 GMT
            Not After : Mar  7 00:59:00 2019 GMT
123
```

查看此前部署的时候配置etcd的ca配置文件，发现指定的证书签署过期时间居然是1年（8760h）。。太坑了！(不清楚证书生成以及etcd安装流程的，可以参考此前的文章：[部署篇](https://blog.csdn.net/ywq935/article/details/80109090))

```
root@009027:~/ssl# cat config.json
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
12345678910111213141516171819
```

居然和kubeadm生产的api-server证书在同一时间过期了，两个坑连在一起，简直防不胜防！

知道问题所在了，那么现在开始替换etcd的证书：

首先备份etcd数据:

```
cd /var/lib
tar -zvcf etcd.tar.gz etcd/
12
```

修改ca配置文件，将默认证书签署过期时间修改为10年:

```
root@009027:~/ssl# cat config.json
{
"signing": {
    "default": {
      "expiry": "87600h"
      },
    "profiles": {
      "kubernetes": {
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
}

123456789101112131415161718192021
```

生成新的证书:

```
#删除过期证书
rm -f /etc/etcd/ssl/*

# 创建新证书
cfssl gencert -ca=ca.pem   -ca-key=ca-key.pem   -config=config.json   -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
cp etcd.pem etcd-key.pem  ca.pem /etc/etcd/ssl/

# 查看新证书
root@009027:/etc/etcd/ssl# openssl x509 -in etcd.pem -noout -text |grep ' Not '
            Not Before: Mar  8 06:59:00 2019 GMT
            Not After : Mar  5 06:59:00 2029 GMT

#拷贝到其他etcd节点
scp -r /etc/etcd/ssl root@${other_node}:/etc/etcd/

# 重启etcd服务(记住，要3个节点一起重启，不然会hang住)
systemctl restart etcd
1234567891011121314151617
```

## 处理结果

etcd证书更换完毕后，再次尝试重启apiserver的容器，启动成功，集群api-server恢复,故障解决。连续2个证书过期，一坑接一坑，卡在apiserver一直无法恢复的时候，一度有些恐慌，担心整个集群由此崩溃数据丢失，还在后面顺利解决。在部署之初没有注意到这些细节问题，例如很明显的是etcd证书的签署ca配置的超时时间居然只有1年，才导致后面才踩了这些坑。平时也要及时备份重要数据，要多多考虑极端情况下的服务恢复方案。