+++
author = "陈 sir"
categories = ["kubernetes"]
tags = ["kubernetes"]
date = "2019-09-07"
description = "deploy glusterfs and heketi in kubernetes"
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "kubernetes 集群中部署 glusterfs 与 heketi"
type = "post"

+++


# 组件介绍
## Heketi
Heketi提供了一个RESTful管理界面，可以用来管理GlusterFS卷的生命周期。 通过Heketi，就可以像使用OpenStack Manila，Kubernetes和OpenShift一样申请可以动态配置GlusterFS卷。Heketi会动态在集群内选择bricks构建所需的volumes，这样以确保数据的副本会分散到集群不同的故障域内。同时Heketi还支持任意数量的ClusterFS集群，以保证接入的云服务器不局限于单个GlusterFS集群。

## Gluster-Kubernetes
[Gluster-Kubernetes](https://github.com/gluster/gluster-kubernetes)是一个可以将GluserFS和Hekiti轻松部署到Kubernetes集群的开源项目。另外也提供在Kubernetes中可以采用StorageClass来动态管理GlusterFS卷。

## 部署环境

服务器分配信息:

| Hostname | 服务器IP | 存储IP | 硬盘信息 | 容量G |
| ------ | ------ | ------ | ------ | ------ |
| master1 | 192.168.10.51 | 192.168.10.51 | /dev/sdb | 300G|
| node1 | 192.168.10.52 | 192.168.10.52 | /dev/sdb | 300G|
| node2 | 192.168.10.53 | 192.168.10.53 | /dev/sdb | 300G|


## 部署步骤
安装依赖组件 （master1 node1 node2）
``` shell
yum install -y centos-release-gluster
yum install -y glusterfs glusterfs-server glusterfs-fuse glusterfs-rdma glusterfs-geo-replication glusterfs-devel
systemctl start glusterd
systemctl enable glusterd
```

内核模块加载 （master1 node1 node2）
``` shell
# 加载 glusterfs 所需的内核模块
touch /etc/sysconfig/modules/glusterfs.modules
echo '#!/bin/bash' >> /etc/sysconfig/modules/glusterfs.modules
echo "modprobe -- dm_snapshot" >> /etc/sysconfig/modules/glusterfs.modules
echo "modprobe -- dm_mirror" >> /etc/sysconfig/modules/glusterfs.modules
echo "modprobe -- dm_thin_pool" >> /etc/sysconfig/modules/glusterfs.modules
# 授权
chmod 755 /etc/sysconfig/modules/glusterfs.modules 

# 加载模块
bash /etc/sysconfig/modules/glusterfs.modules
```
校验模块加载

``` shell
$lsmod |egrep "dm_snapshot|dm_mirror|dm_thin_pool"
dm_thin_pool           66298  0 
dm_persistent_data     75269  1 dm_thin_pool
dm_bio_prison          18209  1 dm_thin_pool
dm_snapshot            39104  0 
dm_bufio               28014  2 dm_persistent_data,dm_snapshot
dm_mirror              22289  0 
dm_region_hash         20813  1 dm_mirror
dm_log                 18411  2 dm_region_hash,dm_mirror
dm_mod                123941  14 dm_log,dm_mirror,dm_bufio,dm_thin_pool,dm_snapshot
```

获取部署文件 （master1 ）
``` shell
git clone https://github.com/gluster/gluster-kubernetes.git
```

修改配置文件 （master1 ）
``` shell
cp gluster-kubernetes/deploy/topology.json.sample gluster-kubernetes/deploy/topology.json
```
将 topology.json 修改为如下内容，注意 manage 的值必须和kubectl get node 中的的名称一样。
``` json
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "master1"
              ],
              "storage": [
                "192.168.10.50"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "node1"
              ],
              "storage": [
                "192.168.10.51"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "node2"
              ],
              "storage": [
                "192.168.10.52"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        }
      ]
    }
  ]
}
```
修改daemonset 文件

```
---
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: glusterfs
  labels:
    glusterfs: daemonset
  annotations:
    description: GlusterFS DaemonSet
    tags: glusterfs
spec:
  template:
    metadata:
      name: glusterfs
      labels:
        glusterfs: pod
        glusterfs-node: pod
    spec:
      nodeSelector:
        storagenode: glusterfs
      hostNetwork: true
      tolerations: #因为master 节点不能被调度，所以此处要添加容忍
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
      containers:
      - image: gluster/gluster-centos:latest
        imagePullPolicy: IfNotPresent
        name: glusterfs
        ............
```

修改 heketi-deployment.yaml

``` yaml
---
kind: Service
apiVersion: v1
metadata:
  name: heketi
  labels:
    glusterfs: heketi-service
    heketi: service
  annotations:
    description: Exposes Heketi Service
spec:
  selector:
    glusterfs: heketi-pod
  ports:
  - name: heketi
    port: 8080
    targetPort: 8080
  type: NodePort # 添加nodeport 如果想用ingress，可以不用修改此文件，略过这一步
```
修改 gk-deploy 文件（官方bug），在文件中搜索 --show-all
将
``` shell
  heketi_pod=$(${CLI} get pod --no-headers --show-all --selector="heketi" | awk '{print $1}')
```
改为
``` shell
if [[ "${CLI}" =~ ^kubectl ]]; then
  heketi_pod=$(${CLI} get pod --no-headers --selector="heketi" | awk '{print $1}')
else 
  heketi_pod=$(${CLI} get pod --no-headers --show-all --selector="heketi" | awk '{print $1}')
fi

```
修改 heketi.json.template, 主要添加admin key 与 user key

``` json
{
	"_port_comment": "Heketi Server Port Number",
	"port" : "8080",

	"_use_auth": "Enable JWT authorization. Please enable for deployment",
	"use_auth" : false,

	"_jwt" : "Private keys for access",
	"jwt" : {
		"_admin" : "Admin has access to all APIs",
		"admin" : {
			"key" : "openstack"
		},
		"_user" : "User only has access to /volumes endpoint",
		"user" : {
			"key" : "openstack"
		}
	},

	"_glusterfs_comment": "GlusterFS Configuration",
	"glusterfs" : {

		"_executor_comment": "Execute plugin. Possible choices: mock, kubernetes, ssh",
		"executor" : "${HEKETI_EXECUTOR}",

		"_db_comment": "Database file name",
		"db" : "/var/lib/heketi/heketi.db",

		"kubeexec" : {
			"rebalance_on_expansion": true
		},

		"sshexec" : {
			"rebalance_on_expansion": true,
			"keyfile" : "/etc/heketi/private_key",
			"port" : "${SSH_PORT}",
			"user" : "${SSH_USER}",
			"sudo" : ${SSH_SUDO}
		}
	},

	"backup_db_to_kube_secret": false
}
```


开始部署
``` shell
./gk-deploy -g -n glusterfs -c kubectl --admin-key openstack --user-key openstack
```



# 验证安装

### 创建 storage class

创建storageclass.yaml 

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs-storage
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: HEKETI_URL
  clusterid: CLUSTERID
  restauthenabled: "true"
  restuser: "admin" # 此处需要与heketi 实际配置匹配
  # secretNamespace: "default"
  # secretName: "heketi-secret"
  #如果使用 secret 可以用如下命令新建对应的secret 
  # kubectl create secret generic heketi-secret \
  # --type="kubernetes.io/glusterfs" --from-literal=key='openstack' \
  # --namespace=default
  # 对应的yaml为 https://github.com/kubernetes/examples/blob/master/staging/persistent-volume-provisioning/glusterfs/glusterfs-secret.yaml
  restuserkey: "openstack" # 此处需要与heketi 实际配置匹配
  gidMin: "40000" #存储类的 GID 范围的最小值和最大值。此范围内的唯一值 （GID）将用于动态预配卷。这些是可选值。如果未指定，则卷将预配值介于 2000-2147483647 之间，该值分别默认为 gidMin 和 gidMax。
  gidMax: "50000"
  # volumetype: "replicate:3" #Replica volume
  # volumetype: disperse:4:2  #Disperse/EC volume
  # volumetype: none  #Distribute volume
```



然后应用

``` shell
kubectl apply -f storageclass.yaml
```



### 创建 PVC

pvc.yaml

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: glusterfs-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: glusterfs-storage  #storage-class的名字
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```



应用

``` shell
kubectl apply -f pvc.yaml
kubectl get pvc,pv
```



### 创建一个pod 使用pvc

nginx-test.yaml

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
  labels:
    name: nginx-test
spec:
  containers:
  - name: nginx-test
    image: nginx
    ports:
    - name: web
      containerPort: 80
    volumeMounts:
    - name: gluster-vol1
      mountPath: /usr/share/nginx/html
  volumes:
    - name: gluster-vol1
      persistentVolumeClaim:
        claimName: glusterfs-claim
```

``` shell
kubectl apply -f nginx-test.yaml
# 在nginx 中添加内容，然后访问
kubectl exec -it nginx-test bash
cd /usr/share/nginx/html
echo "hellow gluster fs " >> index.html
ls
exit
kubectl forward nginx-test 1880:80
curl localhost:1880
```





## 安装失败问题

### 裸磁盘创建 device 失败
解决方法

1 先卸载安装
``` shell
./gk-deploy -g --abort
```
2 清除 vg
``` shell
[root@master1 deploy]# dmsetup ls
vg_d2ebbc4fc88ce69037db2da2b0a533f7-brick_90a958336d0e5b54b6f5643487805938      (253:6)
vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938-tpool   (253:4)
vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938_tdata   (253:3)
vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938_tmeta   (253:2)
vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938 (253:5)
# 注意顺序不能错，必须先brick 然后，不带任何字符的，然后 tpool ，tdata tmeta
$dmsetup remove vg_d2ebbc4fc88ce69037db2da2b0a533f7-brick_90a958336d0e5b54b6f5643487805938

$dmsetup remove vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938

$dmsetup remove vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938-tpool

$dmsetup remove vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938_tdata

$dmsetup remove vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938_tmeta

```

3 删除 gluster 相关文件
``` shell
rm -rf /var/lib/glusterd
```


4 重置裸盘
``` shell
$dd if=/dev/zero of=/dev/sdb bs=1k count=1
blockdev --rereadpt /dev/sdb
```
5 重启

