#  代码说明文档

## 一、项目概述

本项目是一个基于 Streamlit 的本地知识库上传与 RAG（检索增强生成）问答项目。系统通过向量数据库存储知识文档，并结合大语言模型实现智能问答功能。

## 二、核心模块说明

### 1. config_data.py - 配置文件

**文件路径**: `config_data.py`

**功能**: 存储项目的核心配置参数

**配置项说明**:

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `md5_path` | MD5 去重文件路径 | `./md5.text` |
| `collection_name` | Chroma 向量库集合名称 | `rag` |
| `persist_directory` | 向量库持久化存储目录 | `./chroma_db` |
| `chunk_size` | 文本分割块大小（字符数） | 1000 |
| `chunk_overlap` | 相邻文本块重叠字符数 | 100 |
| `separators` | 文本分割符列表 | `["\n\n","\n",".","!","?","。","！","？"," ",""]` |
| `max_spliter_char_number` | 文本分割阈值 | 1000 |
| `similarity_threshold` | 检索返回的文档数量 | 1 |
| `embedding_model_name` | 嵌入模型名称 | `text-embedding-v4` |
| `chat_model_name` | 聊天模型名称 | `qwen3-max` |
| `session_config` | 会话配置 | `{"configurable": {"session_id": "user_001"}}` |

---

### 2. knowledge_base.py - 知识库服务

**文件路径**: `knowledge_base.py`

**功能**: 处理知识库的读取、文本分割、向量化存储和去重

**核心类**: `KnowledgeBaseService`

**方法说明**:

| 方法 | 说明 | 参数 | 返回值 |
|------|------|------|--------|
| `__init__` | 初始化知识库服务 | - | - |
| `upload_by_str` | 将文本字符串向量化存入向量库 | `data: str, filename: str` | 执行结果字符串 |

**辅助函数**:

| 函数 | 说明 |
|------|------|
| `check_md5` | 检查 MD5 字符串是否已处理过 |
| `save_md5` | 将 MD5 字符串保存到文件 |
| `get_string_md5` | 计算字符串的 MD5 值 |

**核心逻辑**:

1. 检查文本内容的 MD5 是否已存在（去重）
2. 如果文本长度超过阈值，使用 `RecursiveCharacterTextSplitter` 进行分割
3. 添加元数据（来源文件、创建时间、操作人）
4. 将分割后的文本块存入 Chroma 向量库
5. 保存 MD5 到本地文件

---

### 3. vector_stores.py - 向量存储服务

**文件路径**: `vector_stores.py`

**功能**: 提供向量数据库的检索功能封装

**核心类**: `VectorStoreService`

**方法说明**:

| 方法 | 说明 | 参数 | 返回值 |
|------|------|------|--------|
| `__init__` | 初始化向量存储服务 | `embedding: 嵌入模型` | - |
| `get_retriever` | 获取向量检索器 | - | `retriever` 对象 |

**核心逻辑**:

1. 初始化 Chroma 向量库，指定集合名称、嵌入函数和持久化目录
2. 将向量库转换为检索器
3. 设置检索参数 `k`（返回相似度最高的 k 个文档）

---

### 4. rag.py - RAG 链组装

**文件路径**: `rag.py`

**功能**: 构建检索增强生成（RAG）执行链

**核心类**: `RagService`

**方法说明**:

| 方法 | 说明 |
|------|------|
| `__init__` | 初始化 RAG 服务 |
| `__get_chain` | 构建 RAG 执行链 |

**RAG 链执行流程**:

```
用户输入
    ↓
[RunnablePassthrough] 传递输入
    ↓
[检索器] 从向量库检索相关文档
    ↓
[format_document] 格式化文档为字符串
    ↓
[format_for_prompt_template] 格式化提示词输入
    ↓
[ChatPromptTemplate] 应用提示词模板
    ↓
[ChatTongyi] 调用通义千问模型生成回答
    ↓
[StrOutputParser] 解析输出为字符串
    ↓
[RunnableWithMessageHistory] 添加会话历史管理
    ↓
返回回答
```

**提示词模板**:

```python
("system", "以我提供的已知参考资料为主，简洁和专业的回答用户问题。参考资料:{context}。以下是对话历史记录："),
MessagesPlaceholder("history"),
("user", "请回答用户提问：{input}")
```

---

### 5. file_history_store.py - 会话历史存储

**文件路径**: `file_history_store.py`

**功能**: 基于文件的会话历史存储管理

**核心类**: `FileChatMessageHistory` (继承自 `BaseChatMessageHistory`)

**方法说明**:

| 方法 | 说明 | 参数 | 返回值 |
|------|------|------|--------|
| `__init__` | 初始化会话历史存储 | `session_id: str, storage_path: str` | - |
| `add_messages` | 添加消息到历史记录 | `messages: Sequence[BaseMessage]` | `None` |
| `messages` (属性) | 获取消息列表 | - | `list[BaseMessage]` |
| `clear` | 清空历史记录 | - | `None` |

**核心逻辑**:

1. 使用 JSON 文件存储消息历史
2. 消息序列化为字典后写入文件
3. 读取时从文件反序列化为消息对象
4. 支持多个会话 ID，每个会话有独立的文件

