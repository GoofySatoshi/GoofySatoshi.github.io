---
title: Spring Cloud OpenFeign 学习笔记
date: 2025-08-26 15:13:19
tags:
---

一、服务调用方式对比
在 Spring Cloud 中，引入 OpenFeign 之前，常使用 Ribbon（负载均衡器）与 RestTemplate 进行服务间调用。但此方式代码耦合性强，维护性与可读性差，修改不便。而 Spring Cloud OpenFeign 采用依赖注入方式调用，如同调用本地 service，极大提升了代码的简洁性与可维护性。


二、OpenFeign 使用步骤
1. 引入依赖：在项目中引入 OpenFeign 的相关 Maven 依赖。
2. 添加注解：在 Spring Boot 启动类上添加 @EnableFeignClients 注解，开启 Feign 客户端功能。
3. 定义接口：
- 方式一：在自定义的 SDK 中定义带有 @FeignClient 注解的接口。
- 方式二：自行定义 OpenFeign 接口，该接口用于声明服务调用方法。
4. 注入使用：将定义好的 Feign 接口注入到需要调用的地方，即可实现服务间的远程调用。
   三、OpenFeign 接口参数处理
- 默认 **@RequestBody**：若接口参数未添加其他注解，OpenFeign 默认添加 @RequestBody 注解。但 @RequestBody 通常只能接收一个对象，传入多个对象会报错。
- GET 请求参数：对于 GET 类型请求，可写多个基本类型参数，这些参数会自动拼接到 URL 上。若想用 form 表单形式传递参数，可使用 @SpringQueryMap 注解，同样会将参数拼接到 URL 上。
  四、OpenFeign 结构与原理
- 服务端与调用端：
    - 服务提供端：通过在接口方法上添加 @FeignClient 注解，将服务封装成接口方法，并打包成 SDK。
    - 服务调用端：引入服务提供端的 SDK，在 Spring Boot 启动时，利用 JDK 动态代理生成代理对象，并注入到 Bean 容器中。当调用接口方法时，通过 InvocationHandler 调用处理器进行远程调用。
- 处理流程：
    - 调用接口方法后，**Contract** 根据注解声明的规则解析底层的 **MethodHandler**。
    - MethodHandler 底层调用 **RequestTemplate**，**RequestTemplate** 内部实现了编码和解码功能。服务发起端对请求进行编码，服务调用端在解析返回结果时进行解码。
    - 最外层有 Interceptors 拦截器，用于对请求和响应进行拦截，例如记录日志等操作。
    - Feign Client 基于负载均衡器和重试器进行策略调控。请求到达服务调用端后，经过解码，**MethodHandler** 根据规则映射找到调用的方法，通过处理器获取 InvocationHandler 的调用结果，并解析成 Response 的 Bean。
      五、OpenFeign 增强方式
      （一）日志增强
- 配置 Feign 的 Config Bean 对象：通过配置该 Bean 对象可实现日志增强。
- YAML 文件配置：在 YAML 文件中进行配置，更为灵活且支持配置中心实时更改。
- 日志级别：
    - NONE：不记录日志。
    - BASIC：仅记录请求方法、URL、响应状态代码及执行时间。
    - HEADERS：在 BASIC 基础上记录请求和响应的 Header。
    - FULL：全量记录请求、响应的 Header、Body 及原始数据。
      （二）HTTP 连接优化
- 引入 Feign HC5：OpenFeign 默认的 HTTP 连接无连接池，性能较差。引入 **Feign HC5**（Apache HTTP5 的实现）依赖，其内部自带连接池，可优化调用效率，减少连接的频繁创建与销毁。由于 OpenFeign 的自动配置类已将相关属性默认配置为启用状态，因此即便不在 YAML 中配置，引入依赖后也会生效。
- 替换为 OKHTTP：除 Feign HC5 外，还可引入 OKHTTP 依赖，并在 YAML 中开启 OKHTTP 配置，将底层 HTTP 请求替换为 OKHTTP。
  （三）压缩配置
- gzip 压缩：OpenFeign 支持 gzip 压缩配置，当请求超过某个阈值（如 1024 字节），可在 YAML 中开启 gzip 压缩，并配置需要压缩的请求类型及最小压缩阈值等参数，以优化请求体大小和传输效率。
- OKHTTP 特殊情况：若使用 OKHTTP 作为底层 HTTP 请求，因其内部默认使用 transparent 压缩方式，一般无需额外配置 gzip 压缩。
  （四）自定义编解码器
  OpenFeign 提供扩展点，可通过实现 Encoder 和 Decoder 接口来自定义编解码器，并在 YAML 中进行配置。

（五）拦截器实现
- BasicAuthRequestInterceptor：通过注入该 Bean 并进行配置来实现拦截器功能，但此方式灵活性欠佳。
- 实现 RequestInterceptor 接口：推荐使用此方式，通过实现接口来定义拦截逻辑，更加灵活。
  六、YAML 基本配置示例
  feign:
  client:
  config:
  default:
  loggerLevel: FULL # 日志级别设置为 FULL
  httpclient:
  enabled: true # 启用 Feign HC5 连接池
  max-connections: 200 # 最大连接数
  max-connections-per-route: 50 # 每个路由的最大连接数
  compression:
  request:
  enabled: true # 开启请求压缩
  mime-types: text/xml,application/xml,application/json # 需要压缩的请求类型
  min-request-size: 1024 # 最小压缩阈值
  okhttp:
  enabled: true # 启用 OKHTTP


在对 OpenFeign 进行配置时，优先考虑 YAML 配置方式，因其可