# TomCat整体结构以及设计源码分析

# 1. Tomcat是什么？

Tomcat 是一款**开源的 Java Web 服务器 + Servlet 容器**，由 Apache 软件基金会开发维护，主要用于部署和运行 Java Web 应用程序（如基于 Servlet、JSP、Spring MVC 等技术的应用）。它是 Java 生态中最流行的 Web 容器之一，兼具轻量性、稳定性和易扩展性，广泛用于开发和生产环境。

```python
apache-tomcat-10.0.0/
├── bin/                # 脚本与可执行文件目录
├── conf/               # 配置文件目录
├── lib/                # 核心依赖库目录
├── webapps/            # Web 应用部署目录
├── logs/               # 日志文件目录
├── temp/               # 临时文件目录
├── work/               # JSP 编译缓存目录
└── LICENSE、NOTICE 等  # 许可证和说明文件
```
<img width="3182" height="870" alt="image" src="https://github.com/user-attachments/assets/d66e17b1-c5e5-4c58-b54e-464dc92b3c83" />



# 2. WEB应用部署的三种方式

### 2.1 自动部署（Webapps 目录部署）

这是最简便、最常用的部署方式，适合开发环境或简单场景，依赖 Tomcat 的自动检测机制。

#### 原理：

Tomcat 启动时会自动扫描 `webapps/` 目录（默认部署目录），对目录中的 Web 应用（包括 **WAR 包** 或 **解压后的应用目录**）进行部署；运行过程中，若向 `webapps/` 目录添加新的应用（如复制 WAR 包），Tomcat 也会自动检测并部署（需开启自动部署配置，默认开启）。

#### 操作步骤：

1. 将 Web 应用打包为 WAR 包（如 `myapp.war`），或直接获取解压后的应用目录（如 `myapp/`）。

2. 将 WAR 包或目录复制到 Tomcat 的 `webapps/` 目录下。

3. 启动 Tomcat（`bin/startup.sh` 或 `startup.bat`），Tomcat 会自动处理：

   * 若为 WAR 包：自动解压为同名目录（如 `myapp.war` → `myapp/`）。

   * 若为目录：直接识别为 Web 应用。

4. 访问应用：通过 `http://localhost:8080/应用名` 访问（如 `http://localhost:8080/myapp`）。

#### 优缺点：

* **优点**：操作简单，无需手动配置，适合开发调试或快速部署。

* **缺点**：灵活性低（应用必须放在 `webapps/` 目录）；若需修改应用路径或配置，需额外操作。

### 2.2 配置文件部署（Context 配置部署）

通过手动配置 Tomcat 的 XML 配置文件，指定应用的部署路径和实际存储位置，适合需要自定义部署路径（非 `webapps/` 目录）或复杂配置的场景。

#### 原理：

Tomcat 通过 **Context 元素** 定义 Web 应用的映射关系（访问路径 → 实际目录）。Context 配置可放在两个位置：

* 在 `conf/server.xml` 中配置（不推荐）

在 `server.xml` 的 `<Host>` 标签内添加 `<Context>` 元素，指定应用的访问路径和实际位置。

**示例**：

xml

```xml
<Server ...><Service ...><Engine ...><Host name="localhost" appBase="webapps" ...><!-- 配置 myapp 应用：访问路径为 /myapp，实际目录为 /opt/apps/myapp --><Context path="/myapp" docBase="/opt/apps/myapp" reloadable="true" /></Host></Engine></Service></Server>
```

* `path`：应用的访问路径（如 `/myapp` 对应 `http://localhost:8080/myapp`）。

* `docBase`：应用的实际存储路径（可绝对路径或相对 `appBase` 的路径）。

* `reloadable`：是否自动重载（`true` 表示当应用类文件变化时，Tomcat 自动重启应用，适合开发环境）。

- 在 `conf/Catalina/localhost/` 下创建独立 XML 文件（推荐）

在 `conf/Catalina/localhost/` 目录下创建以 **应用访问路径** 命名的 XML 文件（如 `myapp.xml`），文件内容为 `<Context>` 元素。

**示例**：创建 `conf/Catalina/localhost/myapp.xml`，内容：

