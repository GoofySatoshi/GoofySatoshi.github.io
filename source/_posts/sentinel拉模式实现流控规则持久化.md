---
title: Sentinel 拉模式实现详解
date: 2025-09-04 23:35:00
tags: [Sentinel, 流量控制, 拉模式]
categories: [技术, 微服务]
---
 
## 一、为什么需要 Sentinel 拉模式？
 
Sentinel 默认的内存模式存在明显缺陷：限流规则仅存储在客户端内存中，应用重启后规则全部丢失，无法满足生产环境的规则持久化与统一管理需求。
 
拉模式是 Sentinel 规则持久化的核心方案之一，其核心逻辑是客户端主动通过定时轮询，从外部存储（如 MySQL、本地文件）拉取规则，并结合 SPI 扩展机制实现自定义数据源，彻底解决“重启失效”问题。本文将完整拆解拉模式的实现流程，基于 MySQL 存储规则，适配 Spring Boot 场景。
 
## 二、拉模式与 SPI 的关系
 
在动手实现前，先理清两个关键概念，避免与推模式混淆：
 
概念 作用说明 
拉模式 客户端通过定时任务主动轮询外部存储（如 MySQL），获取最新规则并加载到 Sentinel，实时性取决于轮询间隔。 
SPI 扩展机制 Sentinel 提供  ReadableDataSource  接口作为 SPI 扩展点，允许自定义“规则读取逻辑”，框架启动时自动扫描加载。 
 
## 三、完整实现步骤（基于 MySQL + Spring Boot）
 
### 1. 环境准备：引入依赖
 
在  pom.xml  中添加 Sentinel 核心依赖、SPI 扩展依赖及 MySQL 连接依赖（使用 Druid 连接池简化操作）：
 
<!-- 1. Sentinel 核心依赖 -->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-core</artifactId>
    <version>1.8.6</version>
</dependency>

<!-- 2. Sentinel DataSource 扩展（SPI 扩展基础） -->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-extension</artifactId>
    <version>1.8.6</version>
</dependency>

<!-- 3. MySQL 连接依赖 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
</dependency>

<!-- 4. Druid 连接池 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.2.16</version>
</dependency>

<!-- 5. Spring Boot 基础（已引入可忽略） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <version>2.7.15</version>
</dependency>
 
 
### 2. 存储设计：创建 MySQL 规则表
 
在 MySQL 中创建 Sentinel 限流规则表，用于持久化存储规则配置（支持后续扩展降级、热点规则等）：
 
-- 数据库：sentinel_db（可自定义）
CREATE DATABASE IF NOT EXISTS sentinel_db DEFAULT CHARSET utf8mb4;
USE sentinel_db;

