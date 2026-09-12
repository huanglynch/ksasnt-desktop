### API使用说明

以下是对提供的代码中核心API（`Conversation`类）的使用说明，重点介绍如何通过Python调用该API与文档进行交互式对话，并附带一个极简的命令行文档对话工具的示例实现。

#### 核心类和功能
`Conversation`类是与知识库交互的主要入口，允许用户基于文档内容提出问题并获取答案。它依赖于`KnowledgeBaseManager`、`DocumentProcessor`、`ExtractionModule`、`IntegrationModule`和`AnswerModule`来完成文档加载、内容提取、整合和答案生成。

##### 关键方法
1. **`Conversation.__init__(kb_manager: KnowledgeBaseManager, kb_ids: List[str])`**
   - **功能**: 初始化一个对话实例，加载指定的知识库（knowledge base）。
   - **参数**:
     - `kb_manager`: `KnowledgeBaseManager`实例，用于管理知识库的创建、加载和删除。
     - `kb_ids`: 知识库ID列表，指定要加载的知识库。
   - **注意**: 知识库必须预先通过`KnowledgeBaseManager`创建并保存。

2. **`Conversation.chat(query: str, stream: bool = False) -> Union[Dict, Iterator[Dict]]`**
   - **功能**: 根据用户查询，从知识库中提取相关信息并生成答案。
   - **参数**:
     - `query`: 用户的查询字符串（例如，“文档中的主要主题是什么？”）。
     - `stream`: 是否以流式方式返回答案（`True`返回迭代器，`False`返回单一结果）。
   - **返回**:
     - 如果`stream=False`，返回`Dict`格式：`{"answer": str, "sources": List[Dict]}`，包含答案和来源。
     - 如果`stream=True`，返回`Iterator[Dict]`，逐步生成答案片段和来源。
   - **来源格式**: 每个来源为`{"extracted": str, "source": str}`，`source`形如`doc_name:start_line-end_line`。

3. **`KnowledgeBaseManager`相关方法**
   - `create_kb_id(kb_name: str) -> str`: 根据知识库名称生成唯一的MD5 ID。
   - `save_knowledge_base(kb_id: str, chunks: List[DocumentChunk], kb_name: str, doc_paths: List[str])`: 保存知识库。
   - `load_knowledge_base(kb_id: str) -> List[DocumentChunk]`: 加载知识库的文档分片。
   - `list_kbs() -> List[Dict[str, str]]`: 列出所有可用知识库，返回`[{"id": str, "name": str}, ...]`。
   - `delete_kb(kb_id: str)`: 删除指定知识库。

4. **`DocumentProcessor.load_documents(doc_paths: List[str], query: str) -> List[DocumentChunk]`**
   - **功能**: 加载并分片文档，支持预过滤以减少无关文档的处理。
   - **参数**:
     - `doc_paths`: 文档路径列表。
     - `query`: 查询字符串，用于预过滤（`"dummy_query"`跳过过滤）。
   - **返回**: 包含`DocumentChunk`对象的列表，每个对象表示一个文档分片。

##### 配置管理
- `ConfigLoader`类管理配置，加载`ksasnt_config.json`和`llmcall.env`。
- 支持的配置项包括：
  - `max_docs`: 最大文档数（默认10）。
  - `file_types`: 支持的文件类型（默认`["md", "txt"]`）。
  - `context_length`: 最大上下文长度（默认50000字符）。
  - `concurrency`: 并行处理线程数（默认4）。
  - `qps`: 查询每秒限制（默认1.0）。
  - `chunk_size`: 文档分片大小（默认40000字符）。
  - `overlap_ratio`: 分片重叠比例（默认0.2）。
  - `kb_dir`: 知识库存储目录（默认`knowledge_bases`）。
  - `conversation_modes`: 对话模式（`query`、`qa`、`auto`），控制提示模板。
  - `default_mode`: 默认对话模式（默认`auto`）。

##### LLM依赖
- 代码依赖`LLMAPI`（未提供具体实现），假设其提供`call_llm(messages: List[Dict], stream: bool)`方法，用于调用语言模型。
- `LLMAPI`需要支持：
  - **输入**: `messages`为消息列表，格式如`[{"role": "system", "content": str}, {"role": "user", "content": str}]`。
  - **输出**: 非流式返回字符串，流式返回迭代器。
  - **配置**: 通过`llmcall.env`或默认参数（如`OLLAMA_URL`、`LLM`、`TEMPERATURE`等）初始化。