xml

```xml
<!-- 访问路径为 /myapp，实际目录为 /opt/apps/myapp --><Context docBase="/opt/apps/myapp" reloadable="true" />
```

* 此时应用的访问路径由 XML 文件名决定（`myapp.xml` → 访问路径 `/myapp`）。

* 无需重启 Tomcat，添加 / 修改此文件后，Tomcat 会自动部署 / 更新应用（热部署）。

#### 优缺点：

* **优点**：灵活（应用可放在任意目录）、支持热部署（独立 XML 文件方式）、便于集中管理配置。

* **缺点**：需手动编写配置文件，对新手稍复杂。

### 2.3 Manager 应用部署（远程管理部署）

通过 Tomcat 自带的 **Manager 应用**（网页或 API）远程部署应用，适合生产环境中无需直接操作服务器文件系统的场景（如运维人员远程部署）。

#### 前提：配置 Manager 访问权限

1. 编辑 `conf/tomcat-users.xml`，添加具有 `manager-gui`（网页管理）或 `manager-script`（API 管理）权限的用户：

2. xml

```xml
<tomcat-users><!-- 允许通过网页管理部署 --><user username="admin" password="123456" roles="manager-gui" /></tomcat-users>
```

1. 重启 Tomcat 使配置生效。

#### 操作步骤（网页方式）：

1. 访问 Manager 应用：`http://localhost:8080/manager/html`，输入配置的用户名 / 密码登录。

2. 在 “Deploy” 区域部署应用：

   * **方式 1（上传 WAR 包）**：点击 “Browse” 选择本地 WAR 包，点击 “Deploy” 上传并部署。

   * **方式 2（指定 URL）**：在 “Context path” 输入访问路径（如 `/myapp`），在 “WAR or Directory URL” 输入远程 WAR 包的 URL（如 `http://example.com/myapp.war`），点击 “Deploy”。

3. 部署成功后，可在 “Applications” 列表中看到应用，支持启动、停止、卸载等操作。

# 3. Tomcat整体架构分析

**Tomcat**本质上只**需要**实现两个核心功能：

* 处理Socket连接，负责网络字节流Request和Response对象的转化

* 加载和管理Servlet，以及具体处理Request请求

因此Tomcat设计两个核心组件连接器（Connector）和容器（Container）来分别做这两件事情，连接器负责对外交流，容器负责内部处理


<img width="2760" height="1400" alt="image" src="https://github.com/user-attachments/assets/cae9cc54-f3d8-4dba-8875-0deba0771b48" />




## 3.1 Tom核心组件详解

| **组件**        | **作用**                                                                                           |
| ------------- | ------------------------------------------------------------------------------------------------ |
| **Server**    | 整个 Tomcat 实例的顶层容器，代表一个运行的 Tomcat 服务器，可包含多个 Service。                                              |
| **Service**   | 关联一个或多个 Connector 与一个 Container，负责将请求从 Connector 传递到 Container。                                  |
| **Connector** | 监听指定端口（如 8080），接收客户端 HTTP 请求，解析请求数据并传递给 Container，同时将 Container 的响应返回给客户端。支持 HTTP、HTTPS、AJP 等协议。 |
| **Container** | 负责处理请求的核心组件，内部采用层级结构：                                                                            |
| **Servlet**   | 业务逻辑组件，由 Container 调用，处理具体的 HTTP 请求（如接收参数、访问数据库、生成响应）。                                           |

### Server组件

整个 Tomcat 实例的顶层容器，代表一个运行的 Tomcat 服务器，可包含多组服务（ Service ）。负责管理和启动各个Service，同时监听8005端口发送过来的shutdown命令

### Service组件

每个service组件都包含了若干用于接收客户端消息的Connector组件和处理请求的Engine组件，Service组件还包含了若干Executor组件，每个Executor都是一个线程池，它可以为Service内部所有组件提供线程池执行任务。Tomcat内部可能有多个Service，这样的设计也是出于灵活性的考虑。通过在Tomcat内部配置多个Service，可以实现通过不同端口号来访问同一台机器上部署的不同应用



