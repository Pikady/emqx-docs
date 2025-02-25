# EMQX 集群连接

::: tip

集群连接是 EMQX 企业版功能。

:::

集群连接（Cluster Linking）是一项将多个独立的 EMQX 集群连接在一起的功能，方便不同且通常地理位置分散的集群之间的客户端通信。与传统的 MQTT 桥接相比，集群连接在效率、可靠性和可扩展性方面更具优势。它能够最大限度地减少带宽需求，并且可以容忍网络中断。

本节介绍了集群连接的概念及其使用和配置方法。

## 为什么使用集群连接

本节解释了集群连接如何为不同集群之间的客户端通信提供高效的解决方案，并突出其在带宽使用、网络容忍度和可扩展性方面相较于 MQTT 桥接的优势。

### 单集群的挑战

单个 EMQX 集群可以有效地为成千上万个地理位置分散的 MQTT 客户端提供服务。然而，当客户端分布在全球时，高延迟和网络连接不佳的问题会随之而来。通过在不同区域部署多个 EMQX 集群从而服务本地客户端来缓解这些问题。

### 集群连接 vs. MQTT 桥接

在不同区域部署多个 EMQX 集群引入了一个新挑战：如何实现连接到不同集群的客户端之间的无缝通信。

传统解决方案是为每个集群添加一个 MQTT 桥接，该桥接在集群之间转发所有消息。这种方法会导致带宽使用过多，并可能增加消息延迟，因为许多转发的消息可能与桥接另一端的客户端无关。

集群连接通过仅在集群之间转发相关消息来解决这些问题。这种优化减少了带宽使用，并确保即使在网络中断期间也能进行高效通信。

#### 功能对比

集群连接与 MQTT 桥接共享一些功能。以下是两者之间的对比，突出 MQTT 桥接和集群连接的主要区别和相似之处，帮助您理解每个功能的优点和适用场景。

| 功能                   | MQTT 桥接                                                    | 集群连接                                                     |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **协议**               | MQTT                                                         | MQTT                                                         |
| **认证**               | 遵循远程消息服务器                                           | 遵循远程消息服务器                                           |
| **授权**               | 遵循远程消息服务器                                           | 仅需对控制主题进行授权<br />**控制主题**：集群连接功能所需的数据交换主题<br />**消息主题**：需要进行消息复制的主题 |
| **适用产品**           | 任何标准 MQTT 消息服务器                                     | 仅适用于 EMQX                                                |
| **消息环回**           | 需要支持桥接模式                                             | 自动避免                                                     |
| **命名空间**           | 通常需要添加主题前缀以避免消息回环                           | 统一命名空间                                                 |
| **自定义主题前缀**     | 支持                                                         | 不支持                                                       |
| **桥接方法**           | 支持在远程或本地配置桥接规则（发布/订阅）<br />同一连接可作为发布者和订阅者 | 仅支持推送模式，需要本地配置推送到远程。需要对等配置以进行双向通信<br />同一连接只能作为发布者。推送模式比双向模式更高效、更稳定。 |
| **消息复制方法**       | 对指定主题进行完整订阅，效率较低                             | 基于本地订阅列表按需复制                                     |
| **规则引擎和数据集成** | 始终触发消息发布事件                                         | 不会触发本地规则引擎的消息发布事件                           |

## 工作流程和使用场景

本节介绍了集群连接在两种不同场景下的工作流程。

### 从远程集群复制消息到本地集群

在此场景中，集群 A 作为本地集群，集群 B 作为远程集群。集群 A 需要从集群 B 复制主题 `t/#` 和 `c/1`。

1. **客户端认证和授权**：
   - 在集群 A 和集群 B 的客户端认证配置中添加凭证。
   - 在集群 A 和集群 B 的客户端 ACL 中添加授权。
2. **在集群 A 中配置集群连接**：
   - 输入集群 B 的 MQTT 地址。
   - 提供必要的认证信息。
   - 指定要复制的主题：`t/#` 和 `c/1`。
3. **在集群 B 中配置集群连接**：
   - 输入集群 A 的 MQTT 地址。
   - 提供必要的认证信息。
   - 无需指定主题（集群 B 不需要集群 A 的消息）。
4. **建立相互连接**：
   - 确保两个集群彼此连接。
5. **启动路由信息推送（A → B）**：
   - 从集群 A 向集群 B 启动路由信息推送。
6. **订阅和消息流**：
   - 集群 A 中的客户端订阅主题。
   - 路由信息从集群 A 推送到集群 B。
   - 客户端向集群 B 发布消息。
   - 消息从集群 B 推送到集群 A。
7. **取消订阅**：
   - 集群 A 中的客户端取消订阅主题。
   - 删除从集群 A 到集群 B 的路由信息推送。
8. **配置过程完成**。

### 将旧集群迁移到新集群

在此场景中，集群 A 是旧集群，集群 B 是新集群。目标是将所有消息从集群 A 完全复制到集群 B。

1. 在集群 A 和集群 B 之间配置对称连接，在两个集群中配置主题为 `#`。
2. 确保两个集群彼此连接。
3. 实施 DNS 更改或负载均衡器流量切换。
4. 将客户端迁移到新集群。消息不会中断，但会话无法迁移，需要客户端重新订阅。
5. 将集群 A 从网络中断开。
6. 在集群 B 中建立新连接，客户端需要再次发起主动订阅。迁移过程完成。

## 下一步

接下来，您可以通过以下指南学习如何使用集群连接功能并配置其功能：

- [集群连接快速开始](./quick-start.md)
- [配置集群连接](./configuration.md)
