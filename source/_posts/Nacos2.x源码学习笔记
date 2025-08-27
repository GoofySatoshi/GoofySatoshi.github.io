# Nacos 2.X 核心源码学习笔记

---
title: Nacos 2.X 核心源码学习笔记
date: 2025-08-27 23:11:00
categories: 中间件
tags:
- Nacos
- 源码剖析
- 分布式服务治理
---

## 一、引言

Nacos 作为阿里开源的分布式服务发现与配置管理平台，2.X 版本在架构上进行了协议升级（HTTP→gRPC）、存储结构优化（嵌套 Map→扁平分层）、一致性增强（Distro→Raft）

Nacos 2.X 主要核心优化：
1. 协议升级：gRPC 长连接替代 HTTP，降低通信开销，提升实时性；
2. 存储优化：扁平分层注册表替代嵌套 Map，提升查询与同步效率；
3. 事件驱动：解耦模块依赖，增强扩展性；
4. 健康检查：服务端主动探活替代客户端心跳，减少客户端负担；
5. 一致性增强：Raft+Distro 混合协议，平衡一致性与性能。

## 二、服务注册核心源码剖析

服务注册是 Nacos 的基础能力，2.X 版本通过 gRPC 长连接异步通信替代 1.X 的 HTTP 短连接，同时优化了实例存储与一致性逻辑。

### （一）客户端：发起注册请求

客户端核心是封装实例信息、建立 gRPC 连接并发送注册请求，关键类集中在 `nacos-naming-client` 模块。

1. 入口类：`NacosNamingService`

客户端对外暴露的注册入口，负责参数预处理与代理分发：

```java
// com.alibaba.nacos.client.naming.NacosNamingService
@Override
public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {
    // 1. 封装服务唯一标识：group@@serviceName（Nacos 服务的核心命名规则）
    String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);
    // 2. 补全实例默认值（如集群名默认值、命名空间）
    if (instance.getClusterName() == null) {
        instance.setClusterName(ClusterUtils.getDefaultClusterName());
    }
    if (instance.getNamespaceId() == null) {
        instance.setNamespaceId(this.namespace);
    }
    // 3. 委托 gRPC 代理发送注册请求（核心逻辑）
    namingClientProxy.registerInstance(groupedServiceName, instance);
}
```

2. gRPC 代理：`NamingGrpcClientProxy`

负责将注册请求封装为 gRPC 协议格式，通过长连接发送至服务端：

```java
// com.alibaba.nacos.client.naming.remote.gprc.NamingGrpcClientProxy
@Override
public void registerInstance(String serviceName, Instance instance) throws NacosException {
    // 1. 构建 gRPC 注册请求（ProtoBuf 格式，适配 gRPC 传输）
    RegisterInstanceRequest request = RegisterInstanceRequest.newBuilder()
            .setServiceName(serviceName)
            .setInstance(GrpcUtils.convertInstanceToProto(instance)) // 实例信息转 Proto
            .setNamespace(instance.getNamespaceId())
            .build();
    
    // 2. 同步调用服务端 gRPC 接口（失败重试由底层 GrpcUtils 实现）
    GrpcUtils.invokeGrpcSync(
        () -> namingStub.registerInstance(request), // namingStub 是 gRPC 自动生成的客户端存根
        "registerInstance", 
        serviceName
    );
}
```

- 关键细节：客户端初始化时会与服务端建立 gRPC 长连接（默认端口 9848），后续注册、心跳、订阅均复用该连接，减少 TCP 握手开销。

### （二）服务端：处理注册请求

服务端核心是接收 gRPC 请求、校验参数、分层存储实例，并触发事件通知与集群同步，关键类集中在 `nacos-naming-core` 模块。

1. gRPC 请求入口：`NamingGrpcService`

服务端 gRPC 服务实现类，负责接收客户端请求并转发至业务逻辑层：

```java
// com.alibaba.nacos.naming.controllers.v2.grpc.NamingGrpcService
@Override
public void registerInstance(RegisterInstanceRequest request, StreamObserver<CommonResponse> responseObserver) {
    CommonResponse response = CommonResponse.newBuilder().build();
    try {
        // 1. 解析请求参数（服务名、命名空间、实例信息）
        String serviceName = request.getServiceName();
        String namespaceId = request.getNamespace();
        Instance instance = GrpcUtils.convertProtoToInstance(request.getInstance());
        
        // 2. 委托 ServiceManager 处理注册核心逻辑
        serviceManager.registerInstance(namespaceId, serviceName, instance);
        
        // 3. 返回成功响应
        response.setSuccess(true);
    } catch (Exception e) {
        response.setSuccess(false);
        response.setMessage("注册失败：" + e.getMessage());
        log.error("Instance register failed", e);
    } finally {
        responseObserver.onNext(response);
        responseObserver.onCompleted(); // 结束 gRPC 流
    }
}
```