-- 限流规则表
CREATE TABLE IF NOT EXISTS sentinel_flow_rule (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `resource` varchar(255) NOT NULL COMMENT '限流资源名（如接口路径 /order/create）',
  `limit_app` varchar(64) DEFAULT 'default' COMMENT '限制来源（默认所有来源）',
  `grade` tinyint(1) NOT NULL COMMENT '限流类型：1=QPS，0=线程数',
  `count` double NOT NULL COMMENT '限流阈值（如 QPS=10）',
  `strategy` tinyint(1) DEFAULT 0 COMMENT '流控策略：0=直接模式，1=关联模式，2=链路模式',
  `control_behavior` tinyint(1) DEFAULT 0 COMMENT '流控效果：0=快速失败，1=预热，2=排队',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT 'Sentinel 限流规则表';

-- 插入测试数据：对 /order/create 接口限制 QPS=10
INSERT INTO sentinel_flow_rule (resource, limit_app, grade, count, strategy, control_behavior)
VALUES ('/order/create', 'default', 1, 10, 0, 0);
 
 
### 3. 核心实现：自定义 SPI 数据源（DataSource）
 
拉模式的核心是实现  ReadableDataSource  接口（Sentinel 的 SPI 扩展点），在类中封装“定时轮询逻辑”与“MySQL 规则读取逻辑”。
 
#### 自定义 DataSource 代码
 
package com.example.sentinel.datasource;

import com.alibaba.csp.sentinel.datasource.ReadableDataSource;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRule;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRuleManager;
import com.alibaba.druid.pool.DruidDataSource;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

/**
 * Sentinel 拉模式数据源：从 MySQL 定时拉取限流规则
 * 实现 ReadableDataSource 接口，通过 SPI 被 Sentinel 自动加载
 */
public class MysqlPullFlowRuleDataSource implements ReadableDataSource<Void, List<FlowRule>> {

    // 1. 配置参数（生产环境建议从配置文件注入，此处简化为硬编码）
    private final String jdbcUrl = "jdbc:mysql://127.0.0.1:3306/sentinel_db?useSSL=false&serverTimezone=UTC";
    private final String dbUsername = "root"; // 替换为你的 MySQL 账号
    private final String dbPassword = "123456"; // 替换为你的 MySQL 密码
    private final long pullInterval = 5000; // 定时轮询间隔：5秒（可根据需求调整）

    // 2. 数据库连接池（复用连接，避免频繁创建销毁）
    private final DruidDataSource druidDataSource;
    // 3. 定时任务线程池（执行规则拉取任务）
    private final ScheduledExecutorService scheduler;

    /**
     * 构造方法：初始化连接池 + 启动定时轮询
     */
    public MysqlPullFlowRuleDataSource() {
        // 初始化 Druid 连接池
        this.druidDataSource = initDruidDataSource();
        // 初始化定时任务：项目启动立即拉取一次，之后每 5 秒拉取
        this.scheduler = Executors.newSingleThreadScheduledExecutor();
        this.scheduler.scheduleAtFixedRate(
                this::loadRuleToSentinel, // 轮询执行的方法：拉取并加载规则
                0, // 初始延迟：0秒（立即执行）
                pullInterval, // 轮询间隔
                TimeUnit.MILLISECONDS
        );
    }

    /**
     * 核心1：从 MySQL 读取最新规则
     */
    private List<FlowRule> readRuleFromMysql() {
        List<FlowRule> flowRules = new ArrayList<>();
        String sql = "SELECT resource, limit_app, grade, count, strategy, control_behavior FROM sentinel_flow_rule";

        // try-with-resources 自动关闭连接、Statement
        try (Connection conn = druidDataSource.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql);
             ResultSet rs = pstmt.executeQuery()) {

            // 遍历结果集，转换为 Sentinel 识别的 FlowRule 对象
            while (rs.next()) {
                FlowRule rule = new FlowRule();
                rule.setResource(rs.getString("resource"));       // 限流资源（接口/方法名）
                rule.setLimitApp(rs.getString("limit_app"));     // 限制来源
                rule.setGrade(rs.getInt("grade"));               // 限流类型（QPS/线程数）
                rule.setCount(rs.getDouble("count"));            // 限流阈值
                rule.setStrategy(rs.getInt("strategy"));         // 流控策略
                rule.setControlBehavior(rs.getInt("control_behavior")); // 流控效果
                flowRules.add(rule);
            }
        } catch (Exception e) {
            throw new RuntimeException("从 MySQL 拉取 Sentinel 规则失败", e);
        }
        return flowRules;
    }

    /**
     * 核心2：将 MySQL 中的规则加载到 Sentinel 核心
     */
    private void loadRuleToSentinel() {
        List<FlowRule> rules = readRuleFromMysql();
        FlowRuleManager.loadRules(rules); // 覆盖 Sentinel 本地规则
        System.out.printf("[Sentinel 拉模式] 规则加载完成：共 %d 条规则%n", rules.size());
    }

    /**
     * 初始化 Druid 连接池（配置基础参数，生产环境需优化）
     */
    private DruidDataSource initDruidDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl(jdbcUrl);
        dataSource.setUsername(dbUsername);
        dataSource.setPassword(dbPassword);
        dataSource.setInitialSize(5); // 初始连接数
        dataSource.setMaxActive(20);  // 最大连接数
        dataSource.setMinIdle(3);     // 最小空闲连接数
        return dataSource;
    }

    // ------------------- 实现 ReadableDataSource 接口（SPI 强制要求） -------------------
    @Override
    public List<FlowRule> readSource() throws Exception {
        // SPI 机制会调用此方法获取最新规则，直接复用 MySQL 读取逻辑
        return readRuleFromMysql();
    }

    @Override
    public void close() throws Exception {
        // 资源销毁：关闭连接池 + 停止定时任务
        if (druidDataSource != null) {
            druidDataSource.close();
        }
        if (scheduler != null && !scheduler.isShutdown()) {
            scheduler.shutdown();
        }
    }
}
 
 
### 4. SPI 注册：让 Sentinel 识别自定义数据源
 
