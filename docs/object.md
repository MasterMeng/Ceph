# Ceph Object GateWay  

[<font color="red">Ceph Object Gateway</font>](https://docs.ceph.com/docs/master/glossary/#term-ceph-object-gateway)是基于librados构建的对象存储接口，为Ceph存储集群提供RESTful网关服务。[<font color="red">Ceph Object Gateway</font>](https://docs.ceph.com/docs/master/glossary/#term-ceph-object-gateway)支持两种接口：  

1. **S3-compatible**：提供对象存储功能，兼容大部分Amazon S3 RESTful API接口。
2. **Swift-compatible**：提供对象存储功能，兼容大部分OpenStack Swift API接口。  

Ceph对象存储使用Ceph Object GateWay守护进程，它是一个HTTP服务，用来与Ceph存储集群通信。由于Ceph对象网关提供了兼容OpenStack Swift和Amazon S3的接口，所以它有自己的用户管理。Ceph对象网关可以存储来自同一Ceph存储集群的Ceph文件系统客户端或Ceph块设备客户端的数据。S3和Swift APIs共享一套通用的命名空间，所以你可以使用一种API来写入数据，用另一种来检索它们。  

![Ceph Object Gateway](../images/Ceph_Object_Gateway.png)  

> **Note**：Ceph对象存储并不需要Ceph元数据服务。

## 安装Ceph对象网关  

从*firefly*（v0.80）开始，Ceph对象网关运行在Civetweb（被嵌入到ceph-radosgw守护进程中）上，而不是Apache和FastCGI。使用Civetweb简化了Ceph对象网关的安装和配置。  

> **Note**：要运行Ceph对象网关服务，你应该拥有一个运行中的Ceph存储集群，网关服务器应该有权访问公共网络。

### 执行安装前的准备  

查看[<font color="red">此处</font>](https://docs.ceph.com/docs/master/start/quick-start-preflight)并在你的Ceph对象网关节点上执行预安装操作。额外说明，在你的Ceph Deploy用户上，你应该禁用*requiretty*，设置SELinux为*Permissive*，并设置Ceph Deploy用户为无密码的sudo用户。就Ceph对象网关而言，在生产环境中你应该开放Civetweb使用的端口。  

> **Note**：Civetweb默认使用7480端口。  

### 安装Ceph对象网关  

从你的管理员服务的工作目录安装Ceph对象网关包到Ceph对象网关节点。例如：  

> ceph-deploy install --rgw \<gateway-node1> [\<gateway-node2> ...]  

`ceph-common`包是依赖项，所以ceph-deploy也将安装它。ceph CLI工具是管理员需要的。要使你的Ceph对象网关成为管理员节点，需要在你的管理员服务的工作目录下执行以下命令：  

> ceph-deploy admin \<node-name>

### 创建网关实例  

从你的管理员服务的工作目录，在Ceph对象网关节点上创建Ceph对象网关实例。例如：  

> ceph-deploy rgw create \<gateway-node1>

网关运行起来后，像下面这样，你可以使用无认证的请求在7489端口上访问到：  

> http://client-node:7480  

如果网关实例正确运行，你将得到如下响应：  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
       <Owner>
               <ID>anonymous</ID>
               <DisplayName></DisplayName>
       </Owner>
       <Buckets>
       </Buckets>
</ListAllMyBucketsResult>
```  

如果你的节点遇到问题或你想重新开始，执行以下操作清除配置：  

> ceph-deploy purge \<gateway-node1> [\<gateway-node2>]
> ceph-deploy purgedata \<gateway-node1> [\<gateway-node2>]

如果你执行了purge操作，那么你必须重新安装Ceph。  

### 修改默认端口  

默认情况下Civetweb运行在7480端口上。要改变默认端口（例如，改到80端口），修改你管理员服务的工作目录下的Ceph配置文件。增加一节条目[client.rgw.\<gateway-node>]，使用你的Ceph对象网关节点的简写名称（即，hostname -s得到的）替换\<gateway-node>。  

> **Note**：从v11.0.1开始，Ceph对象网关**支持**SSL。如何设置详见[<font color="red">Civetweb使用SSL</font>](https://docs.ceph.com/docs/master/install/install-ceph-gateway/#id2)。  

例如，如果你的节点名称为gateway-node1，在[global]章节下添加：  

```text
[client.rgw.gateway-node1]
rgw_frontends = "civetweb port=80"
```  

> **Note**：确保在*rgw_frontends*键值对中`port=<port-number>`之间没有留下空格。[client.rgw.gateway-node1]头将Ceph配置文件中的这一部分标识为配置Ceph存储集群客户端，其中客户端的类型是Ceph对象网关（即，rgw），实例的名称为gateway-node1。  

将更新的配置文件推送到你的Ceph对象网关节点（和其它节点）：  

> ceph-deploy --overwrite-conf config push \<gateway-node> [\<other-nodes>]  

要使新设置的端口生效，需要重启Ceph对象网关：  

> sudo systemctl restart ceph-radosgw  

最后，检查你所选的端口在防火墙上是否开放（例如，80端口）。如果没有开放，添加端口并重新导入防火墙配置。如果你使用的是`firewalld`，执行：  

> sudo firewalld-cmd --list-all
> sudo firewalld-cmd --zone-public --add-port 80/tcp --permanent
> sudo firewalld-cmd --reload  

如果你使用的是`iptables`，执行：  

> sudo iptables --list
> sudo iptables -I INPUT l -i \<iface> -p tcp -s \<ip-address>/\<netmask> --dprot 80 -j ACCEPT

使用你的Ceph对象网关节点的相关信息替换\<iface>和\<ip-address>/\<netmask>。  

完成iptables的配置之后，你必须确保你的修改是持续不变的，这样在重启你的Ceph对象网关节点之后，修改也会生效。执行：  

> sudo apt-get install iptables-persistent  

这将会打开一个UI终端。选择`yes`选项将当前的IPv4配置规则保存到`/etc/iptables/rules.v4`中，将当前的IPv6配置规则保存到`/etc/iptables/rules.v6`中。  

你上一步设置的IPv4配置规则将会导入到`/etc/iptables/rules.v4`中，即使重启之后也会生效。  

如果你在安装iptables-persistent之后添加新的IPv4规则，你必须将它添加到规则文件中。在这种情况下，使用root用户执行下列操作：  

> iptables-save > /etc/iptables/rules.v4

### Civetweb使用SSL  

在Civetweb使用SSL之前，你需要一个匹配主机名的证书，它将用来访问Ceph对象网关。为了更灵活你可能需要一个有*subject alternate name*字段的证书。如果你想要使用S3风格的子域（[<font color="red">添加通配符到DNS</font>](https://docs.ceph.com/docs/master/install/install-ceph-gateway/#id3)），你需要一个*通配符*证书。  

Civetweb要求在一个文件中提供服务密钥，服务证书，其它CA或中间证书。上述的每一项都必须是*pem*格式的。因为包含了服务密钥，所以这个组合文件应该避免非认证访问。  

要配置ssl选项，需要在端口后追加s。例如：  

```text
[client.rgw.gateway-node1]
rgw_frontends = civetweb prot=443s ssl_certificate=/etc/ceph/private/keyandcert.pem
```  

以下是在Luminous版中新增的。  

Civetweb可以绑定到多个端口，在配置文件中用 **\+** 分隔。适用于单个rgw实例中同时使用ssl和non-ssl连接的使用场景。例如：  

```text
[client.rgw.gateway-node1]
rgw_frontends = civetweb prot=80+443s ssl_certificate=/etc/ceph/private/keyandcert.pem
```  

### 额外的Civetweb配置选项  

可以在ceph.conf文件中的**Ceph Object Gateway**章节部分为嵌入的Civetweb web服务器调整一些额外的配置选项。支持的选项列表，包括示例，详见[<font color="red">HTTP Forntends</font>](https://docs.ceph.com/docs/master/radosgw/frontends)。  

### 从Apache迁移到Civetweb  

如果你使用v0.80或之前的Ceph存储在Apache和FastCGI上运行Ceph对象网关，所以你已经使用ceph-radosgw守护进程运行Civetweb-it，默认情况下它运行在7480端口，所以它不会与你的Apache和FastCGI安装或其它常用web服务端口冲突。迁移到使用Civetweb基本上需要删除你的Apache安装。这样，你必须从你的Ceph配置文件中移除Apache和FastCGI，然后重新设置rgw_frontends为Civetweb。  

返回使用ceph-deploy安装Ceph对象网关说明，注意配置文件中只有一条rgw_frontends设置（并且假定你选择修改默认的端口）。ceph-deploy工具创建数据目录和密钥环，密钥环放在`/var/lib/ceph/radosgw/{rgw-intance}`目录。守护进程可在默认位置找到，你可能在你的Ceph配置文件中指定了不同的目录。此时你已经准备好了密钥和数据目，如果你使用默认路径意外的位置，你需要在你的Ceph配置文件中声明这些路径。  

典型的基于Apache部署的Ceph对象网关配置文件类似于下面的内容：  

在Red Hat Enterprise Linux：  

```text
[client.radosgw.gateway-node1]
host = {hostname}
keyring = /etc/ceph/ceph.client.radosgw.keyring
rgw socket path = ""
log file = /var/log/radosgw/client.radosgw.gateway-node1.log
rgw frontends = fastcgi socket\_port=9000 socket\_host=0.0.0.0
rgw print continue = false
```

在Ubuntu：  

```text
[client.radosgw.gateway-node]
host = {hostname}
keyring = /etc/ceph/ceph.client.radosgw.keyring
rgw socket path = /var/run/ceph/ceph.radosgw.gateway.fastcgi.sock
log file = /var/log/radosgw/client.radosgw.gateway-node1.log
```  

要修改它来使用Civetweb，只需要移除Apache特定的设置，比如rgw_socket_path和rgw_print_continue。然后，修改rgw_frontends设置，使用Civetweb取代Apache FastCGI和指定你选择使用的端口。例如：  

```text
[client.radosgw.gateway-node1]
host = {hostname}
keyring = /etc/ceph/ceph.client.radosgw.keyring
log file = /var/log/radosgw/client.radosgw.gateway-node1.log
rgw_frontends = civetweb port=80
```  

最后，重启Ceph对象网关。在Red Hat Enterprise Linux上执行：  

> sudo systemctl restart ceph-radosgw.service

Ubuntu上执行：  

> sudo service radosgw restart id=rgw.\<short-hostname>

如果你使用的端口为开放，你需要在你的防火墙上开放端口。  

### 配置桶分片  

Ceph对象网关在index_pool中存储桶索引数据，默认为`.rgw.buckets.index`。有时，用户喜欢在一个桶中放入很多对象（成百上万的对象）。如果不使用网关管理界面为每个存储桶设置最大对象数的配额，那么当用户将大量对象放入存储桶时，存储桶索引会遭受严重的性能下降。  

在Ceph v0.94，你可以对存储桶索引进行分片，以防止允许单个存储桶存储大量对象时出现性能瓶颈。`rgw_overrode_bucket_max_shards`设置允许你设置每个存储桶的最大分片数量。其默认值为0，意味着默认关闭存储桶索引分片。  

要启用存储桶索引分片，只需设置`rgw_overrode_bucket_max_shards`大于0即可。  

为了简化配置，你可以在你的Ceph配置文件中添加`rgw_overrode_bucket_max_shards`。在[global]下添加它来创建系统级别的变量。你也可以在你的Ceph配置文件中未每一个实例设置它。  

一旦在你的Ceph配置文件中改变了你的存储桶分片配置，你需要重启你的网关。在Red Hat Enterprise Linux上执行：  

> sudo systemctl restart ceph-radosgw.service

在Ubuntu上：  

> sudo service radosgw restart id=rgw.\<short-hostname>

对于联合配置，每个区域对于故障转移可能有不同的index_pool设置。要使zonegroup区域的值保持一致，你可以在网关的zonegroup配置中设置`rgw_override_bucket_index_max_shards`。例如：  

> radosgw-admin zonegroup get > zonegroup.json

打开zonegroup.json文件，为每个命名的区域设置`bucket_index_max_shards`。保存文件，重置zonegroup。例如：  

> radosgw-admin zonegroup set < zonegroup.json  

一旦你更新了你的zonegroup，更新并提交。例如：  

> radosgw-admin period update --commit  

> **Note**：将索引池（对于每个区域，如果适用）映射到基于SSD的OSD的CRUSH规则也可以有助于桶索引性能。

### 添加通配符到DNS  

要将Ceph用在S3风格的子域（例如，bucket-name.domain-name.com）中，你需要将通配符添加到与ceph-radosgw守护进程一起使用的DNS服务器的DNS记录中。  

DNS的地址必须在Ceph配置文件中用`rgw dns name = {hostname}`指明设置。  

对于dnsmasq，条件如下在主机名前加上“.”做前缀的地址设置：  

> address=/.{hostname-or-fqdn}/{host-ip-address}

例如：  

> address=/.gateway-node1/192.168.122.75

对于bind，添加通配符到DNS记录。例如：  

```bash
$TTL    604800
@       IN      SOA     gateway-node1. root.gateway-node1. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      gateway-node1.
@       IN      A       192.168.122.113
*       IN      CNAME   @
```  

重启你的DNS服务，并使用子域来ping你的服务，来确保你的DNS配置如预期一样工作：  

> ping mybucket.{hostname}

例如：  

> ping mybucket.gateway-node1

### 添加调试（必要时）  