2. 服务管理核心：`ServiceManager`

负责创建/获取服务、调度实例注册，是服务端的"服务中枢"：

```java
// com.alibaba.nacos.naming.core.ServiceManager
public void registerInstance(String namespaceId, String serviceName, Instance instance) throws NacosException {
    // 1. 生成服务唯一 Key：namespaceId@@group@@serviceName（避免同名冲突）
    String serviceKey = ServiceKeyGenerator.generateServiceKey(namespaceId, serviceName);
    
    // 2. 不存在则创建 Service（懒加载模式，减少内存占用）
    Service service = getOrCreateService(namespaceId, serviceName);
    
    // 3. 将实例注册到对应集群（核心：分层存储的入口）
    service.registerInstance(instance);
    
    // 4. 发布服务注册事件（触发后续健康检查、集群同步）
    eventPublisher.publishEvent(new ServiceInstanceRegisteredEvent(service, instance));
}

// 懒加载创建 Service
private Service getOrCreateService(String namespaceId, String serviceName) {
    String serviceKey = ServiceKeyGenerator.generateServiceKey(namespaceId, serviceName);
    return serviceMap.computeIfAbsent(serviceKey, key -> {
        Service newService = new Service();
        newService.setNamespaceId(namespaceId);
        newService.setServiceName(serviceName);
        newService.setId(IdGenerator.generateServiceId(namespaceId, serviceName));
        return newService;
    });
}
```

3. 分层存储：`Service` → `Cluster` → `Instance`

2.X 采用"服务-集群-实例"三层结构，替代 1.X 的嵌套 Map，每个层级独立管理：

```java
// com.alibaba.nacos.naming.core.entity.Service
public void registerInstance(Instance instance) {
    // 1. 获取/创建实例所属集群（默认集群名：DEFAULT）
    String clusterName = instance.getClusterName();
    Cluster cluster = clusters.computeIfAbsent(clusterName, key -> new Cluster(clusterName, this));
    
    // 2. 将实例注册到集群（集群维护实例集合）
    cluster.addInstance(instance);
}

// com.alibaba.nacos.naming.core.entity.Cluster
public void addInstance(Instance instance) {
    // 实例存储容器：线程安全的 ConcurrentHashMap（key=instanceId，避免重复注册）
    ConcurrentHashMap<String, Instance> instanceMap = getInstanceMap();
    
    // 区分临时实例与持久化实例（核心差异：是否依赖心跳）
    if (instance.isEphemeral()) {
        // 临时实例：内存存储 + 心跳绑定
        instanceMap.put(instance.getInstanceId(), instance);
        // 注册心跳检测任务（服务端主动探活）
        healthCheckReactor.addHealthCheckTask(new GrpcHealthCheckTask(instance, this));
    } else {
        // 持久化实例：内存 + 本地文件持久化（重启不丢失）
        instanceMap.put(instance.getInstanceId(), instance);
        persistInstance(instance); // 持久化到 data/nacos/naming/ 目录
    }
}
```

4. 请求校验：隐式在业务逻辑中

2.X 未单独封装 `InstanceValidator`，而是在实例注册前通过 参数非空校验+格式校验 确保合法性：

```java
// Cluster.addInstance 中隐含的校验逻辑（简化）
if (StringUtils.isEmpty(instance.getIp()) || instance.getPort() <= 0) {
    throw new NacosException(NacosException.INVALID_PARAM, "IP/端口不能为空");
}
if (instance.getWeight() < 0) {
    throw new NacosException(NacosException.INVALID_PARAM, "权重不能为负数");
}
```

## 三、事件驱动架构剖析

Nacos 2.X 基于 "事件发布-订阅"模式 解耦模块依赖，核心由「事件、事件总线、发布者、订阅者」四部分组成，所有核心流程（注册、健康检查、同步）均通过事件触发。

### （一）事件：状态变化的载体

事件是系统状态变化的封装，所有事件均继承自顶层接口 `Event`，核心事件类如下：

| 事件类 | 触发场景 | 核心属性 |
|--------|----------|----------|
| `ServiceInstanceEvent` | 服务实例注册/更新/删除 | `service`（服务）、`instance`（实例） |
| `HealthStateChangeEvent` | 实例健康状态变更（健康→不健康） | `instanceId`、`newHealthyState` |
| `ConfigDataChangeEvent` | 配置中心数据变更 | `dataId`、`group`、`content` |

