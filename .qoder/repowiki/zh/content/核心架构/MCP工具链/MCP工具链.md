# MCP工具链架构文档

<cite>
**本文档引用的文件**
- [start_mcp_services.py](file://agent_tools/start_mcp_services.py)
- [tool_trade.py](file://agent_tools/tool_trade.py)
- [tool_get_price_local.py](file://agent_tools/tool_get_price_local.py)
- [tool_jina_search.py](file://agent_tools/tool_jina_search.py)
- [tool_math.py](file://agent_tools/tool_math.py)
- [general_tools.py](file://tools/general_tools.py)
- [price_tools.py](file://tools/price_tools.py)
- [base_agent.py](file://agent/base_agent/base_agent.py)
- [base_agent_astock.py](file://agent/base_agent_astock/base_agent_astock.py)
- [default_config.json](file://configs/default_config.json)
</cite>

## 目录
1. [项目概述](#项目概述)
2. [MCP协议架构](#mcp协议架构)
3. [工具链组件分析](#工具链组件分析)
4. [服务启动与管理](#服务启动与管理)
5. [客户端集成](#客户端集成)
6. [数据流与交互](#数据流与交互)
7. [性能与扩展性](#性能与扩展性)
8. [故障排除指南](#故障排除指南)
9. [总结](#总结)

## 项目概述

AI-Trader是一个基于模型上下文协议(MCP)的智能交易系统，通过FastMCP框架将多种交易功能以HTTP服务形式暴露给AI代理调用。该系统采用模块化设计，支持US股票和中国A股市场的自动化交易，为AI模型提供完整的工具链支持。

### 核心特性

- **完全自主决策**：AI代理独立完成市场分析、决策制定和执行
- **纯工具驱动架构**：基于MCP协议的标准工具调用机制
- **多模型竞争环境**：支持GPT、Claude、Qwen等AI模型的公平竞争
- **实时性能分析**：全面的交易记录、仓位监控和盈亏分析
- **智能市场情报**：集成Jina AI进行实时市场新闻检索

## MCP协议架构

### 协议基础

MCP（Model Context Protocol）是专门为AI代理与外部工具交互设计的协议标准。在AI-Trader中，MCP协议通过以下方式实现：

```mermaid
graph TB
subgraph "AI代理层"
Agent[AI代理]
LangChain[LangChain框架]
end
subgraph "MCP客户端层"
MCPClient[MultiServerMCPClient]
Config[MCP配置]
end
subgraph "服务层"
MathSvc[Math服务<br/>端口8000]
SearchSvc[Search服务<br/>端口8001]
TradeSvc[Trade服务<br/>端口8002]
PriceSvc[Price服务<br/>端口8003]
end
subgraph "工具层"
MathTool[数学计算工具]
SearchTool[搜索工具]
TradeTool[交易工具]
PriceTool[价格查询工具]
end
subgraph "数据层"
Position[仓位数据<br/>position.jsonl]
MergedData[OHLCV数据<br/>merged.jsonl]
LogFiles[日志文件]
end
Agent --> LangChain
LangChain --> MCPClient
MCPClient --> Config
Config --> MathSvc
Config --> SearchSvc
Config --> TradeSvc
Config --> PriceSvc
MathSvc --> MathTool
SearchSvc --> SearchTool
TradeSvc --> TradeTool
PriceSvc --> PriceTool
TradeTool --> Position
PriceTool --> MergedData
MathTool --> LogFiles
SearchTool --> LogFiles
```

**图表来源**
- [start_mcp_services.py](file://agent_tools/start_mcp_services.py#L20-L40)
- [base_agent.py](file://agent/base_agent/base_agent.py#L288-L315)

### FastMCP框架集成

每个工具都使用FastMCP框架作为HTTP传输层，提供标准化的服务接口：

```mermaid
classDiagram
class FastMCP {
+string name
+dict tools
+run(transport, port)
+tool()
}
class MathService {
+add(a : float, b : float) float
+multiply(a : float, b : float) float
}
class TradeService {
+buy(symbol : str, amount : int) dict
+sell(symbol : str, amount : int) dict
+_position_lock(signature : str)
+_get_today_buy_amount(symbol : str, today_date : str, signature : str) int
}
class PriceService {
+get_price_local(symbol : str, date : str) dict
+get_price_local_daily(symbol : str, date : str) dict
+get_price_local_hourly(symbol : str, date : str) dict
+_workspace_data_path(filename : str, symbol : str) Path
+_validate_date_daily(date_str : str)
+_validate_date_hourly(date_str : str)
}
class SearchService {
+get_information(query : str) str
+parse_date_to_standard(date_str : str) str
+WebScrapingJinaTool
}
FastMCP <|-- MathService
FastMCP <|-- TradeService
FastMCP <|-- PriceService
FastMCP <|-- SearchService
```

**图表来源**
- [tool_math.py](file://agent_tools/tool_math.py#L10-L15)
- [tool_trade.py](file://agent_tools/tool_trade.py#L20-L25)
- [tool_get_price_local.py](file://agent_tools/tool_get_price_local.py#L15-L20)
- [tool_jina_search.py](file://agent_tools/tool_jina_search.py#L120-L125)

**章节来源**
- [start_mcp_services.py](file://agent_tools/start_mcp_services.py#L1-L50)
- [tool_math.py](file://agent_tools/tool_math.py#L1-L45)

## 工具链组件分析

### 数学计算工具 (tool_math.py)

数学计算工具提供基础的算术运算能力，支持整数和浮点数操作。

#### 核心功能

- **加法运算**：`add(a: float, b: float) -> float`
- **乘法运算**：`multiply(a: float, b: float) -> float`

#### 实现特点

- 使用`@mcp.tool()`装饰器注册为MCP工具
- 支持类型注解确保参数正确性
- 返回值自动转换为浮点数

#### 配置端口

默认监听端口：8000

**章节来源**
- [tool_math.py](file://agent_tools/tool_math.py#L15-L45)

### 交易工具 (tool_trade.py)

交易工具实现了完整的买入/卖出交易逻辑，包括交易验证、持仓更新和资金计算。

#### 交易流程

```mermaid
flowchart TD
Start([开始交易]) --> GetEnv[获取环境变量]
GetEnv --> ValidateSymbol[验证股票代码格式]
ValidateSymbol --> CheckMarket{检查市场类型}
CheckMarket --> |中国A股| ValidateLotSize[验证100股整数倍]
CheckMarket --> |美股| DirectCheck[直接检查]
ValidateLotSize --> LoadPosition[加载当前仓位]
DirectCheck --> LoadPosition
LoadPosition --> GetPrice[获取当日开盘价]
GetPrice --> ValidateBalance{验证资金余额}
ValidateBalance --> |不足| ReturnError[返回错误信息]
ValidateBalance --> |充足| UpdatePosition[更新仓位]
UpdatePosition --> WriteLog[写入交易日志]
WriteLog --> UpdateConfig[更新配置标记]
UpdateConfig --> ReturnPosition[返回新仓位]
ReturnError --> End([结束])
ReturnPosition --> End
```

**图表来源**
- [tool_trade.py](file://agent_tools/tool_trade.py#L40-L120)
- [tool_trade.py](file://agent_tools/tool_trade.py#L200-L280)

#### 关键特性

1. **多市场支持**：自动识别US股票和中国A股市场
2. **交易规则验证**：
   - 美股：最小交易单位为1股
   - 中国A股：必须为100股的整数倍
   - T+1规则：当日买入的股票次日才能卖出
3. **并发安全**：使用文件锁确保仓位更新的原子性
4. **日志记录**：详细的交易记录保存到JSONL文件

#### 文件锁机制

```mermaid
sequenceDiagram
participant Client as 客户端请求
participant Lock as 文件锁
participant Position as 仓位数据
Client->>Lock : 请求写入锁
Lock->>Lock : fcntl.flock(LOCK_EX)
Lock-->>Client : 锁定成功
Client->>Position : 读取当前仓位
Position-->>Client : 返回仓位数据
Client->>Position : 更新仓位数据
Client->>Position : 写入交易记录
Client->>Lock : 释放写入锁
Lock->>Lock : fcntl.flock(LOCK_UN)
Lock-->>Client : 解锁完成
```

**图表来源**
- [tool_trade.py](file://agent_tools/tool_trade.py#L25-L40)

**章节来源**
- [tool_trade.py](file://agent_tools/tool_trade.py#L1-L372)

### 本地价格查询工具 (tool_get_price_local.py)

本地价格查询工具从本地merged.jsonl文件查询OHLCV价格数据，支持日线和小时线数据。

#### 数据格式支持

- **日线数据**：`YYYY-MM-DD`格式（如'2025-10-30'）
- **小时线数据**：`YYYY-MM-DD HH:MM:SS`格式（如'2025-10-30 14:30:00'）

#### 查询逻辑

```mermaid
flowchart TD
Input[输入: symbol, date] --> DetectFormat{检测日期格式}
DetectFormat --> |包含空格或'T'| Hourly[小时线查询]
DetectFormat --> |仅日期| Daily[日线查询]
Hourly --> ValidateHourly[验证小时格式]
Daily --> ValidateDaily[验证日线格式]
ValidateHourly --> LoadFile[加载merged.jsonl]
ValidateDaily --> LoadFile
LoadFile --> ParseJSON[解析JSONL文件]
ParseJSON --> MatchSymbol{匹配股票代码}
MatchSymbol --> |未匹配| NextRecord[下一个记录]
MatchSymbol --> |匹配| MatchDate{匹配日期}
MatchDate --> |未找到| ReturnError[返回错误]
MatchDate --> |找到| CheckToday{是否今日}
CheckToday --> |是| TodayOHLCV[返回今日OHLCV]
CheckToday --> |否| HistoricalOHLCV[返回历史OHLCV]
NextRecord --> ParseJSON
ReturnError --> End([结束])
TodayOHLCV --> End
HistoricalOHLCV --> End
```

**图表来源**
- [tool_get_price_local.py](file://agent_tools/tool_get_price_local.py#L50-L80)
- [tool_get_price_local.py](file://agent_tools/tool_get_price_local.py#L85-L150)

#### 市场类型自动检测

工具根据股票代码自动检测市场类型：

- **中国A股**：代码以`.SH`或`.SZ`结尾
- **美股**：默认市场类型

#### 数据访问控制

- **未来数据限制**：当日数据无法获取实时高低价和成交量
- **历史数据完整**：历史交易日数据包含完整的OHLCV信息

**章节来源**
- [tool_get_price_local.py](file://agent_tools/tool_get_price_local.py#L1-L285)

### Jina搜索工具 (tool_jina_search.py)

Jina搜索工具集成了Jina AI服务，提供市场情报搜索和网页内容抓取功能。

#### 搜索流程

```mermaid
sequenceDiagram
participant Agent as AI代理
participant Search as 搜索工具
participant JinaSearch as Jina搜索API
participant JinaScrape as Jina抓取API
participant Filter as 日期过滤器
Agent->>Search : get_information(query)
Search->>JinaSearch : 搜索关键词
JinaSearch-->>Search : 返回URL列表
Search->>Filter : 过滤未来日期内容
Filter-->>Search : 返回过滤后URL
Search->>JinaScrape : 抓取网页内容
JinaScrape-->>Search : 返回结构化内容
Search-->>Agent : 返回格式化结果
```

**图表来源**
- [tool_jina_search.py](file://agent_tools/tool_jina_search.py#L130-L150)
- [tool_jina_search.py](file://agent_tools/tool_jina_search.py#L155-L200)

#### 日期处理功能

支持多种日期格式的标准化处理：

- **相对时间**：`4 hours ago`, `1 day ago`, `2 weeks ago`
- **ISO格式**：`2025-10-01T08:19:28+00:00`
- **常见格式**：`May 31, 2025`, `2025-10-01`

#### 内容过滤机制

- **时间过滤**：只保留早于当前交易日的信息
- **质量控制**：随机选择高质量内容进行抓取
- **错误处理**：完善的异常捕获和错误报告

**章节来源**
- [tool_jina_search.py](file://agent_tools/tool_jina_search.py#L1-L281)

## 服务启动与管理

### 启动管理器架构

MCPServiceManager类负责所有MCP服务的启动、监控和管理。

```mermaid
classDiagram
class MCPServiceManager {
+dict services
+bool running
+dict ports
+dict service_configs
+Path log_dir
+__init__()
+start_all_services()
+start_service(service_id, config)
+check_service_health(service_id)
+check_all_services()
+stop_all_services()
+keep_alive()
+signal_handler(signum, frame)
+is_port_available(port)
+check_port_conflicts()
+print_service_info()
+status()
}
class ServiceConfig {
+string script
+string name
+int port
}
MCPServiceManager --> ServiceConfig : manages
```

**图表来源**
- [start_mcp_services.py](file://agent_tools/start_mcp_services.py#L15-L45)

### 端口配置与冲突检测

#### 默认端口分配

| 服务名称 | 默认端口 | 环境变量 |
|---------|---------|---------|
| Math | 8000 | MATH_HTTP_PORT |
| Search | 8001 | SEARCH_HTTP_PORT |
| Trade | 8002 | TRADE_HTTP_PORT |
| Price | 8003 | GETPRICE_HTTP_PORT |

#### 端口冲突处理

```mermaid
flowchart TD
Start([启动服务]) --> CheckPorts[检查端口可用性]
CheckPorts --> HasConflict{存在冲突?}
HasConflict --> |否| StartServices[启动所有服务]
HasConflict --> |是| AskUser[询问用户处理方式]
AskUser --> UserChoice{用户选择}
UserChoice --> |自动查找| FindAvailable[查找可用端口]
UserChoice --> |手动处理| ManualFix[手动解决冲突]
FindAvailable --> UpdatePorts[更新端口配置]
UpdatePorts --> StartServices
ManualFix --> Fail[启动失败]
StartServices --> Monitor[监控服务状态]
Monitor --> Healthy{服务健康?}
Healthy --> |是| Success[启动成功]
Healthy --> |否| Retry[重试或失败]
Success --> End([结束])
Fail --> End
Retry --> End
```

**图表来源**
- [start_mcp_services.py](file://agent_tools/start_mcp_services.py#L60-L120)

### 服务监控与健康检查

#### 健康检查机制

1. **进程状态检查**：验证子进程是否正常运行
2. **端口响应检查**：测试HTTP服务是否可访问
3. **自动重启**：检测到服务异常时自动重启

#### 日志管理

- **日志文件**：每个服务单独的日志文件
- **日志轮转**：支持日志文件大小管理和清理
- **错误追踪**：详细的错误信息记录

**章节来源**
- [start_mcp_services.py](file://agent_tools/start_mcp_services.py#L1-L294)

## 客户端集成

### MultiServerMCPClient配置

AI代理通过MultiServerMCPClient连接到各个MCP服务，实现工具的动态发现和调用。

#### 配置结构

```mermaid
graph LR
subgraph "MCP客户端配置"
Config[MCP配置字典]
subgraph "服务配置"
MathConfig[数学服务<br/>port: 8000]
SearchConfig[搜索服务<br/>port: 8001]
TradeConfig[交易服务<br/>port: 8002]
PriceConfig[价格服务<br/>port: 8003]
end
Config --> MathConfig
Config --> SearchConfig
Config --> TradeConfig
Config --> PriceConfig
end
subgraph "服务发现"
Discovery[工具发现]
Registration[工具注册]
ToolList[工具列表]
end
MathConfig --> Discovery
SearchConfig --> Discovery
TradeConfig --> Discovery
PriceConfig --> Discovery
Discovery --> Registration
Registration --> ToolList
```

**图表来源**
- [base_agent.py](file://agent/base_agent/base_agent.py#L288-L315)
- [base_agent_astock.py](file://agent/base_agent_astock/base_agent_astock.py#L237-L272)

#### 工具注册流程

```mermaid
sequenceDiagram
participant Agent as AI代理
participant Client as MultiServerMCPClient
participant MathSvc as Math服务
participant TradeSvc as Trade服务
participant SearchSvc as Search服务
participant PriceSvc as Price服务
Agent->>Client : 初始化客户端
Client->>MathSvc : 发送工具发现请求
MathSvc-->>Client : 返回数学工具列表
Client->>TradeSvc : 发送工具发现请求
TradeSvc-->>Client : 返回交易工具列表
Client->>SearchSvc : 发送工具发现请求
SearchSvc-->>Client : 返回搜索工具列表
Client->>PriceSvc : 发送工具发现请求
PriceSvc-->>Client : 返回价格工具列表
Client->>Client : 合并所有工具
Client-->>Agent : 返回完整工具列表
Agent->>Agent : 注册到LangChain
```

**图表来源**
- [base_agent.py](file://agent/base_agent/base_agent.py#L313-L341)

### LangChain集成

#### 工具注册机制

AI代理将MCP工具注册到LangChain框架中，实现：

- **工具调用链**：支持复杂的工具组合调用
- **参数验证**：自动验证工具调用参数
- **错误处理**：统一的错误处理和恢复机制

#### 动态工具发现

- **运行时发现**：服务启动后自动发现可用工具
- **版本兼容**：支持工具版本的向后兼容
- **热更新**：服务重启时不丢失工具注册信息

**章节来源**
- [base_agent.py](file://agent/base_agent/base_agent.py#L313-L341)
- [base_agent_astock.py](file://agent/base_agent_astock/base_agent_astock.py#L270-L286)

## 数据流与交互

### 交易数据流

```mermaid
flowchart TD
subgraph "AI代理决策"
Decision[交易决策]
Action[执行动作]
end
subgraph "工具调用"
BuyTool[买入工具]
SellTool[卖出工具]
PriceTool[价格查询]
SearchTool[搜索工具]
end
subgraph "数据存储"
PositionFile[position.jsonl]
LogFile[日志文件]
ConfigFile[配置文件]
end
subgraph "外部服务"
JinaAPI[Jina AI API]
LocalData[本地数据文件]
end
Decision --> BuyTool
Decision --> SellTool
Decision --> PriceTool
Decision --> SearchTool
BuyTool --> PositionFile
SellTool --> PositionFile
BuyTool --> ConfigFile
SellTool --> ConfigFile
PriceTool --> LocalData
SearchTool --> JinaAPI
BuyTool --> LogFile
SellTool --> LogFile
PriceTool --> LogFile
SearchTool --> LogFile
```

**图表来源**
- [tool_trade.py](file://agent_tools/tool_trade.py#L334-L370)
- [tool_get_price_local.py](file://agent_tools/tool_get_price_local.py#L280-L285)

### 状态同步机制

#### 仓位管理

- **原子操作**：使用文件锁确保仓位更新的原子性
- **版本控制**：每次交易生成唯一的操作ID
- **数据持久化**：交易记录永久保存到JSONL文件

#### 配置同步

- **运行时配置**：通过`.runtime_env.json`文件持久化配置
- **环境变量**：支持环境变量覆盖默认配置
- **签名隔离**：不同AI模型使用独立的配置空间

**章节来源**
- [tool_trade.py](file://agent_tools/tool_trade.py#L334-L370)
- [general_tools.py](file://tools/general_tools.py#L40-L70)

## 性能与扩展性

### 并发处理能力

#### 服务架构

- **独立进程**：每个MCP服务运行在独立进程中
- **无状态设计**：服务之间不共享状态，提高可靠性
- **负载均衡**：支持多个相同类型服务实例

#### 性能优化

- **连接池**：HTTP客户端连接复用
- **缓存机制**：频繁访问的数据缓存
- **异步处理**：非阻塞的网络请求处理

### 扩展性设计

#### 模块化架构

- **插件式工具**：新工具可以轻松添加到系统中
- **配置驱动**：通过配置文件管理服务行为
- **接口标准化**：统一的MCP工具接口规范

#### 监控与告警

- **健康检查**：定期检查服务状态
- **性能指标**：关键性能指标监控
- **错误追踪**：详细的错误日志和追踪

## 故障排除指南

### 常见问题诊断

#### 服务启动失败

**症状**：MCP服务无法启动或立即退出

**排查步骤**：
1. 检查端口占用情况
2. 验证Python依赖安装
3. 查看服务日志文件
4. 检查环境变量配置

**解决方案**：
- 使用`python start_mcp_services.py status`检查服务状态
- 手动指定可用端口
- 重新安装依赖包

#### 工具调用失败

**症状**：AI代理无法调用特定工具

**排查步骤**：
1. 检查MCP服务是否正常运行
2. 验证网络连接
3. 检查工具接口定义
4. 查看客户端错误日志

**解决方案**：
- 重启相关MCP服务
- 检查防火墙设置
- 验证API密钥配置

#### 交易执行异常

**症状**：买入/卖出操作失败

**排查步骤**：
1. 检查账户资金余额
2. 验证股票代码有效性
3. 检查交易规则约束
4. 查看仓位文件权限

**解决方案**：
- 补充账户资金
- 使用正确的股票代码格式
- 等待T+1交易周期
- 修复文件权限问题

### 性能调优建议

#### 服务性能优化

- **内存管理**：定期清理临时文件和缓存
- **磁盘I/O**：使用SSD存储交易数据文件
- **网络优化**：调整HTTP连接超时设置

#### AI代理优化

- **工具缓存**：缓存常用的工具调用结果
- **批量处理**：合并多个小的工具调用
- **错误恢复**：实现智能的错误重试机制

**章节来源**
- [start_mcp_services.py](file://agent_tools/start_mcp_services.py#L200-L250)

## 总结

AI-Trader的MCP工具链架构展现了现代AI交易系统的最佳实践。通过FastMCP框架和Model Context Protocol，系统实现了：

### 架构优势

1. **模块化设计**：每个工具独立部署，便于维护和扩展
2. **标准化接口**：统一的MCP协议确保工具间的互操作性
3. **高并发支持**：独立进程架构支持大规模并发访问
4. **可靠的数据管理**：文件锁机制确保交易数据的一致性

### 技术创新

- **智能市场情报**：集成Jina AI实现实时市场搜索
- **跨市场支持**：同时支持US股票和中国A股市场
- **自动化程度高**：完全无需人工干预的自主交易
- **透明度高**：详细的交易记录和决策过程追踪

### 应用价值

该架构不仅适用于AI交易系统，还可作为其他AI工具链项目的参考模板。通过标准化的MCP协议，开发者可以快速构建自己的AI工具生态系统，实现各种复杂的业务场景。

未来的扩展方向包括：
- 支持更多金融产品类型
- 集成更多的外部数据源
- 实现实时风险管理系统
- 提供更丰富的分析工具

这套MCP工具链为AI驱动的金融应用提供了坚实的技术基础，展示了人工智能在复杂商业环境中的巨大潜力。