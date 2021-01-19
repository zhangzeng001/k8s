# Deployment

查看pod状态

`kubectl get deployment`

```shell
kubectl get deployment
NAME            RESIRED        CURRENT      UP-TO-DATE     AVAILABLE        AGE
tomcat-deploy      1              1            1               1            4m


# RESIRED: Pod副本适量的期望值，即在Deployment里定义的Replica
# CURRENT: 当前Replica的值，实际上是Deployment创建的Replica Set里的Replica值，这个值不断在增加直到            DESIRED为止
# UP-TO-DATE: 最新版本的Pod 的副本数量，用于只是在滚动升级的过程中，有多少个Pod副本已经成功升级
# AVAILABLE: 当前集群种可用的Pod副本数量，即集群种当前存活的Pod数量。
```

运行下面命令查看Replica Set

`kubectl get rs`

可以看到他的命令与Deployment的名称有关系：

```shell
kubectl get rs
NAME                           RESIRED     CURRENT    AGE
tomcat-deploy-12309125912         1           1        4m
```



