示例：服务实例注册事件定义

```java
// com.alibaba.nacos.naming.core.v2.event.ServiceInstanceRegisteredEvent
public class ServiceInstanceRegisteredEvent extends AbstractEvent {
    private final Service service;
    private final Instance instance;

    public ServiceInstanceRegisteredEvent(Service service, Instance instance) {
        this.service = service;
        this.instance = instance;
    }

    // getter 省略
}
```

### （二）事件总线：事件路由的核心

事件总线负责连接发布者与订阅者，提供事件注册、发布、路由能力，核心实现类为 `NacosEventBus`：

```java
// com.alibaba.nacos.common.event.NacosEventBus
public class NacosEventBus extends AbstractEventBus {
    // 存储：事件类型 → 订阅者列表（线程安全）
    private final ConcurrentHashMap<Class<? extends Event>, CopyOnWriteArrayList<Subscriber>> subscriberMap = new ConcurrentHashMap<>();

    // 1. 注册订阅者（根据事件类型关联）
    @Override
    public void register(Subscriber subscriber) {
        Class<? extends Event> eventType = subscriber.getSubscribeType();
        subscriberMap.computeIfAbsent(eventType, key -> new CopyOnWriteArrayList<>())
                     .add(subscriber);
    }

    // 2. 发布事件（路由到对应订阅者）
    @Override
    public void publish(Event event) {
        Class<? extends Event> eventType = event.getClass();
        // 找到订阅该事件的所有订阅者
        CopyOnWriteArrayList<Subscriber> subscribers = subscriberMap.get(eventType);
        if (subscribers == null) {
            return;
        }
        // 异步触发订阅者的事件处理（默认线程池：EventExecutor）
        for (Subscriber subscriber : subscribers) {
            EventExecutor.execute(() -> subscriber.onEvent(event));
        }
    }
}
```

### （三）发布者：事件的产生者

发布者在特定业务逻辑完成后发布事件，核心实现是 `EventPublisher` 接口，默认实现为 `DefaultEventPublisher`：

```java
// ServiceManager 中发布注册事件的逻辑（前文已提及）
eventPublisher.publishEvent(new ServiceInstanceRegisteredEvent(service, instance));
```

常见发布场景：
- 服务注册完成 → 发布 `ServiceInstanceRegisteredEvent`
- 实例健康状态变更 → 发布 `HealthStateChangeEvent`
- 配置数据更新 → 发布 `ConfigDataChangeEvent`

### （四）订阅者：事件的消费者

订阅者通过 实现 `Subscriber` 接口 或 添加 `@EventListener` 注解 订阅事件，核心是编写事件处理逻辑。

示例：健康检查订阅者（监听注册事件，初始化探活任务）

```java
// com.alibaba.nacos.naming.healthcheck.v2.subscriber.InstanceRegisteredSubscriber
@Component
public class InstanceRegisteredSubscriber implements Subscriber<ServiceInstanceRegisteredEvent> {

    @Autowired
    private HealthCheckReactor healthCheckReactor;

    // 订阅的事件类型
    @Override
    public Class<? extends Event> getSubscribeType() {
        return ServiceInstanceRegisteredEvent.class;
    }

    // 事件处理逻辑
    @Override
    public void onEvent(ServiceInstanceRegisteredEvent event) {
        Instance instance = event.getInstance();
        Cluster cluster = event.getService().getCluster(instance.getClusterName());
        // 为临时实例添加 gRPC 探活任务
        if (instance.isEphemeral()) {
            healthCheckReactor.addHealthCheckTask(new GrpcHealthCheckTask(instance, cluster));
        }
    }
}
```

- 关键优势：模块间无需硬编码依赖（如注册模块不直接调用健康检查模块），只需通过事件交互，扩展性极强。

## 四、注册表结构变动剖析

Nacos 2.X 对注册表的优化是性能提升的核心，主要解决 1.X 嵌套 Map 带来的 深拷贝开销大、查询效率低 问题。

### （一）旧结构（1.X）：嵌套 Map 的痛点

1.X 注册表采用 四层嵌套 Map，结构如下：

```java
// 1.X 注册表结构（简化）
Map<String, Map<String, Map<String, Map<String, List<Instance>>>>> registry;
// 层级：namespace → group → service → cluster → instanceList
```

- 痛点：
1. 查询实例需逐层遍历（如 `registry.get(namespace).get(group).get(service).get(cluster)`），效率低；
2. 数据同步/备份时需深拷贝整个嵌套结构，内存与时间开销大；
3. 新增维度（如元数据）需修改多层结构，扩展性差。