<img width="2760" height="1400" alt="image" src="https://github.com/user-attachments/assets/e3d8b4e6-f43e-4c1d-a933-1636a6947687" />


### Connector组件

Tomcat与外部世界的连接器，监听固定端口接收外部请求，传递给Container,并将Container处理的结果返回给外部。连接器对Servlet容器屏蔽了不同的应用层协议以及IO模型，无论是HTTP还是AJP，在容器中获取到的都是一个标准的ServletRequest对象

### Container组件

容器，顾名思义就是用来装载对象的器具，在Tomcat里，容器就是用了装载servlet的。tomcat通过分层架构，使得Servlet容器具有很好的灵活性。Tomcat设计了四种容器，分别是Engine、Host、Context和Wrapper。这四种容器并不是平行关系，而是父子关系。

* **Engine**：引擎，Servlet的顶层容器，用来管理多个虚拟站点，一个Service最多只能有一个Engine

* **Host**:虚拟主机，负责web应用的部署和context的创建。可以给Tomcat配置多个虚拟主机地址，而一个虚拟主机可以部署多个Web应用程序

* **Context**：Web应用程序上下文，包含多个Wrapper，负责Web配置的解析，管理所有的Web资源。一个Context对应一个Web应用程序

* **Wrapper**:表示一个Servlet,最底层的容器，是对Servlet的封装，负责Servlet实例的创建、执行和销毁

<img width="289" height="290" alt="image" src="https://github.com/user-attachments/assets/7abb6b51-c53a-4f70-b8b1-90947b6fe8d0" />


## 3.2 请求定位Servlet的过程

网络请求先到达 Connector，也就是连接器，它负责监听端口，接收 HTTP 请求。然后，Connector 会把请求封装成 Request 对象，可能还有 Response 对象，然后传递给 Engine 处理。接下来，Engine 会根据请求的 Host 头找到对应的 Host 虚拟主机。Host 再根据请求的上下文路径（Context Path）找到对应的 Context，也就是 Web 应用。然后，Context 内部需要根据请求的 Servlet 路径（Servlet Path）来匹配对应的 Servlet。这里可能涉及到 Web 应用的 web.xml 配置或者注解定义的 Servlet 映射。

# 4. Tomcat架构设计

## 4.1 Connector高内聚低耦合设计

Tomcat连接器需要实现的功能：

* 监听网络端口

* 接受网络连接请求

* 读取请求网络字节流

* 根据具体应用层协议解析字节流，生成统一的TomcatRequest对象

* 将TomcatRequest对象转化成ServletRequest对象

* 调用Servlet容器，得到ServletResponse对象

* 将ServletResponse对象转化成TomcatResponse对象

* 将TomcatResponse转成网络字节流

* 将响应字节流写回浏览器

通过连接器需要实现的功能列表，会发现连接器需要实现3个高内聚的功能：

* 网络通信

* 应用层协议解析

* Tomcat Request/Response 与 Servlet Request/Response相互转化

### ProtocolHandler

> 因此Tomcat设计者设计了3个组件来实现这3个核心功能，分别是EndPoint、Processor和Adapter

* **Endpoint：**&#x8D1F;责提供字节流给Processor

* **Processor：**&#x8D1F;责提供TomcatRequest对象给Adapter

* **Adapter**：负责提供ServletRequest对象给容器

> 组件之间通过抽象接口交互，这样的好处是封装变化。这是面向对象设计的精髓，将系统中经常变化的部分和稳定的部分隔离，有助于增加系统的复用性，并降低系统耦合度



<img width="4698" height="1430" alt="image" src="https://github.com/user-attachments/assets/78713cf6-af9e-4b54-9133-bf2cd6f8e859" />






由于IO模型和应用层协议之间可以自由组合，比如NIO+HTTP或者NIO2+NJP。Tomcat设计者将网络通信和应用层协议解析放在一起考虑，设计了一个叫做ProtocolHandler的接口来封装这两种变化点。各种协议和通信模型的组合有相应的具体实现类，比如：Http11NioProtocol和AjpNioProtocol

