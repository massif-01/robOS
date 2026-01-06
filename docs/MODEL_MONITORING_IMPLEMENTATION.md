# 模型状态监控功能实施文档

## 📋 概述

本文档描述了为 robOS 和 rm01OrinStatus 项目添加的模型状态监控功能的完整实现。

## ✅ 已实现功能

### 1. 推理模组端 (rm01OrinStatus)

#### 新增文件
- `src/tegrastats_api/model_monitor.py` - 模型监控核心模块

#### 修改文件
- `src/tegrastats_api/server.py` - 集成模型监控
- `src/tegrastats_api/__init__.py` - 导出新模块

#### 功能特性
- ✅ 监控 3 个 systemd 服务：
  - `dev-llm.service` (主模型)
  - `dev-embedding.service` (嵌入模型)
  - `dev-reranker.service` (重排模型)
- ✅ 通过 `journalctl` 实时跟踪启动日志
- ✅ 识别启动关键节点并计算进度（10%, 25%, 40%, 50%, 75%, 100%）
- ✅ 提取模型名称和 API 端口信息
- ✅ 检测模型是否启用（10秒超时主模型，5秒超时其他模型）
- ✅ 通过 WebSocket 推送 `model_status_update` 事件
- ✅ REST API 端点：
  - `GET /api/model_status` - 获取所有模型状态
  - `GET /api/model_logs/<model_type>?lines=100` - 获取模型日志

### 2. ESP32 端 (robOS)

#### 修改文件
- `components/agx_monitor/include/agx_monitor.h` - 添加模型数据结构
- `components/agx_monitor/agx_monitor.c` - 实现模型数据解析和存储
- `components/web_server/web_server.c` - 添加模型状态 API

#### 功能特性
- ✅ 接收并解析 `model_status_update` WebSocket 事件
- ✅ 线程安全的模型数据存储
- ✅ REST API 端点：
  - `GET /api/model_status` - 代理模型状态数据

### 3. Web 界面

#### 修改文件
- `sdcard/web/index.htm`

#### 功能特性

**折叠功能**
- ✅ 推理模组监控板块可折叠
- ✅ 应用模组监控板块可折叠
- ✅ 模型状态监控板块可折叠
- ✅ 折叠状态保存到 localStorage

**模型拉起状态**
- ✅ 3 个进度条（主模型、嵌入模型、重排模型）
- ✅ 高度与 GPU 监控一致
- ✅ 从左到右循环滚动光线动画（加载时）
- ✅ 显示进度百分比或状态文本
- ✅ 100% 后 5 秒显示模型名称和 API 端口
- ✅ 未启用时显示"未启用XXX模型"

**模型运行日志**
- ✅ 3 个日志窗口（主模型、嵌入模型、重排模型）
- ✅ 每个窗口可独立折叠（默认折叠）
- ✅ 高度为温度监控高度的 2 倍
- ✅ 支持滚动，自动滚动到底部
- ✅ 终端风格显示（黑色背景，绿色文字）
- ✅ 未启用时显示"未启用XXX模型"

**多语言支持**
- ✅ 中文和英文完整翻译

## 🚀 部署步骤

### 1. 部署 rm01OrinStatus（推理模组）

```bash
# SSH 到推理模组
ssh user@10.10.99.98

# 停止服务
sudo systemctl stop tegrastats-api.service

# 拉取最新代码
cd rm01OrinStatus
git pull

# 重启服务
sudo systemctl start tegrastats-api.service

# 查看状态
sudo systemctl status tegrastats-api.service

# 查看日志确认模型监控启动
sudo journalctl -u tegrastats-api.service -f
```

### 2. 部署 robOS（ESP32）

```bash
# 在开发机器上
cd /Users/massif/robOS

# 构建项目
idf.py build

# 烧录到 ESP32
idf.py flash

# 查看串口日志
idf.py monitor
```

### 3. 更新 Web 界面

Web 界面文件 `sdcard/web/index.htm` 会在 ESP32 重启后自动从 SD 卡加载。

## 🧪 测试验证

### 1. 验证推理模组端

```bash
# 测试 API 端点
curl http://10.10.99.98:58090/api/model_status | python -m json.tool

# 预期响应
{
  "timestamp": "2025-01-06T...",
  "llm": {
    "service": "dev-llm.service",
    "model_type": "llm",
    "progress": 100,
    "status_text": "模型：RM-01 LLM | 端口：58000",
    "model_name": "RM-01 LLM",
    "api_port": "58000",
    "is_enabled": true,
    "startup_complete": true
  },
  "embedding": { ... },
  "reranker": { ... }
}
```

### 2. 验证 ESP32 端

```bash
# 从 ESP32 获取模型状态
curl http://10.10.99.97/api/model_status | python -m json.tool
```

### 3. 验证 Web 界面

1. 打开浏览器访问 `http://10.10.99.97`
2. 检查三个板块都有折叠按钮
3. 点击折叠按钮验证折叠功能
4. 滚动到"模型状态监控"板块
5. 验证三个进度条显示正确
6. 点击日志板块标题展开日志窗口
7. 切换语言验证翻译

## 📊 数据流架构

