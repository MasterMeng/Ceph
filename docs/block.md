# Ceph块设备  

块是一个字节序列（例如，一个512字节的数据块）。基于块的存储接口是最常见的存储数据的方法，它通常基于旋转介质，像硬盘、CD、软盘，甚至传统的9道磁带。  

# 基本的块设备命令  

rbd命令可以让你创建、罗列、审查和删除块设备image。你也可以用它来克隆image、创建快照、回滚快照、查看快照等。关于rbd命令使用细节，可查看[<font color="red">RBD - Manage RADOS Block Device(RBD) Images</font>](https://docs.ceph.com/docs/master/man/8/rbd/)来了解详情。  

## 创建块设备资源池  

1. 在管理节点，使用ceph工具<font color="red">创建一个资源池</font>。
2. 在管理节点，使用rbd工具RBD来初始化资源池：

> rbd pool init \<pool-name>

## 创建块设备用户  

除非另有说明，否则rbd命令将使用管理员ID访问Ceph集群。此ID允许对集群进行完全的管理访问。建议尽可能使用限制更严格的用户。  

<font color="red">创建Ceph用户</font>，可以使用Ceph命令`auth get-or-create`，需要提供用户名称、监视器和OSD：
   
> ceph auth get-or-create client.{ID} mon 'profile rbd' osd 'profile {profile name} [pool={pool-name}] [, profile ...]'  

例如，创建一个命名称为qemu用户ID，拥有读写权限访问vms资源池，只读images资源池，执行下列命令：  

> ceph auth get-or-create client.qemu mon 'profile rbd' osd 'profile rbd pool=vms,profile rbd-read-only pool=images'

ceph auth get-or-create 命令的输出是新增用户的密钥环，可以写入到*/etc/ceph/cepn.client.{ID}.keyring*文件中。  

## 创建块设备image  

在你将块设备挂载到节点之前，你必须先在Ceph存储集群中创建一个image。可以使用以下命令创建块设备image：  

> rbd create --size {megabytes} {pool-name}/{image-name}

例如，在名称为`swimmingpool`的资源池中创建一个名称为`bar`的1Gimage来存储消息，执行以下命令：  

> rbd create --size 1024 swimmingpool/bar

如果在创建image时你不指明资源池，image将被存储在默认的资源池`rbd`中。例如，在默认的资源池`rbd`中创建一个名称为`foo`的1G大小的image：  

> rbd create --size 1024 foo

## 罗列块设备image  

罗列`rbd`资源池中的块设备image，可以执行以下命令：  

> rbd ls

罗列指定资源池中的块设备image，可以执行以下命令，但需要将`{poolname}`替换为指定的资源池名称：  

> rbd ls {poolname}

例如：  

> rbd ls swimmingpool  

罗列`rbd`资源池回收站中的待删除的块设备，执行以下命令：  

> rbd trash ls

罗列指定资源池回收站中待删除的块设备，可以执行以下命令，但需要将`{poolname}`替换为指定的资源池名称：  

> rbd trash ls {poolname}

例如：  

> rbd trash ls swimming

## 检索image信息   

检索指定image的信息，可以执行以下命令，但需要将`{image-name}`替换为指定的image名称：

> rbd info {image-name}

例如：  

> rbd info foo  

检索指定资源池中image的信息，可以执行以下命令，但需要将`{image-name}`替换成image名称，`{pool-name}`替换成资源池名称：  

> rbd info {pool-name}/{image-name}

例如：  

> rbd info swimmingpool/bar

## 重置块设备image大小  

<font color="red">Ceph块设备</font>是精简配置的。在你开始在其上保存数据之前它们不会占用任何物理存储。但是它们确实具有你使用`--size`操作指定的最大容量。如果你想增加（或者减小）Ceph块设备的最大尺寸，可以执行以下命令：  

> rbd resize --size 2048 foo (to increase)
> rbd resize --size 2048 foo --allow-shrink (to decrease)

## 删除块设备image  

删除块设备image，可以使用以下命令，但需要将`{image-name}`替换为你想要删除的image名称：  

> rbd rm {image-name}

例如：  

> rbd rm foo

删除指定资源池中的块设备，可以执行以下命令，但需要将`{image-name}`替换成你想要删除的image名称，`{pool-name}`替换成资源池名称：  

> rbd rm {pool-name}/{image-name}

例如：  

> rbd rm swimming/bar

要从资源池回收站中移除块设备，可以执行以下命令，但需要将`{image-name}`替换成你想要删除的image名称，`{pool-name}`替换成资源池名称：  

> rbd trash mv {pool-name}/{image-name}

例如：  

> rbd trash mv swimming/bar

要从资源池回收站中删除块设备，可以执行以下命令，但需要将`{image-id}`替换成你想要删除的image的id，`{pool-name}`替换成资源池名称：  

> rbd trash rm {pool-name}/{image-id}

例如：  

> rbd trash rm swimming/2bf4474bodc51

```text
Note:
    · 你可以将image移动到回收站中，即使它拥有快照或其副本正在使用中，但是不能将它从回收站中删除。
    · 你可以使用 --expires-at 来设置滞留时间（默认为当前时间），如果尚未到达滞留时间，除非你使用 --force，否则它不会被删除
```  

## 恢复块设备image  

恢复`rbd`资源池中待删除的块设备，可以使用以下命令，但需要将`{image-id}`替换为imageid：  

> rbd trash restore {image-id}

例如：  

> rbd trash restore 2bf4474b0dc51

恢复指定资源池中待删除的块设备，可以使用以下命令，但需要将`{image-id}`替换为imageid，`{pool-name}`替换为资源池名称：  

> rbd trash restore {pool-name}/{image-id}

例如：  

> rbd trash restore swimming/2bf4474b0dc51  

你也可以恢复image时使用 --image 来重命名称它：  

例如：  

> rbd trash restore swimming/2bf4474b0dc51 --image new-name  

# 块设备操作  

## 快照  

快照是指定时间点image状态的一种只读拷贝。Ceph块设备的一种高级操作是你可以闯将一个image的快照来保存image状态的历史记录。Ceph还支持快照分层，允许你快速轻松地克隆image（例如，VMimage）。Ceph使用rbd命令和更高级的接口，包括<font color="red">QEMU</font>，<font color="red">libvirt</font>，<font color="red">OpenStack</font>和<font color="red">CloudStack</font>，来支持快照。  

### CEPHX 笔记  

当cephx启用（默认启用）时，则你必须指定一个用户名称或ID以及包含有用户对应的密钥的密钥环的路径。详情参见[<font color="red">用户管理</font>](https://docs.ceph.com/docs/master/rados/operations/user-management/#user-management)。你也可以将CEPH_ARGS添加到环境变量中来避免重复输入以下参数：  

> rbd --id {user-ID} --keyring=/path/to/secret [commands]
> rbd --name {username} --keyring=/path/to/secret [commands]  

例如：  

> rbd --id admin --keyring=/etc/ceph/ceph.keyring [commands]
> rbd --name client.admin --keyring=/etc/ceph/ceph.keyring [commands]

### 快照基本操作  

接下来介绍如何在命令行下使用rbd命令来创建、罗列和删除快照。  

#### 创建快照  

使用rbd创建快照，snap create额外选项包括资源池名称和image名称。  

> rbd snap create {pool-name}/{image-name}@{snap-name}

例如：  

> rbd snap create rbd/foo@snamname

#### 罗列快照  

要罗列image的快照，需要额外指定资源池名称和image名称：  

> rbd snap ls {pool-name}/{image-name}

例如：  

> rbd snap ls rbd/foo

#### 回滚快照  

使用rbd回滚快照，snap rollback操作需要额外指明资源池名称、image名称和快照名称：  

> rbd snap rollback {pool-name}/{image-name}@{snap-name}

例如：  

> rbd snap rollback rbd/foo@snapname

#### 删除快照  

使用rbd删除快照，snap rm操作需要额外指明资源池名称、image名称和快照名称：  

> rbd snap rm {pool-name}/{image-name}@{snap-name}

例如：  

> rbd snap rm rbd/foo@snapname

#### 清空快照  

使用rbd删除image的所有快照，snap purge操作需要额外指明资源池名称、image名称：  

> rbd snap purge {pool-name}/{image-name}

例如：  

> rbd snap purge rbd/foo

### 分层  

Ceph支持创建多个块设备快照的写时复制克隆。快照分层可以使Ceph块设备客户端快速地创建image。例如，你可以创建一个写有Linux VM的块设备image；然后，快照image，保护快照，创建任意数量的写时复制克隆体。因为快照是只读的，克隆快照简化了语义，使得快速创建克隆体成为可能。  

![snapshot clone](../images/snapshot_clone.png)  

每个克隆的image（子）都存储着父image的引用，这使得克隆的image可以打开父快照并读取它。  

快照的写时复制克隆行为跟其它的Ceph块设备image一样。你可以读、写、克隆和重设克隆的image的大小。克隆的image没有特殊的限制。然而，快照的写时复制克隆指的是快照，所以在你克隆快照之前你必须保护它。下面的示意图描述的就是这个过程。  

#### 分层入门  

Ceph块设备分层是个简单的操作。你必须有一个image，必须创建这个image的快照，必须保护这个快照。一旦已完成了这些步骤，你就可以开始克隆快照了。  

![clone a snapshot](../images/clone_a_snapshot.png)

克隆的image保留了父快照的引用，并包含了资源池ID、imageID和快照ID。包含资源池ID意味着可以将快照从一个资源池克隆到另一个资源池中的image。  

1. **Image Template**：块设备分层的一个常见的使用场景是创建一个主image和一个快照，作为克隆的模板。例如，用户可能创建一个Linux发行版（例如，Ubuntu 12.04）的image，并为它创建快照。用户可能会周期性地更新image并创建新的快照（例如，`sudo apt-get update`，`sudo apt-get upgrade`，`sudo apt-get dist-upgrade`之后使用`rbd snap create`创建新的快照）。随着image的完善，用户可以克隆快照中的任何一个。


2. **Extended Template**：更高级的使用场景是包含扩展模板image，提供比基础image更多的信息。例如，用户可能克隆一个image（例如，VM模板）并且安装其它的软件（例如，数据库，内容管理系统，分析系统），然后快照扩展后的image，它本身可以像基础image一样被更新。  


3. **Template Pool**：使用块设备分层的一个方法是创建一个资源池，其中包括作为模板的主image以及那些模板的快照。然后你可以将制度权限扩展到其他用户，这样他们就可以克隆快照，但不能在资源池中写入和执行。  


4. **Image Migration/Recovery**：使用块设备分层的一个方法从一个资源池迁移或恢复到另一个资源池。  

#### 保护快照  

克隆体可以访问父快照。如果用户不小心删除了父快照，那么所有的克隆体都将损坏。为了防止数据丢失，在你克隆快照前你**必须**保护它。  

> rbd snap protect {pool-name}/{image-name}@{snapshot-name}  

例如：  

> rbd snap protect rbd/my-image@my-snapshot  

#### 克隆快照  

要克隆快照，你需要额外说明的信息有父资源池、image和快照，以及子资源池和image名称称。在你克隆快照之前你**必须**先保护它。

> rbd clone {pool-name}/{parent-image}@{snap-name} {pool-name}/{child-image-name}

例如：  

> rbd clone rbd/my-image@my-snapshot rbd/new-image

#### 解保快照  

在你删除快照之前，你必选先接触保护。另外，你**不能**删除具有克隆体引用的快照。在你删除快照之前，你**必须**平整该快照的所有克隆。

> rbd snap unprotect {pool-name}/{image-name}@{snapshot-name}

例如：  

> rbd snap unprotect rbd/my-image@my-snapshot

#### 罗列快照的子快照  

要罗列一个快照的子快照，可以执行以下命令：  

> rbd children {pool-name}/{image-name}@{snapshot-name}

例如：  

> rbd childre rbd/my-image@my-snapshot  

#### 平整克隆的image  

克隆的image保留了到父快照的引用。当你移除子克隆体到父快照的引用时，通过从快照拷贝信息到克隆体，你可以高效地“平整”image。平整image花费的时间随着快照体积的增大而增大。要想删除快照，你必须先平整它的子image。  

> rbd flatten {pool-name}/{image-name}

例如：  

> rbd flatten rbd/new-image  

## RBD 镜像  

RBDimage可以在两个Ceph集群间异步备份。该能力利用了RBD日志image特性来保证集群间的crash-consistent复制。镜像功能需要在同伴集群中的每一个对应的pool上进行配置，可设定自动备份某个存储池内的所有images或仅备份images的一个特定子集。用rbd命令来配置镜像功能。rbd-mirror守护进程负责从远端集群拉取image的更新，并写入本地集群的对应image中。  

根据复制的需要，RBD镜像可以配置为单向或者双向复制：  

* **单向复制**：当数据仅从主集群镜像到从集群时，rbd-mirror守护进程只运行在从集群。
* **双向复制**：当数据从一个集群上的主映像镜像到另一个集群上的非主映像(反之亦然)时，rd -mirror守护进程在两个集群上运行。  

### 资源池配置  

下面的程序说明如何使用rbd命令执行基本的管理任务来配置镜像功能。镜像功能需要在同伴集群中的每一个对应的pool上进行配置。  

资源池的配置操作应在所有的同伴集群上执行。为了清晰起见，这些过程假设可以从单个主机访问两个集群，分别称为“local”和“remote”。  

有关如何连接到不同Ceph集群的详细信息，请参阅[<font color="red">rbd</font>](https://docs.ceph.com/docs/master/man/8/rbd)手册页。  

#### 启用镜像功能  

使用rbd启用镜像，需要`mirror pool enable`命令，指明资源池名称和镜像模式：  

> rbd mirror pool enable {pool-name} {mode}

镜像模式可以是pool或image：  

* **pool**：当配置为pool模式时，带有日志特性的资源池中的所有image都将被镜像。
* **image**：当配置为image模式时，每个image的镜像功能都需要被[<font color="red">显示启用</font>](https://docs.ceph.com/docs/master/rbd/rbd-mirroring/#enable-image-mirroring)。

例如：  

```bash
$ rbd --cluster local mirror pool enable image-pool pool
$ rbd --cluster remote mirror pool enabel image-pool pool
```  

#### 禁用镜像  

使用rbd禁用镜像，需要额外的`mirror pool disable`命令和资源池名称：  

> rbd mirror pool disable {pool-name}

以这种方式在池上禁用镜像时，对于已明确启用镜像的任何映像（池内），也将禁用镜像。  

例如：  

```bash
$ rbd --cluster local mirror pool disable image-pool
$ rbd --cluster remote mirror pool disable image-pool
```  

#### 新增集群伙伴  

为了让`rbd-mirror`守护进程发现它的伙伴集群，伙伴集群需要被注册到资源池中。使用rbd新增镜像伙伴Ceph集群，需要额外的`mirror pool peer`添加命令、资源池名称和集群说明：  

> rbd mirror pool peer add {pool-name} {client-name}@{cluster-name}

例如：  

```bash
$ rbd --cluster local mirror pool peer add image-pool client.remote@remote
$ rbd --cluster remote mirror pool peer add image-pool client.local@local
```

默认情况下，`rbd-mirror`守护进程需要有访问位于*/etc/ceph/{cluster-name}.conf*的Ceph配置的权限，它提供了伙伴集群的监视器地址；此外还有位于默认或者配置的密钥环检索路径（例如，/etc/ceph/{cluster-name}.{client-name}.keyring）下的密钥环的访问权限。  

另外，伙伴集群的监视器或客户端密钥可以安全地存储在本地Ceph监视器的confi-key存储中。要在添加伙伴镜像时指定伙伴集群的连接属性，请使用`--remote-mon-host`和`--remote-key-file`选项。例如：  

```bash
$ rbd --cluster local mirror pool peer add image-pool client.remote@remote --remote-mon-host 192.168.1.1,192.168.1.2 --remote-key-file < (echo 'AQAeuZdbMMoBChAAcj++/XUxNOLFaWdtTREEsw==')
$ rbd --cluster local mirror pool info image-pool --all
Mode: pool
Peers:
  UUID                                 NAME   CLIENT        MON_HOST                KEY
  587b08db-3d33-4f32-8af8-421e77abb081 remote client.remote 192.168.1.1,192.168.1.2 AQAeuZdbMMoBChAAcj++/XUxNOLFaWdtTREEsw==
```  

#### 移除伙伴集群  

要使用rbd移除镜像伙伴Ceph集群，需要额外的`mirror pool peer remove`命令、资源池名称和伙伴的UUID（可从`rbd mirror pool info`命令获取）：  

> rbd mirror pool peer remove {pool-name} {peer-uuid}

例如：  

```bash
$ rbd --cluster local mirror pool peer remove image-pool 55672766-c02b-4729-8567-f13a66893445
$ rbd --cluster remote mirror pool peer remove image-pool 60c0e299-b38f-4234-91f6-eed0a367be08
```  

#### 数据池  

在目标集群上创建images时，`rbd-mirror`收集如下数据池：  

1. 如果目标集群有配置好的默认数据池（`rbd_default_data_pool`配置选项），那么这个数据池会被使用。
2. 否则，如果源image使用单独的数据池，且目标集群上存在同名称的数据池，则使用该池。
3. 如果以上都不成立，则不会设置数据池  

### Image 配置  

与资源池配置不同，image配置只需针对单个镜像的伙伴集群进行操作。  

镜像RBD image被指定为主image或非主image。这是image的属性而不是池的属性。被指定为非主image的image不可修改。  

在image上第一次启用镜像时，image会自动提升为主image（如果镜像模式为池模式，且image启用了日志image特性，则会隐式地将image提升为主image，或者通过rbd命令[显式启用](https://docs.ceph.com/docs/master/rbd/rbd-mirroring/#enable-image-mirroring)镜像）。  

#### 启用image日志支持  

RBD镜像使用RBD日志特性来确保复制的image总是保持crash-consistent。在image可以被镜像到伙伴集群之前，日志特性必须启用。改特性可以在image创建时通过使用rbd命令，提供`--image-feature exclusive-lock,journaling`选项来启用。  

或者，日志特性可以在预先存在的RBD images上启用。要使用rbd启用日志，需要额外的`feature enable`命令、资源池名称、image名称和特性名称：  

> rbd feature enable {pool-name}/{image-name} {feature-name}

例如：  

> $ rbd --cluster local feature enable image-pool/image-1 journaling

#### 启用image镜像  

如果镜像是以image池的image模式配置的，则需要显式地为池中的每个image启用镜像。要使用rbd为特定的image启用镜像，需要额外的`mirror image enable`命令、资源池名称和image名称：  

> rbd mirror image enable {pool-name}/{image-name}

例如：  

> $ rbd --cluster local mirror image enable image-pool/image-1

#### 禁用image镜像  

要使用rbd禁用特定的image镜像，需要额外的`mirror image disable`命令、资源池名称和image名称：  

> rbd mirror image disable {pool-name}/{image-name}

例如：  

> $ rbd --cluster local mirror image disable image-pool/image-1

#### image 晋级和降级  

在故障转移场景中，需要将主节点转移到伙伴集群的image中，应该停止对主image的访问（例如，关闭VM或从VM中移除关联的设备），降级当前的主image，提升新的主image，且恢复对备用集群上image的访问。  

要使用rbd将指定的image降级为non-primary，需要额外的`mirror image demote`命令、资源池名称和image名称：  

> rbd mirror image demote {pool-name}/{image-name}

例如：  

> $ rbd --cluster local mirror image demote image-pool/image-1

要是用rbd将池中的所有主image降级为non-primary，需要额外的`mirror pool demote`命令和资源池名称：  

> rbd mirror pool demote {pool-name}

例如：  

> $ rbd --cluster local mirror pool demote image-pool  

要使用rbd将指定的image升级为primary，需要额外的`mirror image promote`命令、资源池名称和image名称：  

> rbd mirror image promote [--force] {pool-name}/{image-name}

例如：  

> $ rbd --cluster remote mirror image promote image-pool/image-1  

要是用rbd将池中的所有的non-primary image升级为primary，需要额外的`mirror pool promote`命令和资源池名称：  

> rbd mirror pool promote [--force] {pool-name}

例如：  

> $ rbd --cluster local mirror pool promote image-pool  

#### 强制image同步  

如果rbd-mirror守护进程检测到有裂脑事件，则在更正之前它不会镜像受到影响的image。要恢复image镜像，首先要降级已过期的image，然后请求同步到主image。要使用rbd请求同步image，需要额外的`mirror image resync`命令，池名称和image名称：  

> rbd mirror image resync {pool-name}/{image-name}

例如：  

> $ rbd mirror image resync image-pool/image-1

### 镜像状态  

伙伴集群的副本状态存储在每个主镜像中。该状态可被`mirror image status`和`mirror pool status`命令检索到。  

要使用rbd命令请求镜像状态，需要额外的`mirror image status`命令，池名称和image名称：  

> rbd mirror image status {pool-name}/{image-nmae}

例如：  

> $ rbd mirror image status image-pool/image-1

要使用rbd命令请求镜像池的总体状态，需要额外的`mirror pool status`命令和池名称：  

> rbd mirror pool status {pool-name}

例如：  

> $ rbd mirror pool status image-pool

### RBD-mirror 守护进程  

两个rbd-mirror守护进程负责监听远端image日志、伙伴集群和重放本地集群的日志事件。RBD image日志特性安装出现的顺序记录下所有对image的修改。这确保了远程镜像的crash-consistent在本地可以。  

每个rbd-mirror守护进程应该使用唯一的Ceph用户ID。要使用ceph创建一个Ceph用户，需要额外的`auth get-or-create`命令，用户名、监视器上限和OSD上限：  

> ceph auth get-or-create client.rbd-mirror.{unique id} mon 'profile rbd-mirror' osd 'profile rbd'

通过指定用户ID作为守护进程实例，systemd可以管理rbd-mirror守护进程：  

> systemctl enable ceph-rbd-mirror@rbd-mirror.{unique id}

rbd-mirror也可以在前台通过rbd-mirror命令启动：  

> rbd-mirror -f --log-file={log_path}  

## image实时迁移  

RBD images可以在同一集群的不同资源池或不同image格式和布局之间实时迁移。