除了这些变化点，系统也存在一些相对稳定的部分，因此tomcat设计了一系列抽象基类来封装这些稳定的部分，抽象基类AbstractProtocol实现类ProtocolHandler接口。每一种应用层协议有自己的抽象基类，比如AbstractAjpProtocol和AbstractHttp11Protocol，具体协议的实现类扩展了协议层抽象基类。



<img width="821" height="468" alt="image" src="https://github.com/user-attachments/assets/ef9a7208-6550-4783-a507-72d1427b4e17" />


#### Endpoint

endpoint是通信端点，即监听通信的接口，是具体的Socket接收和发送处理器，是对传输层的抽象，因此EndPoint是用来实现TCP/IP协议的。endpoint是一个接口，对应的抽象实现类是AbstractEndpoint，其子类中，例如NioEndpoint和Nio2Endpoint中，有两个重要的子组件：Acceptor和SocketProcessor.其中Acceptor用于监听Socket连接请求。SocketProcessor用于处理接收到的Socket请求，它实现Runnable接口，在Run方法里调用协议处理组件Processor进行处理，为了提高处理能力，SocketProcessor被提交到线程池来执行，而这个线程池叫做执行器（Executor）

#### Processor

Processor用来实现HTTP/AJP协议，Processor接收来自Endpoint的Socket,读取字节流解析成TomcatRequset和Response对象，并通过Adapter将其提交到容器处理，Processor是对应用层协议的抽象，Processor是一个接口，定义了请求的处理等方法，它的抽象实现类AbstractProcessor对一些协议共有的属性进行封装，没有对方法进行实现。具体的实现有AJPProcessor\HTTP11Processor等，这些具体实现类实现了特定协议的解析方法和请求处理方式。EndPoint接收到Socket连接后，生成一个SocketProcessor任务提交到线程池去处理，SocketProcessor的Run方法会调用Processor组件去解析应用层协议，Processor通过解析生成Request对象后没回调有Adapter的Service方法。

<img width="4950" height="1430" alt="image" src="https://github.com/user-attachments/assets/7a2eef46-6b54-4baf-abf2-0cf376bcbb12" />


#### **Adapter**

由于协议不同，客户端发过来的请求信息也不相同，Tomcat定义了自己的Request类来存放这些请求信息。ProtocolHandler接口负责解析请求并生成TomcatRequest类。但是这个Request对象不是标准的ServletRequest，也就是意味着，不能使用TomcatRequest作为参数来调用容器。Tomcat的设计者的解决方案是引入CoyoteAdapter，这是适配器模式的经典运用，连接器调用CoyoteAdapter的Service方法，传入的是TomcatRequest对象CoyoteAdapter负责将TomcatRequest和ServletRequest进行传化，再调用容器的Service方法。



设计复杂系统的思路：

首先分析需求，列出功能点，根据功能以及高内聚、低耦合的原则确定子模块，然后找出子模块中的变化点和不变点，用接口和抽象基类去封装不变点，再抽象基类中定义模板方法，让子类自行实现抽象方法，也就是具体子类去实现变化点。tomcat的父子容器组合模式设计，tomcat通过组合模式来管理这些容器。具体的实现方法是，所有容器组件都实现了Container接口，因此组合模式可以使得用户对单容器对象和组合容器对象具有使用的一致性。

## 4.2 Pipeline-Valve责任链模式设计

连接器中的Adapter会调用容器的Service方法来执行Servlet，最先拿到请求的是Engine容器，Engine容器对请求做一些处理后，会把请求传给自己子容器Host继续处理，依次类推，最后这个请求会传给Wrapper容器，Wrapper会调用最终的Servlet处理，这个过程中使用了Pipeline-Valve管道责任链模式

这种设计模式是指再一个请求处理的过程中有很多处理者一次对请求进行处理，每个处理者负责做自己相应的处理，处理完成之后再调用下一个处理者继续处理。

<img width="3620" height="2316" alt="image" src="https://github.com/user-attachments/assets/c8d0c35e-e97e-46ba-8687-e1fe0cce5e82" />


## 4.3 Tomcat生命周期设计

