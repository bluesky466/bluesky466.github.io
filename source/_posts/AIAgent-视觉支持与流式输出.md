title: AIAgent - 流式输出与视觉支持
date: 2026-03-05 21:26:29
tags:
    - 技术相关
    - AI Agent
---

系列文章:

1. [AIAgent - 简易框架搭建](https://blog.islinjw.cn/2026/02/25/AIAgent-%E7%AE%80%E6%98%93%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/)
1. [AIAgent - LiteLLM](https://blog.islinjw.cn/2026/02/26/AIAgent-LiteLLM/)
1. [AIAgent - 流式输出与视觉支持](https://blog.islinjw.cn/2026/03/05/AIAgent-%E8%A7%86%E8%A7%89%E6%94%AF%E6%8C%81%E4%B8%8E%E6%B5%81%E5%BC%8F%E8%BE%93%E5%87%BA/)
1. [AIAgent - MCP](https://blog.islinjw.cn/2026/03/16/AIAgent-MCP/)
1. [AIAgent - SKILLS](https://blog.islinjw.cn/2026/03/19/AIAgent-SKILLS/)

# 流式输出

目前我们的agent每次对话的耗时可能会比较久，一方面是我们需要等到完整的响应到来之后才一次性打印，另一方也是因为很多模型在深度思考的过程中也会占用时间。LiteLLM也是支持[流式输出](https://docs.litellm.ai/docs/completion/stream)的,然后[深度思考](https://docs.litellm.ai/docs/reasoning_content)的数据放在`reasoning_content`里面。

所以我们只需要对我们的agent做简单的[改造](https://github.com/bluesky466/SimpleAgent/blob/feature/stream/agent_brain.py)即可:

```python
class AgentBrain:
    ...
    def _read_response_stream(self, stream):
        chunks = []
        for chunk in stream:
            chunks.append(chunk)

            if not chunk.choices:
                continue
            delta = chunk.choices[0].delta

            reasoning = getattr(delta, "reasoning_content", None) or ""
            content = getattr(delta, "content", None) or ""
            if not reasoning and not content:
                continue

            if reasoning:
                self._stream_trace_reader("reasoning", reasoning)
            if content:
                self._stream_trace_reader("content", content)
        return stream_chunk_builder(chunks).choices[0].message

    def think(self, prompt):
        try:
            self._memory.add_user_prompt(prompt)
            stream = completion(
                model = self._model,
                messages = self._memory.get_memory(),
                tools=self._tools_definition,
                api_key=self._api_key,
                stream=True,
            )
            message = self._read_response_stream(stream)
            self._memory.add_agent_response(message)
            return message
        except Exception as e:
            return f"思考过程出错: {e}"
```

stream模式下会逐个delta读取增量数据,然后通过`stream_chunk_builder`将他们打包组合成完整的响应,这些增量数据大概是这样的:

```
Delta(reasoning_content='用户', provider_specific_fields=None, refusal=None, content=None, role='assistant', fun
ction_call=None, tool_calls=None, audio=None)

Delta(reasoning_content='问', provider_specific_fields=None, refusal=None, content=None, role=None, function_cal
l=None, tool_calls=None, audio=None)

Delta(reasoning_content='我是', provider_specific_fields=None, refusal=None, content=None, role=None, function_c
all=None, tool_calls=None, audio=None)

Delta(reasoning_content='谁', provider_specific_fields=None, refusal=None, content=None, role=None, function_cal
l=None, tool_calls=None, audio=None)
...
```

然后我们改造下[SimpleAgent](https://github.com/bluesky466/SimpleAgent/blob/feature/stream/simple_agent.py)去打印这些增量数据,就可以看到深度思考的内容会不断地在终端显示出来:

```
请输入(Ctrl+C 退出): 你是谁
....................
...思考中...
用户在问"你是谁"，这是一个关于我身份的基本问题。我需要诚实地回答我的身份，同时保持友好和专业的态度。

我不需要使用任何工具来回答这个问题，因为这是一个直接的身份介绍问题。我应该简单明了地介绍自己，说明我是一个AI助手，以及我的目的和能力。

你好！我是一个AI智能助手，由智谱AI训练开发。我可以帮助你解答问题、提供信息、进行对话交流，以及协助你完成各种文本相关的任务。

我的设计目的是为了提供有用的信息、进行有意义的对话，并帮助用户解决各种问题。有什么我可以帮助你的吗？
...思考结束...

你好！我是一个AI智能助手，由智谱AI训练开发。我可以帮助你解答问题、提供信息、进行对话交流，以及协助你完成各种文本相关的任务。

我的设计目的是为了提供有用的信息、进行有意义的对话，并帮助用户解决各种问题。有什么我可以帮助你的吗？
====================
```

# 视觉支持

一个ai agent不能只能文字聊天,很多任务会需要理解图片才能思考下一步。LiteLLM里面也是支持调用[视觉模型](https://docs.litellm.ai/docs/completion/vision)的:

```python
import os 
from litellm import completion

os.environ["OPENAI_API_KEY"] = "your-api-key"

# openai call
response = completion(
    model = "gpt-4-vision-preview", 
    messages=[
        {
            "role": "user",
            "content": [
                            {
                                "type": "text",
                                "text": "What’s in this image?"
                            },
                            {
                                "type": "image_url",
                                "image_url": {
                                "url": "https://awsmp-logos.s3.amazonaws.com/seller-xw5kijmvmzasy/c233c9ade2ccb5491072ae232c814942.png"
                                }
                            }
                        ]
        }
    ],
)
```

如果是本地图片的话,我们可以在本地启动一个图片的代理http服务器,也可以使用[data url](https://en.wikipedia.org/wiki/Data_URI_scheme)的方式将图片内容用base64编码之后发给llm去识别:

```python
def _file_to_image_url(self, path: str) -> str:
    path = path.strip()
    if path.startswith("http"):
        # 网络图片直接返回
        return path
    # 本地图片使用data url格式返回base64内容
    p = Path(path).expanduser().resolve()
    if not p.exists():
        raise FileNotFoundError(f"文件不存在: {path}")
    raw = p.read_bytes()
    b64 = base64.standard_b64encode(raw).decode("ascii")
    mime, _ = mimetypes.guess_type(str(p))
    mime = mime or "application/octet-stream"
    return f"data:{mime};base64,{b64}"
```

当然我们选择模型的时候也需要选择支持视觉的[模型](https://docs.bigmodel.cn/cn/guide/models/vlm/glm-4.6v)才能看得懂图片。

## agent改造

然后我们就可以对我们的[AgentBrain](https://github.com/bluesky466/SimpleAgent/blob/feature/vision/agent_brain.py)进行改造。

将`{img:/path/to/img}`这样格式的字符串作为图片占位符去解析保存到对话:

```python


IMG_PLACEHOLDER_PATTERN = re.compile(r"\{img:([^}]+)\}")

class AgentBrain:
    def __init__(self, llm_config: dict, memory: AgentMemory, tool_manager: ToolManager, stream_trace_reader: Callable[[str, str], None]):
        self._model = llm_config["model"]
        self._model_support_vision = llm_config["model_support_vision"]
        self._api_key = llm_config["api_key"]
        self._memory = memory
        self._tools_definition = tool_manager.get_tool_definition()
        self._stream_trace_reader = stream_trace_reader
    
    ...

    def _prompt_to_content(self, prompt: str) -> str | list:
        parts = IMG_PLACEHOLDER_PATTERN.split(prompt)
        if len(parts) == 1:
            return prompt
        content = []
        for i, seg in enumerate(parts):
            if not seg:
                continue
            if i % 2 == 1:
                path = seg.strip()
                content.append({"type": "image_url", "image_url": {"url": self._file_to_image_url(path)}})
            else:
                content.append({"type": "text", "text": seg})
        return content

    def think(self, prompt):
        try:
            content = self._prompt_to_content(prompt)
            need_support_vision = content and isinstance(content, list)
            model = self._model_support_vision if need_support_vision else self._model
            messages = self._memory.get_memory() + [{"role": "user", "content": content}]
            stream = completion(
                model = model,
                messages = messages,
                tools=self._tools_definition,
                api_key=self._api_key,
                stream=True,
            )
            self._memory.add_user_content(prompt)
            message = self._read_response_stream(stream)
            self._memory.add_agent_response(message)
            return message
        except Exception as e:
            return f"思考过程出错: {e}"
```

例如`帮我描述下{img:https://blog.islinjw.cn/images/avatar.jpeg}这张图片`这句话最终发给LiteLLM的content就是:

```json
[
    {
        "type": "text",
        "text": "帮我描述下"
    },
    {
        "type": "image_url",
        "image_url": {
            "url": "https://blog.islinjw.cn/images/avatar.jpeg"
        }
    },
    {
        "type": "text",
        "text": "这张图片"
    }
]
```

另外因为并不是所有的模型都支持视觉,将图片传给不支持视觉的模型会出现异常,所以我们可以添加一个[model_support_vision配置](https://github.com/bluesky466/SimpleAgent/blob/feature/vision/config.json)用于在需要识别图片的时候切到专用的视觉模型:

```json
{
    "llm": {
        "api_key" : "<这里填你的智谱apikey>",
        "model": "zai/glm-4.7",
        "model_support_vision": "zai/glm-4.6v"
    }
}
```

于是我们就可以让agent看到云端和本地的这两张图片了:

{% img /AIAgent-视觉支持与流式输出/img.png %}

```shell
请输入(Ctrl+C 退出): 帮我描述下{img:https://blog.islinjw.cn/images/avatar.jpeg}这张图片
....................
...思考中...
# 省略思考的内容
...思考结束...

这张图片展示的是一个卡通风格的头像形象。画面中的人物有着黑色的尖刺状头发，发丝质感清晰且富有动感；佩戴着黑框眼镜，镜片呈现出
一定的反光效果；面部表情显得较为严肃或带有思考的神态，耳朵的轮廓较为明显。整体画风偏向Q版，线条简洁，色彩对比鲜明，给人一种精
致且略带酷感的卡通人物形象感。
====================
请输入(Ctrl+C 退出): 帮我描述下{img:/Users/linjw/Downloads/ollama.png}这张图片
....................
...思考中...
# 省略思考的内容
...思考结束...

这张图片展示的是一个**卡通风格的羊驼（或类似动物）简笔画**，整体采用黑白线条设计，风格简约可爱：

- **头部**：呈圆润的圆形轮廓，两只耳朵是上翘的椭圆形，线条简洁流畅；
- **面部特征**：眼睛为两个小黑点，鼻子周围有圆形的轮廓装饰，鼻子中间有一个小“Y”形（或类似标记）的细节；
- **身体**：线条圆润，整体造型扁平化，没有复杂纹理，给人简洁、呆萌的视觉感受。

整体设计风格偏向**扁平化卡通**，通过简单的线条和几何形状，传递出轻松可爱的氛围。
====================
请输入(Ctrl+C 退出): 哪一张图片画的是动物
....................
...思考中...
# 省略思考的内容
...思考结束...
**第二张图片**（/Users/linjw/Downloads/ollama.png）画的是动物。

具体来说，那是一只**卡通风格的羊驼**（或类似的动物形象），有圆圆的头部、上翘的椭圆形耳朵，整体造型呆萌可爱。
====================
```
