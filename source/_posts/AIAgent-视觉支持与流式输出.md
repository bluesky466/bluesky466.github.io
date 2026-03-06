title: AIAgent - 视觉支持与流式输出
date: 2026-03-05 21:26:29
tags:
    - 技术相关
    - AI Agent
---

系列文章:

1. [AIAgent - 简易框架搭建](https://blog.islinjw.cn/2026/02/25/AIAgent-%E7%AE%80%E6%98%93%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/)
1. [AIAgent - LiteLLM](https://blog.islinjw.cn/2026/02/26/AIAgent-LiteLLM/)
1. [AIAgent - 视觉支持与流式输出](https://blog.islinjw.cn/2026/03/05/AIAgent-%E8%A7%86%E8%A7%89%E6%94%AF%E6%8C%81%E4%B8%8E%E6%B5%81%E5%BC%8F%E8%BE%93%E5%87%BA/)

# LiteLLM的图片接口

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

如果是本地图片的话,我们可以在本地启动一个图片的代理http服务器,也可以使用[data url](https://en.wikipedia.org/wiki/Data_URI_scheme)的方式去做:

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

# agent改造

然后我们就可以对我们的[agent](https://github.com/bluesky466/SimpleAgent/blob/main/brain_support_image.py)进行改造。

将`{img:/path/to/img}`这样格式的字符串作为图片占位符去解析:

```python
from litellm import completion
import json
import platform
import base64
import mimetypes
import re
from pathlib import Path

IMG_PLACEHOLDER_PATTERN = re.compile(r"\{img:([^}]+)\}")

class AgentBrain:
    ...

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
            self.messages.append({"role": "user", "content": content})

            message = completion(
                model=self.model,
                messages=self.messages,
                tools=self.tools_definition,
            ).choices[0].message
            ...
            else:
                return message.content
        except Exception as e:
            return f"思考过程出错: {e}"
    ...
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

于是我们就可以让agent看到云端和本地的这两张图片了:

{% img /AIAgent-视觉支持与流式输出/img.png %}


```shell
请输入(Ctrl+C 退出): 帮我描述下{img:https://blog.islinjw.cn/images/avatar.jpeg}这张图片
....................

这是一张卡通风格的头像图片，展示了一个Q版动漫角色：

**人物特征：**
- **发型**：深黑色的尖刺头，顶部有明显的立体感和层次感，发丝呈现不规则的尖刺状
- **眼镜**：戴着黑框眼镜，镜片是浅色的
- **面部**：侧脸视角，面向右侧，眼睛大而圆，眼神显得有些严肃或思考
- **肤色**：浅肤色
- **耳朵**：清晰可见的耳朵轮廓

**整体风格：**
- 采用日式动漫/漫画风格绘制
- 线条简洁，色彩明快
- 头部比例较大，属于Q版卡通形象
- 背景是纯白色，突出人物主体

这个形象看起来像是一个年轻男性角色，表情略带严肃或沉思的感觉，可能是某个动漫作品中的角色或者原创的卡通形象。
====================
请输入(Ctrl+C 退出): 帮我描述下{img:/Users/linjw/Downloads/ollama.png}这张图片
....................

这张图片展示了一个**卡通风格的羊驼（或类似动物）形象**，采用简约的黑白线条设计：

- **整体形态**：主体为白色，轮廓以黑色线条勾勒，造型圆润可爱，充满童趣。
- **头部特征**：头部呈椭圆形，顶部有两个竖立的耳朵（半圆形轮廓）；脸部中央有一个圆形的“鼻子区域”，内部
包含一个小三角形的嘴巴；眼睛是两个小圆点，位于脸颊两侧，简洁传神。
- **风格特点**：线条流畅、色彩对比鲜明（黑+白），属于平面化的简约设计，适合作为图标、表情包或装饰元素，给人轻松活泼的视觉感受。

这种设计常见于可爱风格的插画、品牌IP或社交媒体表情，通过极简的几何造型传递萌感。
```

# 流式输出

目前我们的agent每次对话的耗时可能会比较久，一方面是我们需要等到完整的响应到来之后才一次性打印，另一方也是因为很多模型在深度思考的过程中也会占用时间。LiteLLM也是支持[流式输出](https://docs.litellm.ai/docs/completion/stream)的,然后[深度思考](https://docs.litellm.ai/docs/reasoning_content)的数据放在`reasoning_content`里面。

所以我们只需要对我们的agent做简单的[改造](https://github.com/bluesky466/SimpleAgent/blob/main/brain_print_thinking.py)即可:

```python
from litellm import completion, stream_chunk_builder

def think(self, prompt):
    ...
    stream = completion(
        model=self.model,
        messages=self.messages,
        tools=self.tools_definition,
        stream=True, # 打开流式输出
    )
    message = self._read_response_stream(stream) # 读取流拼接成完整的数据
    ...

def _read_response_stream(self,stream):
    chunks = []
    is_thinking_start = False
    for chunk in stream:
        chunks.append(chunk)

        if not chunk.choices:
            continue
        delta = chunk.choices[0].delta
        reasoning = getattr(delta, "reasoning_content", None) or ""
        if not reasoning:
            continue

        if not is_thinking_start:
            is_thinking_start = True
            print("...思考中...")
        print(reasoning, end="", flush=True)
    if is_thinking_start:
        print("\n...思考结束...")
    return stream_chunk_builder(chunks, messages=self.messages).choices[0].message
```