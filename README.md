# DeepSeek V4 Proxy

解决第三方agent通过OPENAI API Base URL使用 DeepSeek V4 模型时的 `reasoning_content must be passed back` 错误。

## 问题背景

DeepSeek V4 的 Extended Thinking 功能会生成 `reasoning_content` 字段。DeepSeek API 要求在后续对话中，assistant 消息**必须**包含之前的 `reasoning_content`，否则返回 500 错误。

第三方agent通过OPENAI API Base URL使用 DeepSeek V4 模型时，不会存储和回传 `reasoning_content`，导致多轮对话失败。

## 核心功能

- ✅ **自动缓存 & 回传 reasoning_content**：基于 MD5 哈希匹配，多轮对话不再报错
- ✅ **流式输出支持**：不影响第三方agent 的打字机渲染效果 
- ✅ **智能限流**：内置令牌桶算法，防止突发并发请求打满配额
- ✅ **429 自动重试**：最多重试 3 次，间隔 3 秒
- ✅ **工具调用支持**：正确处理 tool_calls 增量累积
- ✅ **双版本选择**：内存缓存版 / SQLite 持久化版

## 版本说明

### 版本 1: `proxy.py` — 内存缓存版

- **缓存方式**：纯内存字典，重启后丢失
- **适用场景**：临时使用、快速验证
- **依赖**：无额外依赖
- **启动**：`start_proxy.bat`

### 版本 2: `proxy-deepseekv4.py` — SQLite 双层缓存版

- **缓存方式**：内存 + SQLite 双层缓存，重启后保留
- **适用场景**：长期使用、生产环境
- **依赖**：无额外依赖（Python 内置 sqlite3）
- **启动**：`start_proxy-deepseekv4.bat`

## 快速开始

### 1. 安装依赖

```bash
pip install -r requirements.txt
```

### 2. 启动代理

**Windows（含 Cloudflare 隧道）：**

```bash
# 内存缓存版
start_proxy.bat

# SQLite 缓存版（无隧道）
start_proxy-deepseekv4.bat
```

**macOS / Linux：**

```bash
chmod +x start_proxy.sh
./start_proxy.sh
```

### 3. 配置 三方agent

1. 在deepseek-v4-proxy\proxy.py 中，修改 `UPSTREAM_URL` 为你需要的BASEURL (OPENAI API Base URL)
2. 在 "Override OpenAI Base URL" 中，填入：
   - 本地访问：`http://localhost:9000/v1`
3. 在 API Key 处填入你的 DeepSeek API Key

### 4. 验证

发送多轮对话，观察是否还有 `reasoning_content` 错误。

## 技术原理

### reasoning_content 回传规则

根据 DeepSeek 官方文档：

| 场景 | 是否需要回传 reasoning_content |
|------|-------------------------------|
| 两个 user 消息之间，assistant **未进行**工具调用 | **无需回传** |
| 两个 user 消息之间，assistant **进行了**工具调用 | **必须回传**（否则返回 500） |

### 代理工作流程

```
客户端
  │
  │  OpenAI 格式请求（缺少 reasoning_content）
  ▼
┌─────────────────────────────────────────┐
│  DeepSeek V4 Proxy                      │
│                                          │
│  1. 遍历 messages，对缺少               │
│     reasoning_content 的 assistant 消息：│
│     - 计算 MD5 哈希（排除该字段）        │
│     - 查询缓存                          │
│     - 命中 → 注入缓存值                 │
│     - 未命中 → 设置为空字符串           │
│                                          │
│  2. 转发请求到 DeepSeek API             │
│                                          │
│  3. 流式收集 reasoning_content          │
│     和 content                          │
│                                          │
│  4. 流结束后，用最终消息的哈希          │
│     作为 key 缓存 reasoning_content     │
│                                          │
│  5. 返回流式响应给客户端                │
└─────────────────────────────────────────┘
  │
  ▼
DeepSeek API（正常返回 200）
```

### 缓存键生成

```python
def msg_key(msg: dict) -> str:
    """生成不包含 reasoning_content 的完整消息体的哈希"""
    m = {k: v for k, v in msg.items() if k != 'reasoning_content'}
    payload = json.dumps(m, sort_keys=True, ensure_ascii=False)
    return hashlib.md5(payload.encode()).hexdigest()
```

## 配置说明

### 修改上游地址

编辑 `proxy.py` 或 `proxy-deepseekv4.py`，修改 `UPSTREAM_URL`：

```python
UPSTREAM_URL = "https://api.deepseek.com/v1/chat/completions"  # 官方地址
# 或
UPSTREAM_URL = "https://api.knox.chat/v1/chat/completions"     # 第三方中转
```

### 调整限流参数

```python
# 修改令牌桶参数
bucket = TokenBucket(rate=5/60.0, capacity=5)  # 每分钟 5 个请求，突发容量 5

# 如需更保守的限流
bucket = TokenBucket(rate=3/60.0, capacity=3)  # 每分钟 3 个请求
```

## 与 CC-Switch 集成

本代理也可与 [CC-Switch](https://github.com/farion1231/cc-switch) 配合使用：

1. 启动本代理
2. 在 CC-Switch 中添加 Provider：
   - **名称**：`DeepSeek V4 Pro (Local Proxy)`
   - **Base URL**：`http://localhost:9000/v1`
   - **API Key**：你的 DeepSeek API Key
   - **Model**：`deepseek-v4-pro`

## 许可证

MIT License

## 致谢

- [cursor-deepseek-v4-proxy](https://github.com/wustghj/cursor-deepseek-v4-proxy) — 原始项目
- [DeepSeek API Docs](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode) — 官方文档