Sentinel 启动时会扫描  META-INF/services  目录下的 SPI 配置文件，自动加载  ReadableDataSource  的实现类。需手动创建配置文件：
 
1. 创建 SPI 配置文件路径：在项目的  src/main/resources  目录下，新建路径  META-INF/services 。
2. 创建 SPI 配置文件：在上述路径下创建文件，文件名必须为  com.alibaba.csp.sentinel.datasource.ReadableDataSource （即  ReadableDataSource  接口的全路径）。
3. 写入自定义 DataSource 全类名：在配置文件中添加自定义 DataSource 的完整包名+类名（确保 Sentinel 能扫描到）：
 
com.example.sentinel.datasource.MysqlPullFlowRuleDataSource
 
 
### 5. Spring Boot 集成：初始化数据源
 
将自定义 DataSource 注册为 Spring Bean，确保项目启动时自动执行构造方法（初始化连接池与定时任务）：
 
package com.example.sentinel.config;

import com.example.sentinel.datasource.MysqlPullFlowRuleDataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SentinelPullConfig {

    /**
     * 注册拉模式数据源为 Spring Bean
     * - initMethod：启动时执行规则加载
     * - destroyMethod：关闭时销毁资源（连接池、定时任务）
     */
    @Bean(initMethod = "loadRuleToSentinel", destroyMethod = "close")
    public MysqlPullFlowRuleDataSource mysqlPullFlowRuleDataSource() {
        return new MysqlPullFlowRuleDataSource();
    }
}
 
 
### 6. 验证：拉模式是否生效
 
1. 启动 Spring Boot 项目：控制台会打印  [Sentinel 拉模式] 规则加载完成：共 1 条规则 ，说明定时任务已启动，首次拉取规则成功。
2. 修改 MySQL 规则：执行 SQL 语句修改阈值（如将 QPS 从 10 改为 5）：
UPDATE sentinel_flow_rule SET count = 5 WHERE resource = '/order/create';
 
3. 等待轮询间隔：5 秒后，控制台会再次打印规则加载日志，说明规则已自动更新。
4. 测试接口限流：用 Postman 或 JMeter 压测  /order/create  接口，当 QPS 超过 5 时，接口会返回 Sentinel 限流提示（如  Blocked by Sentinel (flow limiting) ），验证规则生效。
 
## 四、拉模式的优势与注意事项
 
优势
 
- 规则持久化：规则存储在 MySQL 中，应用重启后自动拉取，解决默认内存模式的痛点。
- 实现成本低：无需依赖配置中心（如 Nacos、ZooKeeper），仅需定时轮询，适合中小项目。
- 扩展灵活：可替换存储介质（如本地文件、Redis），只需修改  readRuleFromMysql()  中的读取逻辑。
 
## 注意事项
 
- 实时性平衡：轮询间隔越小，实时性越高，但会增加数据库压力（建议设置 3-10 秒）。
- 避免重复加载：确保 DataSource 仅初始化一次（如 Spring Bean 单例），避免多定时任务冲突。
- 生产环境优化：
- 数据库连接池参数需根据并发量调整（如最大连接数、超时时间）。
- 增加规则缓存，减少数据库查询次数。
- 对 SQL 执行异常添加重试机制。
 
## 总结
 
Sentinel 拉模式的核心是“SPI 扩展自定义 DataSource + 客户端定时轮询”：
 
1. 通过 SPI 机制让 Sentinel 加载自定义的“MySQL 规则读取逻辑”；
2. 通过定时任务主动拉取规则并注入 Sentinel 核心；
3. 最终实现规则的持久化与自动更新，解决默认内存模式的缺陷。
 
拉模式适合对规则实时性要求不高、不想依赖配置中心的场景，是中小项目实现 Sentinel 规则持久化的优选方案。