title: 动手做一个AIAgent - RAG基础
date: 2026-03-27 19:52:57
tags:
    - 技术相关
    - AI Agent
---

系列文章:

1. [动手做一个AIAgent - 简易框架搭建](https://blog.islinjw.cn/2026/02/25/%E5%8A%A8%E6%89%8B%E5%81%9A%E4%B8%80%E4%B8%AAAIAgent-%E7%AE%80%E6%98%93%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/)
1. [动手做一个AIAgent - LiteLLM](https://blog.islinjw.cn/2026/02/26/%E5%8A%A8%E6%89%8B%E5%81%9A%E4%B8%80%E4%B8%AAAIAgent-LiteLLM/)
1. [动手做一个AIAgent - 流式输出与视觉支持](https://blog.islinjw.cn/2026/03/05/%E5%8A%A8%E6%89%8B%E5%81%9A%E4%B8%80%E4%B8%AAAIAgent-%E8%A7%86%E8%A7%89%E6%94%AF%E6%8C%81%E4%B8%8E%E6%B5%81%E5%BC%8F%E8%BE%93%E5%87%BA/)
1. [动手做一个AIAgent - MCP](https://blog.islinjw.cn/2026/03/16/%E5%8A%A8%E6%89%8B%E5%81%9A%E4%B8%80%E4%B8%AAAIAgent-MCP/)
1. [动手做一个AIAgent - SKILLS](https://blog.islinjw.cn/2026/03/19/%E5%8A%A8%E6%89%8B%E5%81%9A%E4%B8%80%E4%B8%AAAIAgent-SKILLS/)
1. [动手做一个AIAgent - RAG基础](https://blog.islinjw.cn/2026/03/27/%E5%8A%A8%E6%89%8B%E5%81%9A%E4%B8%80%E4%B8%AAAIAgent-RAG%E5%9F%BA%E7%A1%80/)

LLM是基于某个时间点之前的数据集训练出来的,这就意味着它无法知道在这个时间点之后的事情,也无法知道这个训练集(例如企业的内部文档)之外的事情。

你当然可以将需要告诉它的文档写到prompt里面一股脑发给他再来提问。但文档本身可能会很大,这样一方面会造成token的浪费,另一方面LLM也会由于过多的冗余信息而造成更多的幻觉。还有研究表明LLM处理长上下文的时候会出现只记得开头结尾的信息而丢失了中间部分内容的情况,也就是"Lost in the Middle"现象。

# 什么是RAG

这样就意味着我们需要尽可能的将与当前对话相关的信息告诉LLM,最好不要有其他无关的信息来干扰LLM的文本生成。或者可以换种思路由LLM在需要的时候向数据库查询当前对话相关的内容,这就是RAG( Retrieval-Augmented Generation),中文叫做检索增强生成。

例如用户问"AI是什么?"那我们不能只搜索包含AI两个字母的文档,类似深度学习、人工智能之类语义相近的文档最好也能包含在里面。这样就意味着传统的数据库无法在这个场景里使用,而需要用到*向量数据库*。

向量也可以理解成坐标,把一份文本根据语义转换成一个多维空间的坐标,语义相近的文本转换出来的坐标就相近。

于是我们就可以比较简单的去理解RAG的简要流程:

将文档按某种规则分好块,计算出每一块的向量(也就是坐标)保存到向量数据库,在用户请求的时候将用户的问题也计算出向量,然后在向量数据库里面查询出与问题向量最接近的几个文档分块,最后将这几个分块告诉LLM去生成最终的回答。

{% plantuml %}
文档 -> 文档分块 : 切分
文档分块 -> 向量数据库 : 计算每个分块的向量分别保存
LLM <- 用户 : 询问问题
向量数据库 <- LLM : 计算问题向量去查询
向量数据库 -> LLM : 得到最相近的几个文档分块
LLM -> 用户 : 结合文档分块和问题回复
{% endplantuml %}

我这边截取了部分[阿里巴巴java开发手册](https://developer.aliyun.com/article/1589859)的[内容](https://github.com/bluesky466/SimpleAgent/blob/feature/rag/mcps/rag_demo/coding-standards.md),我们将以它为例子介绍下如何使用python实现整个rag的流程

# 离线阶段

*离线阶段*对应知识库的构建(文档加载 → 分块 → 向量化 → 存储),属于检索阶段的前置准备

### 分块

首先我们需要对文档进行分块,一般会有这几种做法:

- 固定大小分块 : 按固定的字符数分割
- 递归字符分块 : 设定一个块的大小,先按段落分割,如果段落太长再分割出句子,如果句子还是太长再分割出单词
- 按语义分块 : 按语义在文本主题变化的地方去分割

这里的文档由于是按小点列举出来的，我们可以将每一个小点分割成一个块:

```python
def load_rag_chunks(doc_path: Path) -> List[str]:
    """读取文档，按全角左方括号「【」切分为 RAG 分片。

    首段为第一个「【」之前的正文（如章节标题）；其后每段以「【」开头，对应一条规约条目。
    """
    path = Path(doc_path)
    text = path.read_text(encoding="utf-8")
    parts = text.split("【")
    chunks: List[str] = []
    head = parts[0].strip()
    if head:
        chunks.append(head)
    for body in parts[1:]:
        chunk = ("【" + body).strip()
        if chunk:
            chunks.append(chunk)
    return chunks
```

分出来的块是这样的(以横线展示分割):

```
1.1 命名风格
------------------
【强制】代码中的命名均不能以下划线或美元符号开始，也不能以下划线或美元符号结束。
反例：_name / __name / O b j e c t / n a m e / n a m e Object / name_ / name Object/name/​name / Object$                     
------------------
【强制】代码中的命名严禁使用拼音与英文混合的方式，更不允许直接使用中文的方式。
说明：正确的英文拼写和语法可以让阅读者易于理解，避免歧义。注意，即使纯拼音命名方式
也要避免采用。
正例：alibaba / taobao / youku / hangzhou 等国际通用的名称，可视同英文。
反例：DaZhePromotion [打折] / getPingfenByName() [评分] / int 某变量 = 3
------------------
【强制】类名使用 UpperCamelCase 风格，必须遵从驼峰形式，但以下情形例外：DO / BO /
DTO / VO / AO
正例：MarcoPolo / UserDO / XmlService / TcpUdpDeal / TaPromotion
反例：macroPolo / UserDo / XMLService / TCPUDPDeal / TAPromotion
------------------
...
```

### 向量化

然后我们需要使用Embedding模型去计算每个分块的向量

```python
from sentence_transformers import SentenceTransformer
embedding_model = SentenceTransformer("BAAI/bge-small-zh-v1.5")

def embed_chunk(chunk: str) -> List[float]:
    embedding = embedding_model.encode(chunk, normalize_embeddings=True)
    return embedding.tolist()

embeddings = {chunk: embed_chunk(chunk) for chunk in chunks}
```

第一次执行上面的脚本的时候会自动去[huggingface](https://huggingface.co/)下载`BAAI/bge-small-zh-v1.5`模型,会有一段等待时间,下载完以后就会复用缓存了。在[huggingface](https://huggingface.co/BAAI/bge-small-zh-v1.5)上也会有比较详细的用法说明。


模型选择可以参考huggingface的[MTEB(Massive Text Embedding Benchmark)排行榜](https://huggingface.co/spaces/mteb/leaderboard):

{% img /AIAgent-RAG基础/leaderboard.png %}

或者我们可以切到`Performance per Model Size`气泡图去更直观的对比每个模型的性能、成本、文本长度等维度

{% img /AIAgent-RAG基础/leaderboard2.png %}

- 横轴 : 模型参数量 (Number of Parameters),就是llm里面常说的多少多少B,参数量越大一般能力越强但对内存显存等硬件性能的要求越高
- 纵轴 : 得分(Mean Task),使用一些列的测试集去测试模型得出一个分数,分数越高代表这个模型的语义理解能力越强
- 气泡大小 : 维度(Embedding Size),模型最终计算出来的向量的维度,维度越高能保持的语义信息越多但占用的资源也会越多
- 气泡颜色 : 最大处理长度 (Max Tokens) 模型支持的文本长度上线,颜色越深能处理越长的文本

### 保存到向量数据库

向量计算完成我们就可以将向量和它代表的文档关联起来保存到向量数据库。

向量数据库有蛮多的选择,但用于学习的话还是推荐使用轻量级的开源数据库[chroma](https://github.com/chroma-core/chroma):

```python
import chromadb

chromadb_client = chromadb.EphemeralClient()
chromadb_collection = chromadb_client.get_or_create_collection(name="default")

def save_embeddings(embeddings: dict[str, List[float]]) -> None:
    for chunk, embedding in embeddings.items():
        chromadb_collection.add(
            documents=[chunk],
            embeddings=[embedding],
            ids=[chunk]
        )

save_embeddings(embeddings)
```

这里的EphemeralClient是一种用于临时、内存中操作的客户端模式,数据仅存在于内存中，程序退出或会话终止后数据自动丢失。如果需要保存下来下次继续使用可以用PersistentClient

```python
chromadb.PersistentClient(path=str(cache_dir))
```

# 在线阶段

*在线阶段*即用户提问后的实时检索与生成。虽然在文章开头我们简单把这个阶段总结成从向量数据库查询与用户问题最接近的几个文档分块,提供给LLM去生成最终的回答,但实际这个阶段还有蛮多需要处理的细节。

### Query改写

用户的问题会存在模糊、口语化、不完整或歧义等问题,例如可能用户在讨论java里的枚举,然后就问了一句`它的命名有什么样的规范?`。

这里就需要llm根据上下文将查询语句改成`枚举类的命名有什么样的规范?`

### 召回

得到了比较准确的查询语句之后就需要去向量数据库里面检索了,这个步骤被叫做召回:

```python
def retrieve(query: str, top_k: int) -> List[str]:
    query_embedding = embed_chunk(query)
    results = chromadb_collection.query(
        query_embeddings=[query_embedding],
        n_results=top_k
    )
    return results['documents'][0]

retrieved_chunks = retrieve(query, 10)
```

由于向量数据库里面的会有海量的文档,召回其实重点在于速度。需要尽量快的找到相近的文档。

在速度优先的情况下就可能会出现精度的缺失,容易漏掉一些。所以一般会从向量数据库召回比较多的文档,例如20份或者更多,这里的召回数量通常以`Top-K`参数表示

### 重排

召回的步骤得到了比较多的文档,而且他们的排序可能也会有问题,最相关的文档被放到了文档的中间部分。所以我们需要对召回的文档做更加精细化的检查将他们按照相关性排序,这个步骤叫做重排:

```python
cross_encoder = CrossEncoder('cross-encoder/mmarco-mMiniLMv2-L12-H384-v1')

def rerank(query: str, retrieved_chunks: List[str], top_k: int) -> List[str]:
    pairs = [(query, chunk) for chunk in retrieved_chunks]
    scores = cross_encoder.predict(pairs)
    scored_chunks = list(zip(retrieved_chunks, scores))
    scored_chunks.sort(key=lambda x: x[1], reverse=True)
    return [chunk for chunk, _ in scored_chunks][:top_k]

reranked_chunks = rerank(query, retrieved_chunks, 3)
```

召回是广撒网,重排就是对召回的数据精加工。而重排部分也会有专门的rerank模型去做。可以在[排行榜](https://huggingface.co/spaces/mteb/leaderboard)这里筛选rerank模型去选择:

{% img /AIAgent-RAG基础/leaderboard3.png %}

### 生成

这个时候就可以在重排结果里面选择相关性最高的3个或者5个与用户问题一起发给llm做最终结果的生成了。我们可以为这个RAG搭一个简单的[mcp server](https://github.com/bluesky466/SimpleAgent/blob/feature/rag/mcps/rag_demo/coding_standards_rag_server.py)导入我们的AiAgent里面:

```json
{
	"llm": {
		...
	},
	"mcp": {
		...
		"coding-standards-rag-server" : {
			"command": "python3",
			"args": [
				"./mcps/rag_demo/coding_standards_rag_server.py"
			]
		},
		...
	}
}
```

然后就可以和它对话去使用rag查询编码规范了:

```
请输入(Ctrl+C 退出): java的枚举类和c语言的枚举有什么差别     
....................
...省略这部分内容...
====================
请输入(Ctrl+C 退出): 根据我们的编码规范，它的命名有什么要求
....................
...思考中...
用户问的是关于枚举类命名的编码规范要求。我需要使用RAG检索编码规范的相关内容来回答这个问题。我应该搜索关于枚举命名的内容。我来
帮您查找编码规范中关于枚举命名的要求。
调用工具: MCP$coding-standards-rag-server$get_result, 参数: {'query': '枚举命名 enum 命名规范'}
工具调用结果: ["{\"type\":\"text\",\"text\":\"[\\\"\\\\u3010\\\\u53c2\\\\u8003\\\\u3011\\\\u679a\\\\u4e3e\\\\u7c7b\\\\u540d\\
\\u5efa\\\\u8bae\\\\u5e26\\\\u4e0a Enum \\\\u540e\\\\u7f00\\\\uff0c\\\\u679a\\\\u4e3e\\\\u6210\\\\u5458\\\\u540d\\\\u79f0\\\\
u9700\\\\u8981\\\\u5168\\\\u5927\\\\u5199\\\\uff0c\\\\u5355\\\\u8bcd\\\\u95f4\\\\u7528\\\\u4e0b\\\\u5212\\\\u7ebf\\\\u9694\\\
\u5f00\\\\u3002\\\\n\\\\u8bf4\\\\u660e\\\\uff1a\\\\u679a\\\\u4e3e\\\\u5176\\\\u5b9e\\\\u5c31\\\\u662f\\\\u7279\\\\u6b8a\\\\u7
684\\\\u5e38\\\\u91cf\\\\u7c7b\\\\uff0c\\\\u4e14\\\\u6784\\\\u9020\\\\u65b9\\\\u6cd5\\\\u88ab\\\\u9ed8\\\\u8ba4\\\\u5f3a\\\\u
5236\\\\u662f\\\\u79c1\\\\u6709\\\\u3002\\\\n\\\\u6b63\\\\u4f8b\\\\uff1a\\\\u679a\\\\u4e3e\\\\u540d\\\\u5b57\\\\u4e3a Process
StatusEnum \\\\u7684\\\\u6210\\\\u5458\\\\u540d\\\\u79f0\\\\uff1aSUCCESS / UNKOWN_REASON\\\\u3002\\\", \\\"\\\\u3010\\\\u53c2
\\\\u8003\\\\u3011\\\\u5404\\\\u5c42\\\\u547d\\\\u540d\\\\u89c4\\\\u7ea6\\\\uff1a\\\\nA) Service/DAO \\\\u5c42\\\\u65b9\\\\u6
cd5\\\\u547d\\\\u540d\\\\u89c4\\\\u7ea6\\\\n1\\\\uff09 \\\\u83b7\\\\u53d6\\\\u5355\\\\u4e2a\\\\u5bf9\\\\u8c61\\\\u7684\\\\u65
b9\\\\u6cd5\\\\u7528 get \\\\u505a\\\\u524d\\\\u7f00\\\\u3002\\\\n2\\\\uff09 \\\\u83b7\\\\u53d6\\\\u591a\\\\u4e2a\\\\u5bf9\\\
\u8c61\\\\u7684\\\\u65b9\\\\u6cd5\\\\u7528 list \\\\u505a\\\\u524d\\\\u7f00\\\\u3002\\\\n3\\\\uff09 \\\\u83b7\\\\u53d6\\\\u7e
df\\\\u8ba1\\\\u503c\\\\u7684\\\\u65b9\\\\u6cd5\\\\u7528 count \\\\u505a\\\\u524d\\\\u7f00\\\\u3002\\\\n4\\\\uff09 \\\\u63d2\
\\\u5165\\\\u7684\\\\u65b9\\\\u6cd5\\\\u7528 save/insert \\\\u505a\\\\u524d\\\\u7f00\\\\u3002\\\\n5\\\\uff09 \\\\u5220\\\\u96
64\\\\u7684\\\\u65b9\\\\u6cd5\\\\u7528 remove/delete \\\\u505a\\\\u524d\\\\u7f00\\\\u3002\\\\n6\\\\uff09 \\\\u4fee\\\\u6539\\
\\u7684\\\\u65b9\\\\u6cd5\\\\u7528 update \\\\u505a\\\\u524d\\\\u7f00\\\\u3002\\\\nB) \\\\u9886\\\\u57df\\\\u6a21\\\\u578b\\\
\u547d\\\\u540d\\\\u89c4\\\\u7ea6\\\\n1\\\\uff09 \\\\u6570\\\\u636e\\\\u5bf9\\\\u8c61\\\\uff1axxxDO\\\\uff0cxxx \\\\u5373\\\\
u4e3a\\\\u6570\\\\u636e\\\\u8868\\\\u540d\\\\u3002\\\\n2\\\\uff09 \\\\u6570\\\\u636e\\\\u4f20\\\\u8f93\\\\u5bf9\\\\u8c61\\\\u
ff1axxxDTO\\\\uff0cxxx \\\\u4e3a\\\\u4e1a\\\\u52a1\\\\u9886\\\\u57df\\\\u76f8\\\\u5173\\\\u7684\\\\u540d\\\\u79f0\\\\u3002\\\
\n3\\\\uff09 \\\\u5c55\\\\u793a\\\\u5bf9\\\\u8c61\\\\uff1axxxVO\\\\uff0cxxx \\\\u4e00\\\\u822c\\\\u4e3a\\\\u7f51\\\\u9875\\\\
u540d\\\\u79f0\\\\u3002\\\\n4\\\\uff09 POJO \\\\u662f DO/DTO/BO/VO \\\\u7684\\\\u7edf\\\\u79f0\\\\uff0c\\\\u7981\\\\u6b62\\\\
u547d\\\\u540d\\\\u6210 xxxPOJO\\\\u3002\\\\n1.2 \\\\u5e38\\\\u91cf\\\\u5b9a\\\\u4e49\\\", \\\"\\\\u3010\\\\u63a8\\\\u8350\\\
\u3011\\\\u5982\\\\u679c\\\\u53d8\\\\u91cf\\\\u503c\\\\u4ec5\\\\u5728\\\\u4e00\\\\u4e2a\\\\u8303\\\\u56f4\\\\u5185\\\\u53d8\\
\\u5316\\\\uff0c\\\\u4e14\\\\u5e26\\\\u6709\\\\u540d\\\\u79f0\\\\u4e4b\\\\u5916\\\\u7684\\\\u5ef6\\\\u4f38\\\\u5c5e\\\\u6027\
\\\uff0c\\\\u5b9a\\\\u4e49\\\\u4e3a\\\\u679a\\\\u4e3e\\\\u7c7b\\\\u3002\\\\u4e0b\\\\u9762\\\\n\\\\u6b63\\\\u4f8b\\\\u4e2d\\\\
u7684\\\\u6570\\\\u5b57\\\\u5c31\\\\u662f\\\\u5ef6\\\\u4f38\\\\u4fe1\\\\u606f\\\\uff0c\\\\u8868\\\\u793a\\\\u661f\\\\u671f\\\
\u51e0\\\\u3002\\\\n\\\\u6b63\\\\u4f8b\\\\uff1apublic Enum { MONDAY(1), TUESDAY(2), WEDNESDAY(3), THURSDAY(4), FRIDAY(5), SAT
URDAY(6),\\\\nSUNDAY(7);}\\\\n1.3 \\\\u4ee3\\\\u7801\\\\u683c\\\\u5f0f\\\"]\",\"annotations\":null,\"meta\":null}"]
根据编码规范的查询结果，我找到了关于枚举命名的要求。让我整理一下关键信息：
1. 枚举类名建议带上 Enum 后缀
2. 枚举成员名称需要全大写，单词间用下划线隔开

从查询结果来看，这是完整的枚举命名规范要求，我可以直接向用户提供这些信息。根据我们的编码规范，枚举类的命名有明确要求：
...中间部分省略...
...思考结束...
根据我们的编码规范，枚举类的命名有明确要求：

## 枚举命名规范

### 1. 枚举类名
- **建议**带上 `Enum` 后缀
- 示例：`ProcessStatusEnum`

### 2. 枚举成员名称
- 必须**全大写**
- 单词之间用**下划线**隔开
- 示例：`SUCCESS`、`UNKNOWN_REASON`

## 3. 完整示例

public enum ProcessStatusEnum {
    SUCCESS,           // 成功
    FAILED,            // 失败
    UNKNOWN_REASON     // 未知原因
}

或者带有构造参数的枚举：

public enum DayEnum {
    MONDAY(1),
    TUESDAY(2),
    WEDNESDAY(3),
    THURSDAY(4),
    FRIDAY(5),
    SATURDAY(6),
    SUNDAY(7);

    private int value;

    DayEnum(int value) {
        this.value = value;
    }
}
====================

## 使用建议
规范还提到，如果变量值仅在一个范围内变化，且带有名称之外的扩展属性，应该定义为枚举类。例如上面的 `DayEnum` 中，数字就是扩展信
息，表示星期几。

这样命名可以提高代码的可读性和一致性，便于团队协作维护。
```

从上面可以看到我们问它的两个问题是
1. java的枚举类和c语言的枚举有什么差别
2. 根据我们的编码规范，它的命名有什么要求

然后从它调用mcp的日志可以看出它自动联系上下文将请求的query改写成了`枚举命名 enum 命名规范`:

```
调用工具: MCP$coding-standards-rag-server$get_result, 参数: {'query': '枚举命名 enum 命名规范'}
```