#### 使用流程
1. **初始化配置和知识库管理器**:
   - 创建`ConfigLoader`实例加载配置。
   - 使用`KnowledgeBaseManager`管理知识库。
2. **加载文档并创建知识库**:
   - 使用`DocumentProcessor`加载和分片文档。
   - 通过`KnowledgeBaseManager.save_knowledge_base`保存知识库。
3. **初始化对话**:
   - 创建`Conversation`实例，指定知识库ID。
4. **交互式查询**:
   - 调用`Conversation.chat`处理用户查询，获取答案和来源。
5. **管理知识库**:
   - 使用`KnowledgeBaseManager.list_kbs`查看可用知识库。
   - 使用`delete_kb`删除不需要的知识库。

#### 注意事项
- **文件路径**: 确保文档路径有效，支持的格式为`md`和`txt`（可通过配置扩展）。
- **知识库存储**: 知识库以`.pkl`（分片）和`.json`（元信息）格式存储在`kb_dir`中。
- **错误处理**: 代码包含详细的日志记录（`ksasnt.log`），便于调试。
- **并发限制**: `concurrency`和`qps`控制并行处理和LLM调用频率，避免过载。
- **缓存**: `Conversation`类使用线程安全的缓存存储提取结果，提升重复查询效率。

---


### 配置文件模板

`ksasnt_config.json` 是主配置文件，用于控制 `ConfigLoader` 类的行为。以下是一个完整的配置文件模板，包含所有可能的配置项及其默认值，供用户参考。文件应保存为 JSON 格式，编码为 UTF-8。

```json
{
  "max_docs": 10,
  "file_types": ["md", "txt"],
  "context_length": 50000,
  "concurrency": 4,
  "qps": 1.0,
  "chunk_size": 40000,
  "overlap_ratio": 0.2,
  "kb_dir": "knowledge_bases",
  "default_mode": "auto",
  "conversation_modes": {
    "query": {
      "name": "精确查询",
      "description": "适用于查找特定信息、数据、事实",
      "prefilter_prompt_template": "You are an expert in determining document relevance for precise information retrieval.\nGiven the query: \"{query}\"\nFrom the following document: \"{doc_content}\"\nDetermine if the document contains specific information, data, or facts related to the query.\nOutput strictly in JSON format: {\"relevant\": true}\nIf not related, return {\"relevant\": false}\nBe precise - only mark as relevant if the document contains specific information that directly answers the query.",
      "extraction_prompt_template": "You are an expert in extracting precise information from text.\nGiven the query: \"{query}\"\nFrom the following text: \"{chunk_content}\"\nExtract specific facts, data, or information that directly answers the query. Focus on precision and accuracy.\nOutput strictly in JSON format: {\"extracted\": \"precise information that answers the query\", \"source\": \"{source}\"}\nIf no relevant content, return {\"extracted\": \"\", \"source\": \"{source}\"}\nOnly extract information that directly addresses the query.",
      "integration_prompt_template": "You are an expert in synthesizing precise information.\nGiven multiple extracted snippets: {extractions_json}\nCombine the information to provide a comprehensive and accurate answer (max {max_tokens} tokens).\nFocus on factual accuracy and completeness. Remove duplicates and organize information logically.\nOutput in structured text, citing sources (e.g., [doc1:10-15]).",
      "answer_prompt_template": "You are a precise information assistant.\nContext: {context}\nUser query: {query}\nProvide a direct, factual answer based on the context. Be specific and cite all relevant sources (e.g., [doc1:10-15]).\nIf the context doesn't contain the exact information, clearly state what is available and what is missing."
    },
    "qa": {
      "name": "智能问答",
      "description": "适用于概念解释、原理分析、综合讨论",
      "prefilter_prompt_template": "You are an expert in determining document relevance for comprehensive Q&A.\nGiven the query: \"{query}\"\nFrom the following document: \"{doc_content}\"\nDetermine if the document contains information that could help answer the question, even if not directly.\nOutput strictly in JSON format: {\"relevant\": true}\nIf not related, return {\"relevant\": false}\nConsider broader context and related concepts that might be useful for a comprehensive answer.",
      "extraction_prompt_template": "You are an expert in extracting comprehensive information for Q&A.\nGiven the query: \"{query}\"\nFrom the following text: \"{chunk_content}\"\nExtract information that helps answer the question, including context, explanations, and related concepts.\nOutput strictly in JSON format: {\"extracted\": \"comprehensive information for answering the question\", \"source\": \"{source}\"}\nIf no relevant content, return {\"extracted\": \"\", \"source\": \"{source}\"}\nInclude explanatory content and context that aids understanding.",
      "integration_prompt_template": "You are an expert in synthesizing information for comprehensive answers.\nGiven multiple extracted snippets: {extractions_json}\nCreate a well-structured, comprehensive response (max {max_tokens} tokens).\nProvide explanations, context, and organize information in a logical flow.\nOutput in clear, educational text, citing sources (e.g., [doc1:10-15]).",
      "answer_prompt_template": "You are an intelligent Q&A assistant.\nContext: {context}\nUser question: {query}\nProvide a comprehensive, well-explained answer based on the context. Include background information, explanations, and relevant details.\nCite sources appropriately (e.g., [doc1:10-15]). If the context is incomplete, provide the best possible answer and indicate what additional information might be helpful."
    },
    "auto": {
      "name": "智能自适应",
      "description": "根据问题类型自动选择最佳处理方式",
      "prefilter_prompt_template": "You are an expert in determining document relevance.\nGiven the query: \"{query}\"\nFrom the following document: \"{doc_content}\"\nDetermine if the document is semantically related to the query.\nOutput strictly in JSON format: {\"relevant\": true}\nIf not related, return {\"relevant\": false}\nConsider both specific information needs and broader conceptual relevance.",
      "extraction_prompt_template": "You are an expert in extracting relevant information from text.\nGiven the query: \"{query}\"\nFrom the following text: \"{chunk_content}\"\nExtract key points semantically related to the query. Adapt your extraction style based on the query type:\n- For specific questions: focus on precise facts and data\n- For conceptual questions: include explanations and context\n- For general inquiries: provide comprehensive relevant information\nOutput strictly in JSON format: {\"extracted\": \"relevant content adapted to query type\", \"source\": \"{source}\"}\nIf no relevant content, return {\"extracted\": \"\", \"source\": \"{source}\"}}",
      "integration_prompt_template": "You are an expert in synthesizing information adaptively.\nGiven multiple extracted snippets: {extractions_json}\nAnalyze the query type and synthesize information accordingly (max {max_tokens} tokens):\n- For specific queries: organize facts systematically\n- For conceptual queries: provide structured explanations\n- For general queries: create comprehensive overviews\nOutput in appropriate style, citing sources (e.g., [doc1:10-15]).",
      "answer_prompt_template": "You are an adaptive AI assistant.\nContext: {context}\nUser question: {query}\nAnalyze the question type and provide an appropriate response:\n- For specific information requests: give direct, factual answers\n- For conceptual questions: provide explanatory, educational responses\n- For general inquiries: offer comprehensive, well-structured information\nAlways cite relevant sources (e.g., [doc1:10-15]) and adapt your communication style to best serve the user's needs."
    }
  }
}
```