### （二）新结构（2.X）：扁平分层存储

2.X 将注册表拆分为 独立的分层存储单元，通过 唯一 Key 关联，核心实现类为 `ServiceStorage`：

1. 核心存储结构

```java
// com.alibaba.nacos.naming.core.v2.storage.ServiceStorage
public class ServiceStorage {
    // 单例模式（注册表全局唯一）
    private static final ServiceStorage INSTANCE = new ServiceStorage();
    // 存储：namespace → (serviceKey → ServiceInfo)
    private final ConcurrentHashMap<Namespace, ConcurrentHashMap<Service, ServiceInfo>> namespaceServiceInfoMap = new ConcurrentHashMap<>();

    // 获取服务实例列表（通过 namespace + service 直接定位）
    public ServiceInfo getServiceInfo(Namespace namespace, Service service) {
        ConcurrentHashMap<Service, ServiceInfo> serviceInfoMap = namespaceServiceInfoMap.get(namespace);
        if (serviceInfoMap == null) {
            return null;
        }
        return serviceInfoMap.get(service);
    }

    // 新增/更新服务实例
    public void putServiceInfo(Namespace namespace, Service service, ServiceInfo serviceInfo) {
        namespaceServiceInfoMap.computeIfAbsent(namespace, key -> new ConcurrentHashMap<>())
                               .put(service, serviceInfo);
    }
}
```

2. 联合 Key 设计

通过 `ServiceKeyGenerator` 生成 全局唯一 Key，替代多层嵌套的定位逻辑：

```java
// com.alibaba.nacos.naming.core.ServiceKeyGenerator
public class ServiceKeyGenerator {
    // 生成 serviceKey：namespaceId@@groupName@@serviceName
    public static String generateServiceKey(String namespaceId, String serviceName) {
        // 从 serviceName 中解析出 groupName（serviceName 格式：group@@service）
        String[] parts = serviceName.split("@@");
        String groupName = parts.length > 1 ? parts[0] : Constants.DEFAULT_GROUP;
        String pureServiceName = parts.length > 1 ? parts[1] : parts[0];
        return String.format("%s@@%s@@%s", namespaceId, groupName, pureServiceName);
    }
}
```

3. 优化效果

- 查询效率：从 O(n^4) 嵌套遍历降至 O(1) 哈希查询；
- 同步开销：数据同步时只需传输单个 `ServiceInfo` 对象，无需深拷贝；
- 扩展性：新增维度（如实例标签）只需在 `Instance` 类加字段，不影响存储结构。

## 五、服务发现与订阅核心源码剖析

服务发现是客户端获取健康实例的过程，2.X 通过 gRPC 长连接+主动推送 替代 1.X 的 HTTP 轮询，实时性与效率大幅提升。

### （一）通信方式转变：HTTP → gRPC

2.X 客户端默认使用 `RpcNamingClientProxy`（gRPC 代理），仅在兼容模式下使用 `HttpNamingClientProxy`，核心优势：

- 长连接复用：减少 TCP 握手与挥手开销；
- 二进制传输：ProtoBuf 格式比 JSON 体积小 30%+；
- 主动推送：服务变动时服务端主动推送，无需客户端轮询。

### （二）服务发现逻辑

客户端通过 gRPC 向服务端查询实例，服务端从注册表过滤健康实例后返回。

1. 客户端查询入口：`NacosNamingService`

```java
// com.alibaba.nacos.client.naming.NacosNamingService
@Override
public List<Instance> selectInstances(String serviceName, String groupName, boolean healthy) throws NacosException {
    // 1. 封装服务标识
    String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);
    // 2. 委托 gRPC 代理查询
    return namingClientProxy.selectInstances(groupedServiceName, healthy);
}
```

2. gRPC 代理查询实现：`RpcNamingClientProxy`

```java
// com.alibaba.nacos.client.naming.remote.gprc.RpcNamingClientProxy
@Override
public List<Instance> selectInstances(String serviceName, boolean healthy) throws NacosException {
    // 1. 构建查询请求（指定服务名、健康状态过滤）
    QueryInstancesRequest request = QueryInstancesRequest.newBuilder()
            .setServiceName(serviceName)
            .setNamespace(namespace)
            .setHealthyOnly(healthy) // 只返回健康实例
            .build();
    
    // 2. 同步调用服务端 gRPC 接口
    QueryInstancesResponse response = GrpcUtils.invokeGrpcSync(
        () -> namingStub.queryInstances(request),
        "queryInstances",
        serviceName
    );
    
    // 3. 解析响应，返回实例列表
    return response.getInstancesList().stream()
            .map(GrpcUtils::convertProtoToInstance)
            .collect(Collectors.toList());
}
```

