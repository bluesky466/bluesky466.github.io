title: 动手做一个AIAgent - 简易框架搭建
date: 2026-02-25 13:34:53
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

近年来部门内越来越多的使用ai去编程,加上最进openclaw爆火,一方面的确让开发的效率越来越高,另一方面我那该死的掌控欲又让我不断想去探究cursor、claude code这些工具的实现原理。

虽然最底层的llm工作原理实在处于我的知识盲区无能为力,但ai agent其实更偏应用层的工程实践是能被我所理解的。

## 本地模型

随着LLM的发展,很多可以本地部署的小模型的智能程度其实也已经挺高的了,用来做一些翻译、文档提取、文字校正之类的简单工作是完全没有问题的。

而且有了[ollama](https://ollama.com/)之后部署模型也只需要几个指令就能搞定:

```shell
# 安装ollama
curl -fsSL https://ollama.com/install.sh | sh

# 下载qwen3.5:9b这个模型
ollama pull qwen3.5:9b

# 运行qwen3.5:9b直接和qwen3.5:9b对话
ollama run qwen3.5:9b
```

可以在[模型列表](https://ollama.com/search)里面找到你想要的模型去下载运行即可。下载完成之后运行`ollama serve`可以在本地的`11434`端口启动ollama的服务器给代码调用。

## llm接口调用

本地模型虽然是可以协助干一些简单的活,但是如果要体验ai时代的快速发展我还是建议大家花点钱体验下完整版的llm。

例如国产的智谱、minimaxi、kimi、qwen、doubao其实价格并不是特别贵,而且购买也十分简单。

这一系列的博客我就会用[智谱](https://www.bigmodel.cn/glm-coding?ic=RXEAIC4TFX)去实现一个简单的[ai agent](https://github.com/bluesky466/SimpleAgent)。

智谱提供了"OpenAI 兼容模式"的API接口,基本和OpenAI的接口一样。实际上api使用起来也十分简单:

```python
class AgentBrain:
    def __init__(self, llm_config: dict):
        self._model = llm_config["model"]
        self._client = ZhipuAI(api_key=llm_config["api_key"])

    def think(self, prompt):
        try:
            message = self._client.chat.completions.create(
                model=self._model,
                messages=[{"role": "user", "content": prompt}],
            ).choices[0].message
            return message.content
        except Exception as e:
            return f"思考过程出错: {e}"
```

然后ai agent本质上是一个while循环在不断地询问llm:

```python
class SimpleAgent:
    def __init__(self, config: dict):
        self._brain = AgentBrain(config["llm"])
    
    def run(self):
        while True:
            try:
                user_input = input("请输入(Ctrl+C 退出): ")
            except KeyboardInterrupt:
                print("\n再见!")
                break
            if len(user_input) == 0:
                continue
            print("." * 20)
            response = self._brain.think(user_input)
            print(response)
            print("=" * 20)
```

之后只要在[配置文件](https://github.com/bluesky466/SimpleAgent/blob/feature/simple_loop/config.json)里面配置你的api key:

```json
{
    "llm": {
        "api_key" : "<这里填你的智谱apikey>",
        "model": "glm-4.7"
    }
}
```

就可以运行`python3 simple_agent.py`进行简单对话了:

```shell
请输入(Ctrl+C 退出): 你是谁
....................
我是GLM，由Z.ai开发的大语言模型。我被设计用来理解和生成人类语言，协助用户回答问题、提供信息以及参与各种对话。我会尽力提供准确、有帮助的
回应，同时尊重用户的隐私。

有什么我能帮助您的问题或者想了解的内容吗？
====================
```

## 添加记忆

由于llm本身不会保持历史记录所以需要我们每次对话将之前的历史记录也发送给它,实现ai agent的[记忆功能](https://github.com/bluesky466/SimpleAgent/blob/feature/memory/agent_brain.py):

```python
class AgentBrain:
    def __init__(self, llm_config: dict, memory: AgentMemory):
        self._model = llm_config["model"]
        self._client = ZhipuAI(api_key=llm_config["api_key"])
        self._memory = memory

    def think(self, prompt):
        try:
            self._memory.add_user_prompt(prompt)
            response = self._client.chat.completions.create(
                model=self._model,
                messages=self._memory.get_memory(),
            ).choices[0].message
            self._memory.add_agent_response(response)
            return response.content
        except Exception as e:
            return f"思考过程出错: {e}"
```

从AgentMemory的代码可以看到我们按不同角色去保存聊天记录:

```python
class AgentMemory:
    def __init__(self):
        self._memory = [{"role": "system", "content": self._get_system_prompt()},]
    
    def _get_system_prompt(self):
        return f"""
        你是一个AI智能助手.
        """

    def _parse_response_message(self, role, message):
        result = {
            "role": role,
            "content": message.content,
        }
        return result

    def add_user_prompt(self, prompt):
        self._memory.append({"role": "user", "content": prompt})
    
    def add_agent_response(self, message):
        self._memory.append(self._parse_response_message("assistant", message))
    
    def get_memory(self):
        return self._memory
```

这三个角色分别是:

- system : 可以在对话开始前对模型进行全局的角色设定
- user : 是用户发送的消息
- assistant : 是llm作为智能助手回复的消息

 这样从表现上llm就具有了记忆:

```shell
请输入(Ctrl+C 退出): 我是ljw
....................
你好，ljw！很高兴认识你。

我是AI智能助手，请问有什么我可以帮你的吗？
====================
请输入(Ctrl+C 退出): 你知道我是谁吗
....................
在这个对话中，我知道你叫 **ljw**，因为这是你刚才告诉我的名字。

除此之外，我并不了解你的真实身份、背景或其他隐私信息。我只是一个人工智能助手，所有的记忆都仅限于我们当前的对话内容。
====================
```

## 工具使用

此时的agent只能做简单的聊天,我们需要提供一些[工具](https://github.com/bluesky466/SimpleAgent/blob/feature/tool_by_prompt/tool/local_tool.py)给它去操作电脑让他可以做更多事情:

```python
class LocalToolProvider(ToolProvider):
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
    
    def write_file(self, path: str, content: str):
        """
        写入内容到指定文件
        Args:
            path: 要写入的文件路径
            content: 要写入的内容
        """
        with open(os.path.expanduser(path), "w") as f:
            f.write(content)
    ...
```

可以看到我们提供了`list_dir`、`read_file`、`write_file`三个函数给llm使用。但llm要怎么去调用这三个函数呢?

我们可以在[系统提示词](https://github.com/bluesky466/SimpleAgent/blob/feature/tool_by_prompt/agent_memory.py)里面告诉它有这些工具可以使用,并约定调用工具的输出格式:

```python
def _get_system_prompt(self, tool_definition: str):
    runtime = f"{platform.system()} {platform.machine()}, Python {platform.python_version()}"
    return f"""
你是一个AI智能助手.

## 运行环境

{runtime}

## 可用工具列表

{tool_definition}

## 工具调用方法

当你需要调用工具的时候,严格按照下面格式输出,我会判断返回的第一个字符是'{{'且最后一个字符是'}}'就去调用工具:
{{
    "message":"你想说的话"
    "tool_name": "工具名称",
    "tool_args": {{
        "参数名称": "参数值"
    }}
}}"""
```

完整的提示词如下:

```
你是一个AI智能助手.

## 运行环境
Darwin arm64, Python 3.10.19

## 可用工具列表
### list_dir

        列出指定目录下的文件和目录
        Args:
            path: 要列出的目录路径
        Returns:
            目录下的文件和目录列表
        
### read_file

        读取指定文件的内容
        Args:
            path: 要读取的文件路径
        Returns:
            文件内容
        
### write_file

        写入内容到指定文件
        Args:
            path: 要写入的文件路径
            content: 要写入的内容
        

## 工具调用方法
当你需要调用工具的时候,严格按照下面格式输出,我会判断返回的第一个字符是'{'且最后一个字符是'}'就去调用工具:
{
    "message":"你想说的话"
    "tool_name": "工具名称",
    "tool_args": {
        "参数名称": "参数值"
    }
}
```

我们在system的角色设定里面将当前的系统运行环境、可以使用的工具通通告诉llm,然后在[消息循环](https://github.com/bluesky466/SimpleAgent/blob/feature/tool_by_prompt/simple_agent.py)里判断llm的响应是否为调用工具函数,如果是就调用工具函数将结果保存到聊天记录,再让llm执行下一步动作,这样就可以实现agent操控电脑的功能了:

```python
class SimpleAgent:
    ...
        
    def run(self):
        while True:
            try:
                user_input = input("请输入(Ctrl+C 退出): ")
            except KeyboardInterrupt:
                print("\n再见!")
                break
            if len(user_input) == 0:
                continue
            print("." * 20)
            response = self._brain.think(user_input)

            while response.content.strip().startswith("{"):
                tool_call = json.loads(response.content.strip())
                tool_name = tool_call["tool_name"]
                tool_args = tool_call["tool_args"]
                result = self._tool_manager.exec(tool_name, tool_args)
                self._memory.add_tool_invoke_result(tool_name, tool_args, result)
                response = self._brain.think("思考执行结果并决定下一步行动.")
            print(response.content)
            print("=" * 20)
```

我在下载目录写了一个`test.py`代码:

```python
print("hello world!")
```

然后让它帮我修改成打印`hello agent!`:

```shell
请输入(Ctrl+C 退出): 下载目录有什么
....................
调用工具: list_dir, 参数: {'path': '~/Downloads'}
工具调用结果: [".DS_Store", "test.py"]
下载目录下有以下文件和文件夹：

1. **.DS_Store** - macOS系统自动生成的隐藏文件
2. **test.py** - 一个Python脚本文件

目前下载目录中就这两个文件。如果你需要查看 test.py 文件的内容，我可以帮你读取。
====================
请输入(Ctrl+C 退出): test.py是干什么的
....................
调用工具: read_file, 参数: {'path': '~/Downloads/test.py'}
工具调用结果: "print(\"hello world\")\n"
test.py 是一个非常简单的Python脚本文件，它的内容只有一行代码：

print("hello world")

这个脚本的作用是：在运行时打印输出 "hello world" 到控制台。这通常是编程入门时第一个写的程序，用来测试Python
环境是否正常工作。
====================
请输入(Ctrl+C 退出): 帮我改成打印"hello agent"
....................
调用工具: write_file, 参数: {'path': '~/Downloads/test.py', 'content': 'print("hello agent")\n'}
工具调用结果: null
已经成功修改了 test.py 文件！现在它的内容是：

print("hello agent")

当你运行这个脚本时，它将会打印输出 "hello agent"。
====================
```

最终查看修改结果也是正确的:

```shell
$ cat ~/Downloads/test.py
print("hello agent!")
$ python ~/Downloads/test.py
hello agent!
```

## 使用tools传递工具函数信息

像上面那样直接将工具函数信息写到prompt里面容易出现参数缺失、格式不统一、幻觉的问题。例如我明明已经让他严格按照格式输出了,他还是会多出一些文字:

```
请输入(Ctrl+C 退出): 下载目录有什么
....................

我来帮您查看下载目录的内容。
{
    "message": "我将帮您查看下载目录的内容",
    "tool_name": "list_dir",
    "tool_args": {
        "path": "~/Downloads"
    }
}
```

其实很多llm原生支持按结构化格式输入工具信息,例如通过[tools参数](https://github.com/bluesky466/SimpleAgent/blob/feature/tool_by_tools/agent_brain.py)输入:

```python
class AgentBrain:
    def __init__(self, llm_config: dict, memory: AgentMemory, tool_manager: ToolManager):
        self._model = llm_config["model"]
        self._client = ZhipuAI(api_key=llm_config["api_key"])
        self._memory = memory
        self._tools_definition = tool_manager.get_tool_definition()

    def think(self, prompt):
        try:
            self._memory.add_user_prompt(prompt)
            message = self._client.chat.completions.create(
                model = self._model,
                messages = self._memory.get_memory(),
                tools=self._tools_definition,
            ).choices[0].message
            self._memory.add_agent_response(message)
            return message
        except Exception as e:
            return f"思考过程出错: {e}"
```

然后通过tools参数传给llm,接着就可以通过llm返回里面的[tool_calls响应](https://github.com/bluesky466/SimpleAgent/blob/feature/tool_by_tools/simple_agent.py)看到是否需要调用工具函数:

```python
class SimpleAgent:
    ...
    def run(self):
        while True:
            try:
                user_input = input("请输入(Ctrl+C 退出): ")
            except KeyboardInterrupt:
                print("\n再见!")
                break
            if len(user_input) == 0:
                continue
            print("." * 20)
            response = self._brain.think(user_input)

            while hasattr(response, "tool_calls") and response.tool_calls:
                for tool_call in response.tool_calls:
                    id = tool_call.id
                    tool_name = tool_call.function.name
                    args = json.loads(tool_call.function.arguments)
                    result = self._tool_manager.exec(tool_name, args)
                    self._memory.add_tool_invoke_result(id, tool_name, args, result)
                response = self._brain.think("思考执行结果并决定下一步行动.")
            print(response.content)
            print("=" * 20)
```

`tool_calls`响应一定是json结果的数据,这样一来就能保证工具调用响应的准确性。

## 事件循环

这样一来我们就一步步实现了一个简单的ai agent，完整的demo也放到了[github](https://github.com/bluesky466/SimpleAgent/tree/feature/tool_by_tools)上。它的事件循环如下:

{% plantuml %}
state "读取用户输入" as ReadUserInput
state "请求LLM(携带历史记录)" as RequestLLM
state "等待LLM响应" as WaitLLMResponse
state "调用工具得到结果" as UseTool
state "输出响应" as OutputResponse

state c <<choice>>
 
ReadUserInput --> RequestLLM
RequestLLM --> WaitLLMResponse
WaitLLMResponse --> c
c --> UseTool : 需要调用工具
c --> OutputResponse : 不需要调用工具
UseTool --> RequestLLM
OutputResponse --> ReadUserInput
{% endplantuml %}