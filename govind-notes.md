# 无状态 vs 有状态

## 无状态服务（stateless service）
对单次请求的处理，不依赖其他请求，也就是说，处理一次请求所需的全部信息，要么都包含在这个请求里，要么可以从外部获取到（比如说数据库），服务器本身不存储任何信息 有状态服务（stateful service）则相反，它会在自身保存一些数据，先后的请求是有关联的 二、优劣 有状态服务常常用于实现事务（并不是唯一办法，下文有另外的方案）。

## 有状态服务
常用于实现事务（并不是唯一办法，下文有另外的方案）。举一个常见的例子，在商城里购买一件商品。需要经过放入购物车、确认订单、付款等多个步骤。由于HTTP协议本身是无状态的，所以为了实现有状态服务，就需要通过一些额外的方案。比如最常见的session，将用户挑选的商品（购物车），保存到session中，当付款的时候，再从购物车里取出商品信息。

经常听到一种说法，即server要设计为无状态的，这主要是从可伸缩性来考虑的。如果server是无状态的，那么对于客户端来说，就可以将请求发送到任意一台server上，然后就可以通过负载均衡等手段，实现水平扩展。如果server是有状态的，那么就无法很容易地实现了，因为客户端需要始终把请求发到同一台server才行，所谓“session迁移”等方案，也就是为了解决这个问题。

## 无状态实现事务的方法
用session很容易实现这个需求，server只需要将第一次提交的数据，保存在session里，然后返回第2个表单作为相应；然后取出第一次提交的数据，和第二次提交的数据汇聚以后，一起存入数据库即可。

不用session同样也可以实现，server接收到第一次请求以后，将数据作为隐藏元素，放在第2个表单里返回；这样用户第2次提交的时候，就隐含地再次提交了第一次的数据；server将所有数据存入数据库
用HTML5，则还可以进一步优化，client可以将第一次提交的数据，保存在sessionStorage里
用cookie也是类似的道理，同样可以实现，但是不太好。

总的来说，3种替代方案（隐藏表单元素、sessionStorage、cookie）都避免了在server端暂存数据，从而实现了stateless service。本质上，这3种方案的请求里，都包含了所有必须的数据。

## 将有状态服务转换成无状态服务
除了将所有信息都放在请求里之外，还有另外一种方法可以实现无状态服务，即将信息放在一个单独可共享的地方，独立于server存在 比如，同样还是采取session的方式，在服务端保存数据，减少每次client请求传输的数据量（节省流量）；但是将session集中存放，比如放在单独的session层里。

# 如何实现配置中心

## 数据结构
在业务需求上，我们应该提前规划好配置中心支持哪些 “配置”，是普通的基础数据类型值，还是像 Json，或者 Yaml ，XML 这样的文件格式，也就是先确定数据结构。

## 存储
元配置数据肯定是需要存储：所以我们需要一个可靠、高性能，可扩展的存储系统来存储和管理配置信息，例如分布式数据库、分布式文件系统。

## 数据隔离
支持多环境配置数据，各个环境之间需要相互隔离，如何隔离？

表中环境字段隔离，部署单个配置中心集群，定义多套环境隔离不同环境的配置数据可以同享配置中心资源。

还是部署多配置中心集群，集群 1 指定定义环境 product，集群 2 指定定义环境 test 等，避免多集群相互影响。

## 客户端和配置中心的通信选型
协议支持上，节点间的通信可以选则比较常用的 HTTP 协议，配置中心将配置数据暴露为 HTTP API，客户端使用 HTTP 访问该 API 以获取配置数据。

或者使用业界成熟的应用通信技术框架，例如 Dubbo，Thrift，gRPC 等。

再或者使用专为网络通信设计的基于 NIO 的高性能框架 Netty，更好的支持高并发，高吞吐的通信场景。

## 支持客户端的接入方式
使用 HTTP 协议实现的接口，天然支持多端，多语言，但可以为例如 Java 语言，Spring 框架提供接入组件。

## 实现配置变更通知机制
当配置数据发生变更时，需要及时通知各个实例。

可以基于轮询机制，客户端定期向配置中心发送请求，查询配置是否变更，若变更主动向配置中心拉取变更数据，优缺点显而易见，优点：简单，缺点：频繁发送请求，性能、网络资源消耗大。

或者基于长连接机制，客户端和配置中心建立长连接，配置数据更新时，配置中心通过长连接实时通知客户端，显而易见的缺点就是，维护长连接的成本对服务器的资源消耗较大。

在或者引入消息队列，配置中心将配置变更信息发布到消息队列，客户端从消息队列中订阅配置变更信息，缺点显而易见，引入消息队列后增加了系统的复杂度。

再或者可以选型 ZooKeeper 或者 Etcd 等作为分布式协调服务，管理配置，通过订阅配置节点上的变更来获取配置更新。

## 安全的考量
为保护配置数据的安全性，需要添加访问控制机制，例如在 API 接口中添加认证、授权等机制来限制访问。

## WEB 端可视化配置管理
增删改查配置直接通过 WEB 界面完成。

## 跨机房，异地多活
配置中心集群关系对等特性，集群各节点提供幂等的配置服务。因此异地跨机房部署时，只需要请求本机房配置中心即可，避免跨机房请求可能遇到的网络疑难杂症。实现异地多活，也可以防止同机房服务全部宕机，访问其他机房可用，达到容灾的效果。

## 配置中心高可用
支持集群部署可提升系统可用性。设计多级存储可有助于降级容灾，
例如一级存储： DB 做元数据存储。
二级存储：配置中心磁盘文件作为配置中心集群的镜像文件。
三级存储：客户端镜像文件，配置中心故障降级使用。
四级存储：接入配置中心的客户端自己的内存 LocalCache 数据
提升性能的同时，降低对底层配置服务的压力。