3. 服务端查询处理：`InstanceQueryHandler`

服务端接收查询请求后，从注册表过滤实例：

```java
// com.alibaba.nacos.naming.remote.rpc.handler.InstanceQueryHandler
public QueryInstancesResponse handle(QueryInstancesRequest request) {
    String namespaceId = request.getNamespace();
    String serviceName = request.getServiceName();
    boolean healthyOnly = request.isHealthyOnly();
    
    // 1. 获取服务信息
    Service service = serviceManager.getService(namespaceId, serviceName);
    if (service == null) {
        return QueryInstancesResponse.newBuilder().build();
    }
    
    // 2. 过滤实例（健康状态、权重>0）
    List<Instance> filteredInstances = service.getAllInstances().stream()
            .filter(instance -> !healthyOnly || instance.isHealthy()) // 健康状态过滤
            .filter(instance -> instance.getWeight() > 0) // 权重过滤
            .collect(Collectors.toList());
    
    // 3. 构建响应
    return QueryInstancesResponse.newBuilder()
            .addAllInstances(GrpcUtils.convertInstancesToProtos(filteredInstances))
            .build();
}
```

### （三）服务订阅逻辑

订阅是"一次订阅，持续接收更新"的机制，客户端注册监听器，服务端变动时主动推送。

1. 客户端订阅入口：`NacosNamingService`

```java
// com.alibaba.nacos.client.naming.NacosNamingService
@Override
public void subscribe(String serviceName, String groupName, EventListener listener) throws NacosException {
    String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);
    // 1. 本地注册监听器（避免重复订阅）
    subscriptionManager.subscribe(groupedServiceName, listener);
    // 2. 向服务端发起订阅请求（建立推送关系）
    namingClientProxy.subscribe(groupedServiceName);
}
```

2. 服务端订阅管理：`SubscriptionManager`

服务端存储"服务-客户端"订阅关系，核心是 `ConcurrentHashMap` 映射：

```java
// com.alibaba.nacos.naming.core.SubscriptionManager
public class SubscriptionManager {
    // 存储：serviceKey → 订阅该服务的客户端集合
    private final ConcurrentHashMap<String, Set<Subscriber>> serviceSubscribers = new ConcurrentHashMap<>();

    // 注册订阅关系
    public void subscribe(String serviceKey, Subscriber subscriber) {
        serviceSubscribers.computeIfAbsent(serviceKey, key -> ConcurrentHashMap.newKeySet())
                          .add(subscriber);
    }

    // 获取服务的所有订阅者（用于推送）
    public Set<Subscriber> getSubscribers(String serviceKey) {
        return serviceSubscribers.getOrDefault(serviceKey, Collections.emptySet());
    }
}
```

3. 服务端推送触发：事件驱动

当服务实例变动时，通过事件触发推送逻辑：

```java
// com.alibaba.nacos.naming.core.PushService
@Component
public class PushService {
    @Autowired
    private SubscriptionManager subscriptionManager;

    // 监听实例变更事件，触发推送
    @EventListener
    public void onInstanceChange(ServiceInstanceEvent event) {
        Service service = event.getService();
        String serviceKey = ServiceKeyGenerator.generateServiceKey(service.getNamespaceId(), service.getServiceName());
        // 1. 获取订阅该服务的所有客户端
        Set<Subscriber> subscribers = subscriptionManager.getSubscribers(serviceKey);
        if (subscribers.isEmpty()) {
            return;
        }
        // 2. 构建推送数据（最新实例列表）
        ServiceInfo serviceInfo = serviceStorage.getServiceInfo(service.getNamespaceId(), service);
        // 3. 通过 gRPC 向客户端推送
        for (Subscriber subscriber : subscribers) {
            pushToClient(subscriber, serviceInfo);
        }
    }

    // gRPC 推送实现
    private void pushToClient(Subscriber subscriber, ServiceInfo serviceInfo) {
        ServiceChangeResponse response = ServiceChangeResponse.newBuilder()
                .setServiceInfo(GrpcUtils.convertServiceInfoToProto(serviceInfo))
                .build();
        // 通过客户端的 gRPC 连接推送
        subscriber.getGrpcChannel().writeAndFlush(response);
    }
}
```

## 六、服务变动推送订阅客户端源码剖析

客户端接收服务端推送的核心是 gRPC 流监听+本地缓存更新+监听器回调，关键类集中在 `nacos-naming-client` 模块。

### （一）推送接收：`NamingPushResponseHandler`

