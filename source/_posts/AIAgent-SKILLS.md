title: AIAgent - SKILLS
date: 2026-03-19 20:06:39
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
1. [AIAgent - RAG基础](https://blog.islinjw.cn/2026/03/27/AIAgent-RAG%E5%9F%BA%E7%A1%80/)

在上一篇博客我们讲了mcp的原理,但就近期ai圈来看mcp的热点已经逐渐降低了,大家更多开始关注[skills](https://agentskills.io/what-are-skills).

从原理上来讲skills也是一个比较简单的东西,基本上就是人类或者llm可读的操作文档按照一定格式存放:

```shell
my-skill/
├── SKILL.md          # 必要: 具体的操作文档 + 元数据
├── scripts/          # 可选: 脚本
├── references/       # 可选: 二级文档
└── assets/           # 可选: 各种资源
```

例如我demo里面的[clean-up-computer](https://github.com/bluesky466/SimpleAgent/tree/main/skills/clean-up-computer)这个skill。其中对格式要求最强的就是这个SKILL.md了,它要求文档的开头以YAML的格式定义文档的元数据。

```
---
name: 清理电脑垃圾
description: 删除电脑上无用的文件
---

各个目录的文件删除规则如下

- ~/Downloads : references/下载目录清理规则.md
- ~/Documents :  使用scripts/clean_documents.py去清理,该脚本需要你传入需要清理的绝对路径
- ~/Movies : 所有非视频类的文件都可以删除
```

例如上面这个skill,它的名字就是`清理电脑垃圾`,概要描述就是`删除电脑上无用的文件`。

agent在初始化的时候会扫描skills目录下的SKILL.md,[加载](https://github.com/bluesky466/SimpleAgent/blob/main/skill_loader.py)他们的元数据作为系统提示词。也就是说每次传给llm的只是元数据的内容,这样消耗的token就会很少:

```python
class SkillLoader:
    ...
    def _load_skills(self):
        skills = []
        for file in os.listdir(self._skill_dir):
            skill_path = os.path.join(self._skill_dir, file, "SKILL.md")
            if not os.path.exists(skill_path):
                continue
            with open(skill_path, "r") as f:
                content = f.read()
                metadata = self._get_skill_metadata(content)
                metadata["location"] = skill_path
                skills.append(metadata)
        return skills

    def _get_skill_metadata(self, content: str) -> dict | None:
        if not content.startswith("---"):
            return None
        match = re.match(r"^---\n(.*?)\n---", content, re.DOTALL)
        if not match:
            return None
        metadata = {}
        for line in match.group(1).split("\n"):
            if ":" in line:
                key, value = line.split(":", 1)
                metadata[key.strip()] = value.strip().strip('"\'')
        return metadata
    ...
```

然后我们在系统提示词里将skills列出来给ai,并且要求它在需要的时候再向用户请求加载完整的内容:

```python
class SkillLoader:
    ...
    def get_skills_prompt(self):
        prompt = "下面是你的SKILLS列表,在需要的时候向用户请求加载skill权限通过后读取对应SKILL的完整描述并执行. \n"
        prompt += "<skills>\n"
        for skill in self._load_skills():
            prompt += f"    <skill>\n"
            prompt += f"        <name>{skill['name']}</name>\n"
            prompt += f"        <description>{skill['description']}</description>\n"
            prompt += f"        <location>{skill['location']}</location>\n"
            prompt += f"    </skill>\n"
        prompt += "</skills>\n"
        return prompt

class AgentMemory:
    ...
    def _get_system_prompt(self, skills_prompt: str):
        runtime = f"{platform.system()} {platform.machine()}, Python {platform.python_version()}"
        return f"""
你是一个AI智能助手.

## 运行环境
{runtime}

## SKILLS
{skills_prompt}
        """
```

完整的系统提示词打印出来是这样的:

```
你是一个AI智能助手.

## 运行环境
Darwin arm64, Python 3.12.12

## SKILLS
下面是你的SKILLS列表,在需要的时候向用户请求加载skill权限通过后读取对应SKILL的完整描述并执行.
<skills>
    <skill>
        <name>清理电脑垃圾</name>
        <description>删除电脑上无用的文件</description>
        <location>/Users/linjw/workspace/py-agent/skills/clean-up-computer/SKILL.md</location>
    </skill>
</skills>
```

llm在接收到用户`清理电脑`的指令的时候就可以从skills列表里面找到`清理电脑垃圾`这个skill并且知道它的路径是`/Users/linjw/workspace/py-agent/skills/clean-up-computer/SKILL.md`,然后他就会通过我们新增加的[request_load_skill_permission](https://github.com/bluesky466/SimpleAgent/blob/main/tool/local_tool.py)这个工具去请求权限:

```python
class LocalToolProvider(ToolProvider):
    def __init__(self, skill_loader: SkillLoader):
        self._skill_loader = skill_loader
    ...
    def read_file(self, path: str):
        """
        读取指定文件的内容
        Args:
            path: 要读取的文件路径
        Returns:
            文件内容
        """
        with open(os.path.expanduser(path), "r") as f:
            return f.read()

    def request_load_skill_permission(self, skill_name: str):
        """
        请求加载skill的权限
        Args:
            skill_name: 需要加载的skill的名字
        """
        return self._skill_loader.request_load_skill_permission(skill_name)

class SkillLoader:
    def request_load_skill_permission(self, skill_name: str):
        user_input = input(f"是否运行加载skill {skill_name} (Y/N): ")
        return user_input.upper() == "Y"
```

一旦用户输入y或者Y,它就会用我们之前定义的`read_file`工具去读取完整的[SKILL.md](https://github.com/bluesky466/SimpleAgent/blob/main/skills/clean-up-computer/SKILL.md):

```
---
name: 清理电脑垃圾
description: 删除电脑上无用的文件
---

各个目录的文件删除规则如下

- ~/Downloads : references/下载目录清理规则.md
- ~/Documents :  使用scripts/clean_documents.py去清理,该脚本需要你传入需要清理的绝对路径
- ~/Movies : 所有非视频类的文件都可以删除
```

agent就会根据定义的规则先使用`read_file`工具读取[下载目录清理规则.md](https://github.com/bluesky466/SimpleAgent/blob/main/skills/clean-up-computer/references/%E4%B8%8B%E8%BD%BD%E7%9B%AE%E5%BD%95%E6%B8%85%E7%90%86%E8%A7%84%E5%88%99.md)的内容:

```
下载目录的所有文件都可以删除
```

然后再使用`list_dir`工具列出`~/Downloads`目录的所有文件,然后一个个调用`remove_file`去删除(为了安全我这里只是打印了下并没有真的删除文件):

```python
class LocalToolProvider(ToolProvider):
    ...
    def list_dir(self, path: str):
        """
        列出指定目录下的文件和目录
        Args:
            path: 要列出的目录路径
        Returns:
            目录下的文件和目录列表
        """
        expanded_path = os.path.expanduser(path)
        try:
            return os.listdir(expanded_path)
        except FileNotFoundError as e:
            return str(e)

    def remove_file(self, path: str):
        """
        删除指定了解的文件
        Args:
            path: 要删除的文件路径
        """
        print(f"模拟删除文件: {path}")
        return True
    ...
```

然后再去调用我们新增的`exec_skill_py_script`工具执行scripts/clean_documents.py去清理`~/Documents`目录:

```python
def exec_skill_py_script(self, script_path: str, arguments: list[str]):
    """
    执行指定路径的Python脚本
    Args:
        script_path: 要执行的脚本路径
        arguments: 要执行的脚本参数列表,类型为list[str]
    """
    return self._skill_loader.exec_skill_py_script(script_path, arguments)
```

我们在SKILL.md里面说明了这个`clean_documents.py`的使用方式,但实际上即使我们不说,足够聪明的模型也会自己调用`read_file`工具去读取脚本内容然后自己分析出应该如何调用。

最后的`~/Movies`同样是用`list_dir`列出文件然后再由llm去判断是否为视频文件再决定是否删除

整个demo完整的运行视频如下:

<video src="https://blog.islinjw.cn/AIAgent-SKILLS/video.mp4" controls width="600" style="display: block; margin: 0 auto;"></video>
