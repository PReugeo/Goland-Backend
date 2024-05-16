## 卷（Volume）

由于 Container 中文件在磁盘上是临时存放的，当容器崩溃时文件会丢失，重启后文件也不会复原。

而且也不利于同一 pod 运行多个容器并共享文件。

### 背景

Docker 中对于卷只有少量且松散的管理，Docker 卷时磁盘上或者另一个容器内的一个目录，K8S 支持很多类型的卷，Pod 可以同时使用任意数目的卷类型。临时卷类型的生命周期与 pod 相同，持久卷则生命周期会比 Pod 存活周期长。

卷是一个目录，卷类型会决定目录如何形成以及使用何种介质保存数据以及目录中存放内容。

`.spec.volumes` 字段设置 pod 提供的卷，并在 `.spec.containers[*].volumeMounts` 字段声明卷在该容器中的挂载位置。

卷不能挂载到其他卷之上，也不能与其他卷有硬链接。 Pod 配置中的每个容器必须独立指定各个卷的挂载位置。

### 存储相关名词解释

- 卷（Volume）：可以在 pod 创建时指定卷为 pod 提供持久存储
- 持久卷 （PV）：集群中的一块存储，可以由管理员事先供应，或者 使用[存储类（Storage Class）](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/)来动态供应
- 持久卷申领（PVC）：表达的是用户对存储的请求。概念上与 Pod 类似。 Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源。
- 动态卷供应（StorageClass）：动态卷供应允许按需创建存储卷。 如果没有动态供应，集群管理员必须手动地联系他们的云或存储提供商来创建新的存储卷， 然后在 Kubernetes 集群创建 [PersistentVolume 对象](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/)来表示这些卷。通过指定 provisioner 配合官方或者社区 csi 可以创建动态卷供应

### 访问模式

- `ReadWriteOnce`

  卷可以被一个节点以读写方式挂载。 ReadWriteOnce 访问模式也允许运行在同一节点上的多个 Pod 访问卷。

- `ReadOnlyMany`

  卷可以被多个节点以只读方式挂载。

- `ReadWriteMany`

  卷可以被多个节点以读写方式挂载。

- `ReadWriteOncePod`

  卷可以被单个 Pod 以读写方式挂载。 如果你想确保整个集群中只有一个 Pod 可以读取或写入该 PVC， 请使用ReadWriteOncePod 访问模式。这只支持 CSI 卷以及需要 Kubernetes 1.22 以上版本。



当前 Kubernetes Volume Plugin 支持的访问模式

| 卷插件               | ReadWriteOnce | ReadOnlyMany | ReadWriteMany                    | ReadWriteOncePod |
| -------------------- | ------------- | ------------ | -------------------------------- | ---------------- |
| AWSElasticBlockStore | ✓             | -            | -                                | -                |
| AzureFile            | ✓             | ✓            | ✓                                | -                |
| AzureDisk            | ✓             | -            | -                                | -                |
| CephFS               | ✓             | ✓            | ✓                                | -                |
| Cinder               | ✓             | -            | -                                | -                |
| CSI                  | 取决于驱动    | 取决于驱动   | 取决于驱动                       | 取决于驱动       |
| FC                   | ✓             | ✓            | -                                | -                |
| FlexVolume           | ✓             | ✓            | 取决于驱动                       | -                |
| Flocker              | ✓             | -            | -                                | -                |
| GCEPersistentDisk    | ✓             | ✓            | -                                | -                |
| Glusterfs            | ✓             | ✓            | ✓                                | -                |
| HostPath             | ✓             | -            | -                                | -                |
| iSCSI                | ✓             | ✓            | -                                | -                |
| Quobyte              | ✓             | ✓            | ✓                                | -                |
| NFS                  | ✓             | ✓            | ✓                                | -                |
| RBD                  | ✓             | ✓            | -                                | -                |
| VsphereVolume        | ✓             | -            | - （Pod 运行于同一节点上时可行） | -                |
| PortworxVolume       | ✓             | -            | ✓                                | -                |
| StorageOS            | ✓             | -            | -                                | -                |

### CSI-容器存储接口

`csi` 卷类型是一种 out-tree（即跟其它存储插件在同一个代码路径下，随 Kubernetes 的代码同时编译的） 的 CSI 卷插件，用于 Pod 与在同一节点上运行的外部 CSI 卷驱动程序交互。部署 CSI 兼容卷驱动后，用户可以使用 `csi` 作为卷类型来挂载驱动提供的存储。

在 K8S 中可以通过 CSI 创建插件 `StorageClass` 支持动态配置的 CSI Storage 插件启用自动创建/删除，通过 StorageClass 中指定 `provisioner` 来指定相关的 CSI Volume Plugin 动态创建 Persistent Volume