客户端通过 gRPC 流观察者监听服务端推送，核心是 `onNext` 方法（推送消息接收入口）：

```java
// com.alibaba.nacos.client.naming.remote.gprc.NamingPushResponseHandler
public class NamingPushResponseHandler implements StreamObserver<ServiceChangeResponse> {
    @Autowired
    private ServiceInfoHolder serviceInfoHolder; // 本地缓存管理器
    @Autowired
    private SubscriptionManager subscriptionManager; // 本地监听器管理器

    // 接收服务端推送的变动消息
    @Override
    public void onNext(ServiceChangeResponse response) {
        try {
            // 1. 解析推送数据（Proto → ServiceInfo）
            ServiceInfoProto serviceInfoProto = response.getServiceInfo();
            ServiceInfo newServiceInfo = GrpcUtils.convertProtoToServiceInfo(serviceInfoProto);
            
            // 2. 更新本地缓存（覆盖旧数据，保证一致性）
            serviceInfoHolder.processServiceInfo(newServiceInfo);
            
            // 3. 触发本地监听器回调（通知业务层）
            subscriptionManager.notifyListeners(newServiceInfo);
        } catch (Exception e) {
            log.error("处理服务推送失败", e);
        }
    }

    // 流异常处理
    @Override
    public void onError(Throwable t) {
        log.error("gRPC 推送流异常", t);
    }

    // 流关闭处理（重连机制）
    @Override
    public void onCompleted() {
        log.info("gRPC 推送流关闭，尝试重连");
        reconnect(); // 内部实现重连逻辑，确保推送不中断
    }
}
```

### （二）本地缓存更新：`ServiceInfoHolder`

客户端维护本地实例缓存，避免重复查询服务端，核心是 `serviceInfoMap`：

```java
// com.alibaba.nacos.client.naming.cache.ServiceInfoHolder
public class ServiceInfoHolder {
    // 本地缓存：serviceKey → ServiceInfo（serviceKey 格式：group@@service@@cluster）
    private final ConcurrentHashMap<String, ServiceInfo> serviceInfoMap = new ConcurrentHashMap<>();

    // 更新缓存（版本校验，避免旧数据覆盖新数据）
    public void processServiceInfo(ServiceInfo newServiceInfo) {
        String serviceKey = newServiceInfo.getKey();
        ServiceInfo oldServiceInfo = serviceInfoMap.get(serviceKey);
        
        // 版本校验：仅更新比本地新的数据（lastRefTime 为数据最后更新时间）
        if (oldServiceInfo != null && newServiceInfo.getLastRefTime() <= oldServiceInfo.getLastRefTime()) {
            return;
        }
        
        // 更新缓存
        serviceInfoMap.put(serviceKey, newServiceInfo);
        newServiceInfo.setLastRefTime(System.currentTimeMillis());
    }

    // 获取本地缓存的实例列表
    public ServiceInfo getServiceInfo(String serviceKey) {
        return serviceInfoMap.get(serviceKey);
    }
}
```

### （三）监听器回调：`SubscriptionManager`

客户端通知业务层服务变动，核心是遍历监听器并调用 `onEvent` 方法：

```java
// com.alibaba.nacos.client.naming.subscription.SubscriptionManager
public class SubscriptionManager {
    // 存储：serviceKey → 监听器集合
    private final ConcurrentHashMap<String, Set<EventListener>> listenerMap = new ConcurrentHashMap<>();

    // 触发监听器回调
    public void notifyListeners(ServiceInfo serviceInfo) {
        String serviceKey = serviceInfo.getKey();
        Set<EventListener> listeners = listenerMap.get(serviceKey);
        if (listeners == null) {
            return;
        }
        // 遍历监听器，触发业务逻辑
        for (EventListener listener : listeners) {
            try {
                listener.onEvent(new NamingEvent(serviceInfo.getName(), serviceInfo.getHosts()));
            } catch (Exception e) {
                log.error("监听器回调失败", e);
            }
        }
    }
}
```

- 业务层示例：客户端通过实现 `EventListener` 感知变动

```java
// 业务层订阅示例
namingService.subscribe("user-service", "DEFAULT_GROUP", new EventListener() {
    @Override
    public void onEvent(Event event) {
        NamingEvent namingEvent = (NamingEvent) event;
        List<Instance> instances = namingEvent.getInstances();
        // 业务逻辑：更新本地服务列表、重新计算负载均衡等
        System.out.println("服务变动，新实例列表：" + instances);
    }
});
```

## 七、服务端健康检查源码剖析