#### 配置文件说明
- **存储路径**: 默认存储在 `get_data_dir()` 返回的目录（如 `~/Documents/KSASNT/ksasnt_config.json`），或通过 `get_config_path` 查找。
- **字段说明**:
  - `max_docs`: 最大处理文档数，防止资源过载。
  - `file_types`: 支持的文档扩展名，影响 `DocumentProcessor` 的文件加载。
  - `context_length`: 最大上下文字符数，限制 LLM 输入。
  - `concurrency`: 并行处理线程数，影响文档处理和提取速度。
  - `qps`: 查询每秒限制，与 LLM 的 `qps_limit` 结合使用。
  - `chunk_size`: 文档分片大小，决定每个 `DocumentChunk` 的内容长度。
  - `overlap_ratio`: 分片重叠比例，确保信息连续性。
  - `kb_dir`: 知识库存储目录，相对路径基于 `get_data_dir()`。
  - `default_mode`: 默认对话模式（`query`、`qa` 或 `auto`）。
  - `conversation_modes`: 定义不同模式的提示模板，支持定制化处理逻辑。
- **注意**:
  - 如果文件不存在，`ConfigLoader` 会创建默认配置文件。
  - 提示模板中的 `{query}`, `{doc_content}`, `{chunk_content}`, `{extractions_json}`, `{max_tokens}`, `{context}` 是占位符，会在运行时动态替换。
  - 确保 JSON 格式正确，编码为 UTF-8，避免解析错误。


### API说明表

以下表格详细列出核心类和方法的 API 说明，包括参数、返回值和用途，涵盖 `Conversation`、`KnowledgeBaseManager` 和 `DocumentProcessor` 的主要接口。

