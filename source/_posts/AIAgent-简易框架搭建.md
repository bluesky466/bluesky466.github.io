title: AIAgent - 简易框架搭建
date: 2026-02-25 13:34:53
tags:
    - 技术相关
    - AI Agent
---

近年来部门内越来越多的使用ai去编程,加上最进openclaw爆火,一方面的确让开发的效率越来越高,另一方面我那该死的掌控欲又让我不断想去探究cursor、claude code这些工具的实现原理。

虽然最底层的llm工作原理实在处于我的知识盲区无能为力,但ai agent其实更偏应用层的工程实践是能被我所理解的。

## llm接口调用

ai agent本质上是一个while循环在不断地询问llm:

```python
class SimpleAgent:
    def __init__(self):
        self.brain = AgentBrain()
    
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
            response = self.brain.think(user_input)
            print(response)
            print("=" * 20)
        
if __name__ == "__main__":
    my_agent = SimpleAgent()
    my_agent.run()
```

这里我选用国内的[智谱](https://www.bigmodel.cn/glm-coding?ic=RXEAIC4TFX)作为Agent的大脑,它提供了"OpenAI 兼容模式"的API接口基本和OpenAI的接口一样,实际上api使用起来也十分简单:

```python
from zhipuai import ZhipuAI
import os

class AgentBrain:
    def __init__(self):
        self.model = "glm-4.7"
        self.client = ZhipuAI(api_key=os.environ.get("ZAI_API_KEY"))

    def think(self, prompt):
        try:
            message = self.client.chat.completions.create(
                model=self.model,
                messages=[{"role": "user", "content": prompt}],
            ).choices[0].message
            return message.content
        except Exception as e:
            return f"思考过程出错: {e}"
```

之后只要先设置apikey的环境变量:

```shell
export ZAI_API_KEY="<你的智谱api key>"
```

就可以运行py脚本进行简单对话了:

```shell
请输入(Ctrl+C 退出): 你是谁
....................
我是GLM，由Z.ai开发的大语言模型。我被设计用来理解和生成人类语言，协助用户回答问题、提供信息以及参与各种对话。我会尽力提供准确、有帮助的
回应，同时尊重用户的隐私。

有什么我能帮助您的问题或者想了解的内容吗？
====================
```

# 添加记忆

由于llm本身不会保持历史记录所以需要我们每次对话将之前的历史记录也发送给它,实现ai agent的记忆功能:

```python
from zhipuai import ZhipuAI
import os

class AgentBrain:
    def __init__(self):
        self.model = "glm-4.7"
        self.client = ZhipuAI(api_key=os.environ.get("ZAI_API_KEY"))
        self.messages = [{"role": "system", "content": self.__get_system_prompt()},]
    
    def __get_system_prompt(self):
        return f"""
        你是一个AI智能助手.
        """
    def think(self, prompt):
        try:
            self.messages.append({"role": "user", "content": prompt})
            message = self.client.chat.completions.create(
                model=self.model,
                messages=self.messages,
            ).choices[0].message
            self.messages.append(self.parse_response_message("assistant", message))
            return message.content
        except Exception as e:
            return f"思考过程出错: {e}"
    
    def parse_response_message(self, role, message):
        result = {
            "role": role,
            "content": message.content,
        }
        return result
```

从上面的代码也可以看到我们可以按不同角色保存聊天记录,例如system可以在对话开始前对模型进行角色的设定,user是用户发送的消息,而assistant是llm作为智能助手回复的消息。这样从表现上llm就具有了记忆:


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

此时的agent只能做简单的聊天,我们需要提供一些工具给它去操作电脑让他可以做更多事情:

```python
import os
import json
from docstring_parser import parse as parse_docstring

class AgentTools:
    __TYPE_MAP = {"str": "string", "int": "integer", "float": "number", "bool": "boolean"}
    __EXCLUDED = {"get_definition_for_prompt", "exec"}

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
    
    def exec(self, func_name, arguments):
        func = getattr(self, func_name)
        return json.dumps(func(**arguments))

    def get_definition_for_prompt(self):
        tools_definition = []
        for attr_name in dir(self):
            if attr_name.startswith("_") or attr_name in self.__EXCLUDED:
                continue
            attr = getattr(self, attr_name)
            if not callable(attr):
                continue
            func = getattr(self.__class__, attr_name, None)
            if func is None:
                continue
            tools_definition.append(f"###{attr_name}\n{func.__doc__}")
        return tools_definition
```

可以看到我们提供了`list_dir`、`read_file`、`write_file`三个工具给llm使用,而`get_definition_for_prompt`用于获取工具函数的注释文档,`exec`用于调用工具函数。

在system的角色设定里面将当前的系统运行环境、可以使用的工具通通告诉llm,然后在llm的响应里面判断它是否需要调用工具函数,如果是就调用工具函数将结果保存到聊天记录,再让llm执行下一步动作,这样就可以实现agent操控电脑的功能了:

```python
from zhipuai import ZhipuAI
import os
import json
import platform

class AgentBrain:
    def __init__(self, tools):
        self.model = "glm-4.7"
        self.tools = tools
        self.client = ZhipuAI(api_key=os.environ.get("ZAI_API_KEY"))
        self.tools_definition = "\n".join(tools.get_definition_for_prompt())
        self.messages = [{"role": "system", "content": self.__get_system_prompt()},]
    
    def __get_system_prompt(self):
        runtime = f"{platform.system()} {platform.machine()}, Python {platform.python_version()}"
        return f"""
        你是一个AI智能助手.

        ## 运行环境
        {runtime}

        ## 可用工具列表
        {self.tools_definition}

        ## 工具调用方法
        直接返回以下json格式的工具调用参数，不要包含任何其他内容:
        {{
            "tool_name": "工具名称",
            "tool_args": {{
                "参数名称": "参数值"
            }}
        }}"""
    def think(self, prompt):
        try:
            self.messages.append({"role": "user", "content": prompt})
            message = self.client.chat.completions.create(
                model=self.model,
                messages=self.messages,
            ).choices[0].message
            self.messages.append(self.parse_response_message("assistant", message))
            if message.content.startswith("{") or message.content.startswith("```json"):
                tool_call = json.loads(message.content.strip("```json").strip("```"))
                self.messages.append({
                    "role": "tool",
                    "tool_name": tool_call["tool_name"],
                    "tool_args": tool_call["tool_args"],
                    "content": self.tools.exec(tool_call["tool_name"], tool_call["tool_args"]),
                })
                return self.think("思考执行结果并决定下一步行动.")
            else:
                return message.content
        except Exception as e:
            return f"思考过程出错: {e}"
    
    def parse_response_message(self, role, message):
        result = {
            "role": role,
            "content": message.content,
        }
        return result
```

我在下载目录写了一个`test.py`代码:

```python
print("hello world!")
```

然后让它帮我修改成打印`hello agent!`:

```shell
请输入(Ctrl+C 退出): 下载目录有什么
....................
根据查询结果，您的下载目录（~/Downloads）下包含以下文件：

1. **recv.txt** - 文本文件
2. **test.py** - Python脚本文件
3. **send.txt** - 文本文件
4. **录屏.mov** - 视频文件（录屏文件）

如果您需要查看某个文件的具体内容，请告诉我文件名，我可以帮您读取。
====================
请输入(Ctrl+C 退出): test.py 脚本做了什么事情
....................
test.py 脚本非常简单，它只做了一件事：

**打印 "hello world!" 到控制台**

具体来说，当运行这个脚本时，它会执行 print("hello world!") 语句，在终端或命令行中输出 hello world! 这段文字。

这是一个最基础的Python程序，通常用于测试Python环境是否正确安装和配置。
====================
请输入(Ctrl+C 退出): 帮我改成帮我改成打印"hello agent!"
....................
修改完成！我已经将 test.py 文件的内容更新为：

print("hello agent!")

现在当您运行这个脚本时，它会在控制台输出 "hello agent!" 而不是之前的 "hello world!"。

您可以通过运行 python ~/Downloads/test.py 来验证修改是否成功。
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

像上面那样直接将工具函数信息写到prompt里面容易出现参数缺失、格式不统一、幻觉的问题。可以看到我上面的代码就有将前后的'\`\`\`json'和'\`\`\`'去掉的代码,实际上有时候它的输出会带这个但有时候不会,有时候又会有其他的东西出现或者缺少。

其实很多llm原生支持按结构化格式解析工具信息,这里为`AgentTools`增加一个`get_definition_for_json`函数用于解析函数的文档然后打包成json格式:

```python
class AgentTools:
    ...

    def get_definition_for_json(self):
        tools_definition = []
        for attr_name in dir(self):
            if attr_name.startswith("_") or attr_name in self.__EXCLUDED:
                continue
            attr = getattr(self, attr_name)
            if not callable(attr):
                continue
            func = getattr(self.__class__, attr_name, None)
            if func is None:
                continue
            doc_info = parse_docstring(func.__doc__ or "")
            params, required_params = {}, []
            for param in doc_info.params:
                ann = func.__annotations__.get(param.arg_name)
                type_name = getattr(ann, "__name__", "string") if ann else "string"
                params[param.arg_name] = {
                    "type": self.__TYPE_MAP.get(type_name, type_name),
                    "description": param.description or ""
                }
                required_params.append(param.arg_name)
            tools_definition.append({
                "type": "function",
                "function": {
                    "name": attr_name,
                    "description": doc_info.short_description or "",
                    "parameters": {"type": "object", "properties": params, "required": required_params}
                }
            })
        return tools_definition
```

然后通过tools参数传给llm,接着就可以通过llm返回里面的`tool_calls`看到是否需要调用工具函数:

```python
from zhipuai import ZhipuAI
import os
import json
import platform

class AgentBrain:
    def __init__(self, tools):
        self.model = "glm-4.7"
        self.tools = tools
        self.client = ZhipuAI(api_key=os.environ.get("ZAI_API_KEY"))
        self.tools_definition = tools.get_definition_for_json()
        self.messages = [{"role": "system", "content": self.__get_system_prompt()},]
    
    def __get_system_prompt(self):
        runtime = f"{platform.system()} {platform.machine()}, Python {platform.python_version()}"
        return f"""
        你是一个AI智能助手.

        ## 运行环境
        {runtime}
        """
        
    def think(self, prompt):
        try:
            self.messages.append({"role": "user", "content": prompt})
            
            message = self.client.chat.completions.create(
                model=self.model,
                messages=self.messages,
                tools=self.tools_definition
            ).choices[0].message
            self.messages.append(self.parse_response_message(message))
            if hasattr(message, "tool_calls") and message.tool_calls:
                for tool_call in message.tool_calls:
                    args = tool_call.function.arguments
                    if isinstance(args, str):
                        args = json.loads(args)

                    self.messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "name": tool_call.function.name,
                        "content": self.tools.exec(tool_call.function.name, args),
                    })
                return self.think("思考执行结果并决定下一步行动.")
            else:
                return message.content
        except Exception as e:
            return f"思考过程出错: {e}"
    
    def parse_response_message(self, message):
        result = {
            "role": message.role,
            "content": message.content,
            "reasoning_content": message.reasoning_content,
        }
        if hasattr(message, "tool_calls") and message.tool_calls:
            result["tool_calls"] = [
                {"id":tc.id, "type":tc.type, "function":{"name":tc.function.name, "arguments":tc.function.arguments}} for tc in message.tool_calls
            ]
        return result
```

这样一来就能保证工具调用响应的准确性

## 事件循环

这样一来我们就一步步实现了一个简单的ai agent，完整的demo也放到了[github](https://github.com/bluesky466/SimpleAgent)上。它的事件循环如下:

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