Nacos 2.X 健康检查从 客户端主动心跳 改为 服务端主动探活，减少客户端开销，同时提高探活准确性。

### （一）核心机制：服务端 gRPC 主动探活

2.X 对临时实例（默认）采用 gRPC 探活，对持久化实例采用 HTTP 探活（可配置），核心由 `HealthCheckReactor` 调度探活任务。

### （二）探活任务管理：`HealthCheckReactor`

负责创建、调度探活任务，采用线程池异步执行，避免阻塞主线程：

```java
// com.alibaba.nacos.naming.healthcheck.HealthCheckReactor
public class HealthCheckReactor {
    // 探活线程池（核心线程数：CPU 核心数 * 2）
    private final ScheduledExecutorService healthCheckExecutor = Executors.newScheduledThreadPool(
        Runtime.getRuntime().availableProcessors() * 2,
        new ThreadFactoryBuilder().setNameFormat("nacos-health-check-%d").build()
    );

    // 添加探活任务（延迟 0 秒，周期 20 秒执行一次）
    public void addHealthCheckTask(HealthCheckTask task) {
        healthCheckExecutor.scheduleAtFixedRate(
            task,
            0, // 初始延迟
            20000, // 周期（20 秒，默认值）
            TimeUnit.MILLISECONDS
        );
    }
}
```

### （三）gRPC 探活实现：`GrpcHealthCheckTask`

临时实例的探活任务，通过 gRPC 向客户端发送探活请求，判断实例是否存活：

```java
// com.alibaba.nacos.naming.healthcheck.v2.processor.GrpcHealthCheckTask
public class GrpcHealthCheckTask implements Runnable {
    private final Instance instance;
    private final Cluster cluster;
    // 探活超时时间（5 秒）
    private static final int TIMEOUT = 5000;

    @Override
    public void run() {
        try {
            // 1. 构建 gRPC 探活请求
            HealthCheckRequest request = HealthCheckRequest.newBuilder()
                    .setInstanceId(instance.getInstanceId())
                    .build();
            
            // 2. 向实例发送探活请求（客户端需实现 gRPC 健康检查接口）
            ManagedChannel channel = GrpcChannelFactory.getChannel(instance.getIp(), instance.getPort());
            HealthCheckGrpc.HealthCheckBlockingStub stub = HealthCheckGrpc.newBlockingStub(channel)
                    .withDeadlineAfter(TIMEOUT, TimeUnit.MILLISECONDS);
            HealthCheckResponse response = stub.check(request);
            
            // 3. 更新实例健康状态
            boolean newHealthy = response.getStatus() == HealthCheckResponse.ServingStatus.SERVING;
            updateInstanceHealthStatus(newHealthy);
        } catch (Exception e) {
            // 探活失败，标记为不健康
            updateInstanceHealthStatus(false);
        }
    }

    // 更新实例健康状态并发布事件
    private void updateInstanceHealthStatus(boolean healthy) {
        boolean oldHealthy = instance.isHealthy();
        if (oldHealthy == healthy) {
            return; // 状态无变化，不处理
        }
        // 更新健康状态
        instance.setHealthy(healthy);
        // 发布健康状态变更事件（触发推送、集群同步）
        eventPublisher.publishEvent(new HealthStateChangeEvent(instance.getInstanceId(), healthy));
    }
}
```

### （四）健康状态判断与实例剔除

服务端通过 "探活失败次数" 而非固定时间判断实例状态，默认规则：

1. 连续 1 次探活失败 → 标记实例为 `healthy=false`；
2. 连续 3 次探活失败（累计 60 秒） → 从注册表中移除实例（仅临时实例）；
3. 持久化实例探活失败不会被剔除，仅标记为不健康。

实例剔除逻辑在 `InstanceCleaner` 中实现：

```java
// com.alibaba.nacos.naming.core.InstanceCleaner
@Component
public class InstanceCleaner {
    @Autowired
    private ServiceManager serviceManager;

    // 监听健康状态变更事件，触发剔除判断
    @EventListener
    public void onHealthChange(HealthStateChangeEvent event) {
        String instanceId = event.getInstanceId();
        boolean healthy = event.isHealthy();
        if (healthy) {
            return;
        }
        // 查询实例
        Instance instance = instanceManager.getInstanceById(instanceId);
        if (instance == null || !instance.isEphemeral()) {
            return;
        }
        // 累计探活失败次数
        instance.incrementHealthFailureCount();
        // 连续 3 次失败，剔除实例
        if (instance.getHealthFailureCount() >= 3) {
            serviceManager.removeInstance(instance.getNamespaceId(), instance.getServiceName(), instance);
        }
    }
}
```