| 类/方法 | 参数 | 返回值 | 用途 | 注意事项 |
|---------|------|--------|------|----------|
| **Conversation.__init__** | `kb_manager: KnowledgeBaseManager`<br>`kb_ids: List[str]` | 无 | 初始化对话实例，加载指定知识库的分片 | 确保 `kb_ids` 对应的知识库存在，否则返回空知识库 |
| **Conversation.chat** | `query: str`<br>`stream: bool = False` | `Dict` 或 `Iterator[Dict]`<br>`Dict` 格式: `{"answer": str, "sources": List[Dict]}`<br>`sources` 格式: `[{"extracted": str, "source": str}, ...]` | 处理用户查询，返回答案和来源 | 如果知识库为空，返回错误信息；`stream=True` 返回迭代器，逐步生成答案 |
| **KnowledgeBaseManager.create_kb_id** | `kb_name: str` | `str` | 根据知识库名称生成 MD5 ID | 返回固定长度的唯一标识符 |
| **KnowledgeBaseManager.save_knowledge_base** | `kb_id: str`<br>`chunks: List[DocumentChunk]`<br>`kb_name: str`<br>`doc_paths: List[str] = None` | 无 | 保存知识库分片和元信息到文件 | 存储为 `.pkl`（分片）和 `.json`（元信息），使用临时文件确保原子性 |
| **KnowledgeBaseManager.load_knowledge_base** | `kb_id: str` | `List[DocumentChunk]` | 加载指定知识库的分片 | 如果文件不存在或损坏，返回空列表 |
| **KnowledgeBaseManager.list_kbs** | 无 | `List[Dict[str, str]]`<br>格式: `[{"id": str, "name": str}, ...]` | 列出所有知识库的 ID 和名称 | 依赖 `kb_dir` 中的 `_info.json` 文件 |
| **KnowledgeBaseManager.delete_kb** | `kb_id: str` | 无 | 删除指定知识库的文件 | 删除 `.pkl` 和 `_info.json` 文件，忽略不存在的文件 |
| **DocumentProcessor.load_documents** | `doc_paths: List[str]`<br>`query: str` | `List[DocumentChunk]` | 加载并分片文档，应用预过滤 | 使用 `query` 过滤无关文档；`"dummy_query"` 跳过过滤；分片大小由 `chunk_size` 和 `overlap_ratio` 控制 |
| **DocumentProcessor.prefilter_document** | `doc_path: str`<br>`query: str` | `bool` | 判断文档是否与查询相关 | 使用 LLM 判断，返回 `True`（相关）或 `False`（无关）；出错时默认返回 `True` |
| **ExtractionModule.extract_from_chunk** | `chunk: DocumentChunk`<br>`query: str` | `Dict`<br>格式: `{"extracted": str, "source": str}` | 从文档分片中提取与查询相关的内容 | 输出 JSON 格式；若提取失败，返回空内容 |
| **IntegrationModule.integrate_extractions** | `extractions: List[Dict]` | `str` | 整合提取结果，生成上下文 | 去除重复内容，限制长度为 `context_length // 2`；出错时返回简单拼接的上下文 |
| **AnswerModule.generate_answer** | `context: str`<br>`query: str`<br>`stream: bool = False` | `str` 或 `Iterator[str]` | 根据上下文和查询生成答案 | 如果上下文为空，返回提示信息；`stream=True` 返回迭代器 |

#### 表格说明
- **参数**: 列出方法的所有参数及其类型。
- **返回值**: 描述返回值的类型和格式。
- **用途**: 说明方法的功能和使用场景。
- **注意事项**: 指出使用时的关键点或潜在问题。
- **依赖**: 所有方法依赖 `LLMAPI` 的 `call_llm` 方法，确保其正确实现。
- **线程安全**: `Conversation` 的缓存使用 `threading.Lock` 确保线程安全，适合并发环境。

---

### 补充说明

#### 配置文件使用
- **创建配置文件**: 将上述 `ksasnt_config.json` 模板保存到 `get_data_dir()` 指定的目录（如 `~/Documents/KSASNT/`），或通过 `ConfigLoader` 自动生成默认配置。
- **自定义提示模板**: 可修改 `conversation_modes` 中的模板，添加新模式（如 `"summary"`）以支持特定场景。
- **LLM 配置**: 确保 `llmcall.env` 与 LLM 服务（如 Ollama）匹配，或使用默认值运行本地模型。


### 极简命令行文档对话工具

以下是一个极简的命令行工具实现，允许用户加载文档、创建知识库并进行交互式查询。