```
推理模组 (10.10.99.98)
  ├─ ModelMonitor
  │   ├─ journalctl监控 → 解析日志 → 识别关键点
  │   └─ 每1秒更新状态
  └─ TegrastatsServer
      ├─ WebSocket: model_status_update 事件
      └─ REST API: /api/model_status

         ↓ (WebSocket)

ESP32 (10.10.99.97)
  ├─ agx_monitor
  │   ├─ 接收 model_status_update
  │   ├─ 解析并存储数据
  │   └─ 线程安全访问
  └─ web_server
      └─ REST API: /api/model_status

         ↓ (HTTP)

Web 界面 (浏览器)
  ├─ 每2秒轮询 /api/model_status
  ├─ 更新进度条
  └─ 更新日志显示
```

## 🔧 配置说明

### 进度检查点

在 `rm01OrinStatus/src/tegrastats_api/model_monitor.py` 中定义：

```python
CHECKPOINTS = [
    (r"Initializing a V1 LLM engine", 10),
    (r"Loading safetensors checkpoint shards:.*100%.*Completed", 25),
    (r"Available KV cache memory", 40),
    (r"Capturing CUDA graphs", 50),
    (r"Graph capturing finished", 75),
    (r"Application startup complete", 100),
]
```

### 超时设置

- 主模型 (llm): 10 秒无日志显示"未启用"
- 嵌入模型 (embedding): 5 秒
- 重排模型 (reranker): 5 秒

### 更新频率

- 推理模组 WebSocket 推送：1 秒
- Web 界面轮询：2 秒
- 日志更新：3 秒

## 🐛 故障排查

### 问题：模型状态不更新

**检查步骤**：
1. 验证推理模组服务运行：`sudo systemctl status tegrastats-api.service`
2. 检查模型服务状态：
   ```bash
   sudo systemctl status dev-llm.service
   sudo systemctl status dev-embedding.service
   sudo systemctl status dev-reranker.service
   ```
3. 查看推理模组日志：`sudo journalctl -u tegrastats-api.service -f`
4. 验证 WebSocket 连接：查看浏览器控制台

### 问题：进度条卡在某个百分比

**原因**：日志关键词不匹配

**解决**：
1. 查看实际的服务日志：`sudo journalctl -u dev-llm.service -f`
2. 对比 `CHECKPOINTS` 中的正则表达式
3. 调整正则表达式以匹配实际日志

### 问题：折叠状态不保存

**原因**：浏览器 localStorage 问题

**解决**：
1. 清除浏览器缓存
2. 检查浏览器控制台是否有错误
3. 验证 `localStorage` 是否可用

## 📝 API 参考

### rm01OrinStatus API

#### GET /api/model_status

返回所有模型的状态。

**响应示例**：
```json
{
  "timestamp": "2025-01-06T12:00:00.000Z",
  "llm": {
    "service": "dev-llm.service",
    "model_type": "llm",
    "progress": 100,
    "status_text": "模型：RM-01 LLM | 端口：58000",
    "model_name": "RM-01 LLM",
    "api_port": "58000",
    "is_enabled": true,
    "startup_complete": true,
    "last_update": 1704542400
  },
  "embedding": { ... },
  "reranker": { ... }
}
```

#### GET /api/model_logs/{model_type}?lines=100

获取指定模型的日志。

**参数**：
- `model_type`: llm | embedding | reranker
- `lines`: 返回行数（默认100，最大1000）

**响应示例**：
```json
{
  "model_type": "llm",
  "logs": [
    "[12:00:00] Initializing a V1 LLM engine",
    "[12:00:05] Loading safetensors checkpoint shards: 100% Completed",
    ...
  ],
  "count": 50,
  "timestamp": "2025-01-06T12:00:00.000Z"
}
```

### robOS API

#### GET /api/model_status

代理推理模组的模型状态数据。

格式与 rm01OrinStatus 相同。

## 🎨 UI 设计说明

### 进度条动画

使用 CSS 关键帧动画实现光线滚动效果：

```css
@keyframes shimmer {
    0% {
        background-position: -1000px 0;
    }
    100% {
        background-position: 1000px 0;
    }
}

.model-status-progress-fill.loading {
    background: linear-gradient(...);
    background-size: 1000px 100%;
    animation: shimmer 2s infinite linear;
}
```

### 折叠动画

使用 `max-height` 过渡实现平滑折叠：

```css
.collapsible-content {
    max-height: 5000px;
    transition: max-height 0.5s ease, opacity 0.3s ease;
}

.collapsible-content.collapsed {
    max-height: 0;
    opacity: 0;
}
```

## 📚 相关文件清单

### rm01OrinStatus
- `src/tegrastats_api/model_monitor.py` ⭐ 新建
- `src/tegrastats_api/server.py` ✏️ 修改
- `src/tegrastats_api/__init__.py` ✏️ 修改

### robOS
- `components/agx_monitor/include/agx_monitor.h` ✏️ 修改
- `components/agx_monitor/agx_monitor.c` ✏️ 修改
- `components/web_server/web_server.c` ✏️ 修改
- `sdcard/web/index.htm` ✏️ 修改

## 🎯 未来改进建议

1. **实时日志流**：通过 WebSocket 实时推送日志，而不是轮询
2. **日志搜索**：添加日志搜索和过滤功能
3. **历史记录**：记录模型启动历史和失败次数
4. **告警功能**：模型启动失败时发送告警
5. **性能优化**：使用虚拟滚动处理大量日志

---

**实施日期**：2025-01-06  
**实施者**：AI Assistant  
**版本**：1.0.0