## 八、服务变动集群同步源码剖析

Nacos 2.X 集群同步采用 "Raft 协议（持久化实例）+ Distro 协议（临时实例）" 混合方案，确保数据一致性与性能平衡。

### （一）临时实例同步：Distro 协议

临时实例存储在内存，通过 Distro 协议实现 最终一致性 同步，核心是"分片+主动推送"。

1. Distro 分片策略

每个 Nacos 节点负责部分实例的同步（按 `instanceId` 哈希分片），避免全量同步：

```java
// com.alibaba.nacos.naming.consistency.implementation.DistroHashMapper
public class DistroHashMapper {
    // 根据 instanceId 计算分片，确定负责节点
    public List<String> mapInstanceToServers(String instanceId, List<String> allServers) {
        int hash = Math.abs(instanceId.hashCode());
        int index = hash % allServers.size();
        return Collections.singletonList(allServers.get(index));
    }
}
```

2. 同步实现：`DistroDataSyncService`

当临时实例变动时，节点向负责该实例的其他节点推送同步数据：

```java
// com.alibaba.nacos.naming.consistency.implementation.DistroDataSyncService
@Component
public class DistroDataSyncService {
    @Autowired
    private DistroHashMapper hashMapper;
    @Autowired
    private GrpcClient grpcClient;

    // 同步临时实例变动
    public void syncEphemeralInstance(Instance instance) {
        // 1. 计算负责该实例的节点
        List<String> targetServers = hashMapper.mapInstanceToServers(instance.getInstanceId(), serverListManager.getServerList());
        // 2. 构建同步数据
        DistroData data = DistroDataBuilder.buildInstanceData(instance);
        // 3. 向目标节点推送同步请求
        for (String server : targetServers) {
            syncToServer(server, data);
        }
    }

    // gRPC 同步请求
    private void syncToServer(String server, DistroData data) {
        DistroSyncRequest request = DistroSyncRequest.newBuilder()
                .setData(ByteString.copyFrom(data.getContent()))
                .build();
        grpcClient.sendRequest(server, request, DistroSyncResponse.class);
    }
}
```

### （二）持久化实例同步：Raft 协议

持久化实例存储在 MySQL，通过 Raft 协议实现 强一致性 同步，确保所有节点数据一致。

1. Raft 核心实现：`RaftCore`

负责 Raft 协议的 ** Leader 选举、日志复制、数据提交**：

```java
// com.alibaba.nacos.naming.consistency.implementation.raft.RaftCore
public class RaftCore {
    // Raft 状态（FOLLOWER/CANDIDATE/LEADER）
    private volatile RaftPeer.State state = RaftPeer.State.FOLLOWER;
    // Raft 日志（存储所有变更操作）
    private final List<LogEntry> logEntries = new CopyOnWriteArrayList<>();

    // 1. Leader 选举（省略细节，核心是投票机制）
    public void startElection() {
        // 触发选举逻辑，获取其他节点投票，得票超过半数成为 Leader
    }

    // 2. 日志复制（Leader 向 Follower 同步日志）
    public void replicateLog(LogEntry entry) {
        if (state != RaftPeer.State.LEADER) {
            throw new IllegalStateException("非 Leader 节点，无法复制日志");
        }
        // 添加日志到本地
        logEntries.add(entry);
        // 向所有 Follower 节点复制日志
        for (String follower : followerList) {
            replicateToFollower(follower, entry);
        }
    }

    // 3. 数据提交（日志复制到多数节点后，提交数据）
    public void commitLog(LogEntry entry) {
        // 检查日志是否已复制到多数节点
        if (isLogReplicatedToMajority(entry.getIndex())) {
            // 提交日志（更新本地注册表）
            commitEntry(entry);
            // 通知 Follower 提交日志
            notifyFollowersCommit(entry.getIndex());
        }
    }
}
```

2. 持久化实例同步触发

当持久化实例变动时，Leader 节点通过 Raft 日志复制同步到所有 Follower 节点：

```java
// com.alibaba.nacos.naming.consistency.implementation.RaftConsistencyServiceImpl
@Override
public void put(String key, Record value) throws NacosException {
    // 1. 构建 Raft 日志条目
    LogEntry entry = LogEntry.newBuilder()
            .setKey(key)
            .setValue(ByteString.copyFrom(SerializeUtils.serialize(value)))
            .setType(LogEntryType.WRITE)
            .build();
    // 2. Leader 节点复制日志
    raftCore.replicateLog(entry);
    // 3. 等待日志提交
    raftCore.waitForLogCommit(entry.getIndex(), 5000);
}
```