通过对Tomcat架构的分析，我们知道了Tomcat都有哪些组件，以及组件之间的关系，处理http请求的流程，如果想让tomcat能够对外提供服务，我们需要创建、组装并启动Tomcat组件，在服务停止的时候，我们还需要释放资源，销毁Tomcat组件，这是一个动态的过程。Tomcat需要动态地管理这些组件的生命周期。

### 一键式启停：LifeCycle接口

Tomcat 组件层级复杂（如 `Server → Service → Engine → Host → Context`），若每个组件各自实现启停逻辑，会导致：

* 组件间依赖混乱（如必须先启动 `Connector` 才能接收请求，但 `Connector` 依赖 `Service` 初始化）；

* 无法统一控制（启动 / 停止操作需逐个处理组件，无法 “一键操作”）。

`LifeCycle` 接口通过以下方式解决问题：

1. **统一生命周期方法**：定义所有组件必须实现的初始化、启动、停止、销毁方法；

2. **状态管理**：规范组件的生命周期状态（如 “未初始化”“启动中”“已启动” 等）及状态转换规则；

3. **事件监听**：支持通过监听器感知组件状态变化，实现扩展逻辑（如启动前初始化资源、停止后释放连接）。

#### 1. 核心方法（定义在 `org.apache.catalina.Lifecycle` 接口中）

#### 2. 生命周期状态流转（关键状态）

Tomcat 定义了严格的状态转换规则（通过 `LifecycleState` 枚举），确保组件按顺序执行生命周期操作，核心状态流转如下：

plaintext

```plain&#x20;text
NEW（新建未初始化）→ INITIALIZING（初始化中）→ INITIALIZED（已初始化）
→ STARTING_PREP（启动准备）→ STARTING（启动中）→ STARTED（已启动）
→ STOPPING_PREP（停止准备）→ STOPPING（停止中）→ STOPPED（已停止）
→ DESTROYING（销毁中）→ DESTROYED（已销毁）
```

* 每个状态转换只能通过特定方法触发（如 `start()` 触发从 `INITIALIZED` 到 `STARTED` 的转换）；

* 状态转换不可逆（如 `STARTED` 不能直接回到 `INITIALIZED`，必须先 `stop()` 到 `STOPPED`）。

#### 3. “一键式启停” 的实现逻辑

`LifeCycle` 接口的层级联动机制，是 “一键式启停” 的核心：**顶层组件的生命周期方法会自动触发所有子组件的对应方法**。

以 Tomcat 启动为例（从 `Server` 到子组件）：

1. 用户执行 `startup.sh` 或点击启动按钮，最终调用 **`Server` 的 `start()` 方法**；

2. `Server` 的 `start()` 会先执行自身启动逻辑，然后遍历所有子组件（`Service`），调用每个 `Service` 的 `start()` 方法；

3. `Service` 的 `start()` 会启动自身，再调用子组件 `Engine` 和 `Connector` 的 `start()` 方法；

4. `Engine` 的 `start()` 会启动自身，再调用子组件 `Host` 的 `start()` 方法；

5. 以此类推，直到最底层的 `Context`（Web 应用）和 `Wrapper`（Servlet）启动完成。

停止过程同理：调用 `Server` 的 `stop()` 方法，会逐级触发所有子组件的 `stop()` 方法，最终实现 “一键停止”。

#### 4.3.4 监听器机制：扩展生命周期行为

`LifeCycle` 接口通过 **`LifecycleListener` 监听器**支持扩展，允许在组件状态变化时执行自定义逻辑（如日志记录、资源预热）。

示例：给 `Host` 组件添加启动监听器，在 Host 启动后打印日志：

```java
// 自定义监听器
public class HostStartListener implements LifecycleListener {
    @Override
    public void lifecycleEvent(LifecycleEvent event) {
        // 当事件类型为“已启动”时触发
        if (Lifecycle.START_EVENT.equals(event.getType())) {
            Host host = (Host) event.getSource();
            System.out.println("Host " + host.getName() + " 已启动，准备处理请求");
        }
    }
}

// 在 Host 组件中注册监听器（通常在 server.xml 或代码中配置）
Host host = new StandardHost();
host.addLifecycleListener(new HostStartListener());
```