---

### 6. app_upload.py - 知识库上传页面

**文件路径**: `app_upload.py`

**功能**: 提供基于 Streamlit 的知识库文件上传界面

**运行命令**: `streamlit run app_upload.py`

**页面功能**:

1. 标题显示："知识库更新服务"
2. 文件上传组件：支持上传 TXT 文件（单文件）
3. 显示文件信息：文件名、格式、大小
4. 读取文件内容并调用 `KnowledgeBaseService` 上传
5. 显示上传结果（成功/重复）

**核心逻辑**:

```python
if uploader_file is not None:
    # 提取文件信息
    file_name = uploader_file.name
    file_size = uploader_file.size / 1024

    # 读取文件内容
    text = uploader_file.getvalue().decode("utf-8")

    # 上传到知识库
    result = st.session_state["service"].upload_by_str(text, file_name)
```

---

### 7. app_chat.py - 智能客服页面

**文件路径**: `app_chat.py`

**功能**: 提供基于 Streamlit 的智能问答界面

**运行命令**: `streamlit run app_chat.py`

**页面功能**:

1. 标题显示："智能客服"
2. 显示聊天历史记录
3. 用户输入框进行提问
4. 流式输出 AI 回答
5. 自动保存聊天历史

**核心逻辑**:

```python
# 初始化 RAG 服务
if "rag" not in st.session_state:
    st.session_state["rag"] = RagService()

# 获取用户输入
prompt = st.chat_input()

if prompt:
    # 调用 RAG 链进行问答
    res_stream = st.session_state["rag"].chain.stream(
        {"input": prompt},
        config.session_config
    )

    # 流式输出回答
    st.chat_message("assistant").write_stream(capture(res_stream, ai_res_list))
```

---

## 三、数据流与架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      Streamlit 前端                          │
├──────────────────────┬──────────────────────────────────────┤
│   app_upload.py      │         app_chat.py                  │
│   (知识库上传)        │         (智能客服)                    │
└──────────┬───────────┴──────────────┬───────────────────────┘
           │                          │
           ▼                          ▼
┌──────────────────────┬──────────────────────────────────────┐
│  KnowledgeBaseService │         RagService                   │
│  (知识库服务)         │         (RAG链服务)                   │
├──────────────────────┼──────────────────────────────────────┤
│  文本分割/向量化       │   检索 → 提示词 → 大模型 → 输出解析   │
└──────────┬───────────┴──────────────┬───────────────────────┘
           │                          │
           ▼                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Chroma 向量数据库                          │
│  (持久化存储 + HNSW 索引)                                     │
└─────────────────────────────────────────────────────────────┘
           ▲                          │
           │                          ▼
┌─────────────────────────────────────────────────────────────┐
│                  通义千问 API (Qwen3-max)                   │
│  (DashScope 阿里云服务)                                      │
└─────────────────────────────────────────────────────────────┘
```

### 问答流程详解

1. **用户提问**: 用户在 `app_chat.py` 界面输入问题
2. **向量检索**: `RagService` 调用 `VectorStoreService` 从 Chroma 向量库中检索相似文档
3. **提示词构建**: 将用户问题、检索到的上下文和历史消息拼接成提示词
4. **模型调用**: 将提示词发送给通义千问模型
5. **回答生成**: 模型生成回答并流式返回
6. **历史保存**: 问答记录保存到 `FileChatMessageHistory`

---

## 四、关键配置说明

### 环境变量

在项目根目录创建 `.env` 文件：

```bash
DASHSCOPE_API_KEY=your_api_key_here
```

### 模型配置

| 模型类型 | 模型名称 | 说明 |
|----------|----------|------|
| 嵌入模型 | `text-embedding-v4` | 将文本转换为向量 |
| 聊天模型 | `qwen3-max` | 通义千问旗舰模型 |

### Chroma 配置

- **集合名称**: `rag`
- **存储目录**: `./chroma_db`
- **检索方式**: HNSW 近似最近邻搜索

---

## 五、文件清单

| 文件 | 说明 |
|------|------|
| `config_data.py` | 配置文件 |
| `knowledge_base.py` | 知识库服务 |
| `vector_stores.py` | 向量存储服务 |
| `rag.py` | RAG 链组装 |
| `file_history_store.py` | 会话历史存储 |
| `app_upload.py` | 知识库上传页面 |
| `app_chat.py` | 智能客服页面 |
| `requirements.txt` | 项目依赖 |
| `.env` | 环境变量配置 |

---

## 六、运行指南

### 1. 安装依赖

```bash
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### 2. 配置 API Key

在 `.env` 文件中配置通义千问 API Key：

```bash
DASHSCOPE_API_KEY=your_api_key_here
```

### 3. 启动服务

```bash
# 启动知识库上传服务
streamlit run app_upload.py

# 启动智能客服
streamlit run app_chat.py
```

### 4. 使用流程

1. 启动 `app_upload.py`，上传 TXT 格式的知识文档
2. 启动 `app_chat.py`，在对话框中提问
3. 系统会自动检索知识库并生成回答