#### 使用方法
1. **准备文档**:
   - 创建一些`.md`或`.txt`文件，例如：
     ```
     doc1.md:
     # 人工智能简介
     人工智能（AI）是模拟人类智能的技术，包括机器学习和自然语言处理。

     doc2.txt:
     机器学习是AI的一个分支，涉及从数据中学习模式以进行预测或决策。
     ```

2. **运行工具**:
   - 保存上述代码为`doc_chat.py`。
   - 确保文档路径有效，例如将文档放在`~/Documents/KSASNT/`目录下。
   - 运行命令：
     ```bash
     python doc_chat.py --docs ~/Documents/KSASNT/doc1.md ~/Documents/KSASNT/doc2.txt --kb-name MyKnowledgeBase
     ```
   - 程序会加载文档，创建知识库，并进入交互模式。

3. **交互查询**:
   - 示例交互：
     ```
     找到 2 个有效文档：['/home/user/Documents/KSASNT/doc1.md', '/home/user/Documents/KSASNT/doc2.txt']
     已创建知识库 'MyKnowledgeBase' (ID: 5f4dcc3b5aa765d61d8327deb882cf99)

     输入查询（输入 'quit' 退出）：
     > 人工智能是什么？
     答案: 人工智能（AI）是模拟人类智能的技术，包括机器学习和自然语言处理。机器学习是AI的一个分支，涉及从数据中学习模式以进行预测或决策 [doc1.md:1-1, doc2.txt:1-1]。
     来源:
     - 人工智能（AI）是模拟人类智能的技术，包括机器学习和自然语言处理。 [doc1.md:1-1]
     - 机器学习是AI的一个分支，涉及从数据中学习模式以进行预测或决策。 [doc2.txt:1-1]

     > quit
     退出程序
     ```


     以下是为提供的代码添加的**配置文件模板**和**API说明表格**，以补充API使用说明，并保持清晰、结构化的格式。

---
#### API调用示例
以下是补充的 API 调用示例，扩展了命令行工具的功能，展示如何使用 `list_kbs` 和 `delete_kb`。

```python
import os
from conversation import ConfigLoader, KnowledgeBaseManager, DocumentProcessor, Conversation

def advanced_tool():
    config = ConfigLoader()
    llm = config.llm_config
    kb_manager = KnowledgeBaseManager(config, llm)
    doc_processor = DocumentProcessor(config, llm)

    # 列出已有知识库
    print("可用知识库:")
    for kb in kb_manager.list_kbs():
        print(f"- {kb['name']} (ID: {kb['id']})")

    # 创建知识库
    doc_paths = ["doc1.md", "doc2.txt"]  # 替换为实际路径
    existing_docs = [p for p in doc_paths if os.path.exists(p)]
    if not existing_docs:
        print("错误：无有效文档")
        return
    kb_name = "TestKB"
    kb_id = kb_manager.create_kb_id(kb_name)
    chunks = doc_processor.load_documents(existing_docs, "dummy_query")
    kb_manager.save_knowledge_base(kb_id, chunks, kb_name, existing_docs)
    print(f"创建知识库: {kb_name} (ID: {kb_id})")

    # 初始化对话
    conv = Conversation(kb_manager, [kb_id])

    # 查询
    query = "文档中的主要主题是什么？"
    result = conv.chat(query)
    print(f"\n查询: {query}")
    print(f"答案: {result['answer']}")
    if result['sources']:
        print("来源:")
        for src in result['sources']:
            print(f"- {src['extracted']} [{src['source']}]")

    # 删除知识库
    kb_manager.delete_kb(kb_id)
    print(f"已删除知识库: {kb_id}")

if __name__ == "__main__":
    advanced_tool()
```

#### 示例输出
假设 `doc1.md` 和 `doc2.txt` 存在：
```
可用知识库:
- MyKB (ID: 5f4dcc3b5aa765d61d8327deb882cf99)
创建知识库: TestKB (ID: a1b2c3d4e5f67890abcdef1234567890)

查询: 文档中的主要主题是什么？
答案: 人工智能（AI）是模拟人类智能的技术，包括机器学习和自然语言处理。机器学习是AI的一个分支，涉及从数据中学习模式以进行预测或决策 [doc1.md:1-1, doc2.txt:1-1]。
来源:
- 人工智能（AI）是模拟人类智能的技术，包括机器学习和自然语言处理。 [doc1.md:1-1]
- 机器学习是AI的一个分支，涉及从数据中学习模式以进行预测或决策。 [doc2.txt:1-1]
已删除知识库: a1b2c3d4e5f67890abcdef1234567890
```