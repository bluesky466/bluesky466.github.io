title: AIAgent - MCP
date: 2026-03-16 20:08:14
tags:
    - 技术相关
    - AI Agent
---

系列文章:

1. [AIAgent - 简易框架搭建](https://blog.islinjw.cn/2026/02/25/AIAgent-%E7%AE%80%E6%98%93%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/)
1. [AIAgent - LiteLLM](https://blog.islinjw.cn/2026/02/26/AIAgent-LiteLLM/)
1. [AIAgent - 视觉支持与流式输出](https://blog.islinjw.cn/2026/03/05/AIAgent-%E8%A7%86%E8%A7%89%E6%94%AF%E6%8C%81%E4%B8%8E%E6%B5%81%E5%BC%8F%E8%BE%93%E5%87%BA/)
1. [AIAgent - MCP](https://blog.islinjw.cn/2026/03/16/AIAgent-MCP/)

在之前的文章里我们已经知道agent里llm是如何去调用tool的,但我们之前的tool是直接写在python代码里面的。

如何支持用户接入第三方的工具给llm调用呢?这就要用到[mcp](https://modelcontextprotocol.io/docs/getting-started/intro)了。

mcp本质上是一套基于json的发送和接收数据的协议,用于告诉llm有哪些工具可以使用和如何使用:

```plantuml
package "MCP Host (AI Agent)" {
    rectangle "MCP Client1"
    rectangle "MCP Client2"
}
package "远程业务如天气查询" {
    rectangle "MCP Server1"
    rectangle "远程数据库"
}

package "本地业务如文件系统操作" {
rectangle "MCP Server2"
rectangle "本地文件系统"
}


"MCP Client1" <-left-> "MCP Server1" :  "      mcp协议(json)      "
"MCP Client2" <-right-> "MCP Server2" : "      mcp协议(json)      "

"MCP Server1" <--> "远程数据库"
"MCP Server2" <--> "本地文件系统"
```

从上面的图我们可以看到mcp协议里面分为三个角色:

- MCP Server : 业务系统搭建的服务，用于将本业务的功能通过llm可以理解的方式暴露出来
- MCP Host : 发起mcp请求的ai应用,如我们开发的ai agent
- MCP Client : MCP Host内部与MCP Server具体进行通讯的组件


除了三种角色之外,MCP还有三个重要的概念,client可以向server请求下面三种东西:

- Resources : 可以是文件系统、数据库等原本无法从网络中直接获取到的静态的资源,也可以是一些基础的配置信息。它以URI的方式去定义,例如后面定义的`weather://citys`这个资源。
- Tools : 提供给ai调用的工具,和我们之前定义的tool函数是一样的概念
- Prompts : 某些场景下的预设提示词,如格式化的输出某些内容,ai在需要的时候可以从mcp server选择Prompts去加载

由于协议本身使用json去通讯,所以可以很方便的使用各种语言去开发,官方也提供了各种语言的[sdk](https://modelcontextprotocol.io/docs/sdk),这里我依然使用[python](https://github.com/modelcontextprotocol/python-sdk)去开发。

实际上我们只需要pip去安装mcp的库就可以开始开发了:

```shell
pip install mcp
```

## MCP Server

我们这里先给出一个简单mcp server demo:

```python
# mcp_server.py
import logging

from mcp.server.fastmcp import FastMCP, Context
logging.getLogger("mcp").setLevel(logging.WARNING)

mcp = FastMCP("demo-weather-server")

@mcp.tool()
async def get_weather(city: str, ctx: Context) -> dict:
	"""Get the current weather for a city. should get support cities from resource weather://citys first"""
	weather_data = {
		"guangzhou": {"temp": 22, "condition": "sunny"},
		"shanghai": {"temp": 15, "condition": "cloudy"},
		"beijing": {"temp": 12, "condition": "rainy"},
	}

	city_lower = city.lower()
	if city_lower in weather_data:
		await ctx.session.send_tool_list_changed()
		return {"city": city, **weather_data[city_lower]}
	else:
		return {"city": city, "temp": "unknown", "condition": "unknown", "error": "city not supported"}
	
@mcp.resource("weather://citys")
async def citys() -> str:
	"""Get the list of cities."""
	return ["guangzhou", "shanghai", "beijing"]

if __name__ == "__main__":
	mcp.run(transport="stdio")
```

其实代码比较简单,无非是提供了`weather://citys`这个`resource`去获取城市列表,和`get_weather`这个`tool`去获取指定城市的天气信息。

PS: Prompts这里没有用到,但用法也是一样的直接使用`@mcp.prompt()`即可

## MCP JSON协议

Server的代码写好之后就可以写Client的代码去请求它,但在使用sdk编写client代码去调用之前我想先深入到mcp协议的原理来看看。

之前有说过mcp本质就是用json去发送和接收数据的协议,通讯链路有两种
 - 如果server运行在云端则通过[http](https://modelcontextprotocol.io/registry/remote-servers#transport-type)的`sse`或者`streamable-http`去传输数据
 - 如果是本地的server则直接将mcp server直接作为子进程启动通过标准输入输出去通讯

整个通讯的[生命周期](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle)如下:

![](lifecycle.png)

{% img /AIAgent-MCP/lifecycle.png %}

初始化有下面三步:

1. client发送初始化请求将自己的信息告诉server
2. server将自己的信息返回给client作为初始化响应
3. 客户端发送初始化完成的通知给服务端

例如我们可以直接通过`python3 mcp_server.py`启动mcp server,然后终端就会阻塞主等待我们的输入

这个时候我们可以手动输入json去模拟client发送初始化请求给server:

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"ExampleClient","version":"1.0.0","description":"An example MCP client application"}}}
```

server会返回初始化响应,打印在终端上:

```json
{"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2025-11-25","capabilities":{"experimental":{},"prompts":{"listChanged":false},"resources":{"subscribe":false,"listChanged":false},"tools":{"listChanged":false}},"serverInfo":{"name":"demo-weather-server","version":"1.26.0"}}}
```

然后我们再输入初始化完成的通知:

```json
{"jsonrpc":"2.0","method":"notifications/initialized"}
```

虽然server不会有任何的响应,所以终端上看是没有任何的反应,但实际上初始化已经完成了。

我们这个时候可以发送列出支持的tools的请求:

```json
{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{"cursor":"optional-cursor-value"}}
```

就能看到定义在server的`get_weather`了:

```json
{"jsonrpc":"2.0","id":1,"result":{"tools":[{"name":"get_weather","description":"Get the current weather for a city. should get support cities from resource weather://citys first","inputSchema":{"properties":{"city":{"title":"City","type":"string"}},"required":["city"],"title":"get_weatherArguments","type":"object"}}]}}
```

也可以使用`tools/call`方法去调用工具:

```json
{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_weather","arguments":{"city":"guangzhou"},"task":{"ttl":60000}}}
```

然后回在终端里面看到server返回的结果:

```json
{"jsonrpc":"2.0","id":1,"result":{"content":[{"type":"text","text":"{\n  \"city\": \"guangzhou\",\n  \"temp\": 22,\n  \"condition\": \"sunny\"\n}"}],"isError":false}}
```

## MCP Client

从上面的通讯原理我们可以看到,MCP协议其实并不复杂,client的demo我们使用`stdio_client`去通过标准输入输出调用`mcp_server.py`:


```python
import asyncio
import os
import sys

from mcp.client.session import ClientSession
from mcp.client.stdio import stdio_client, StdioServerParameters

async def main() -> None:
    script_dir = os.path.dirname(os.path.abspath(__file__))
    server_params = StdioServerParameters(
        command=sys.executable,
        args=["mcp_server.py"],
        cwd=script_dir,
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            init_result = await session.initialize()
            server_info = init_result.serverInfo
            print(f"Connected to server: {server_info.name} (version {server_info.version})\n")

            tools_result = await session.list_tools()
            print("Available tools:")
            for tool in tools_result.tools:
                print(f" - {tool.name}: {tool.description} {tool.inputSchema}")

            resources_result = await session.list_resources()
            print("Available resources (static):")
            for resource in resources_result.resources:
                print(f" - {resource.uri}: {resource.description}")

            print("\n" + "=" * 50 + "\n")

            tool_result = await session.call_tool("get_weather", {"city": "guangzhou"})
            print(f"get_weather result: {tool_result}")

            resource_result = await session.read_resource("weather://citys")
            print(f"Citys result: {resource_result.contents}")


if __name__ == "__main__":
    asyncio.run(main())
```

在命令行`python3 mcp_client.py`运行client就能看到server的一些信息了:

```shell
Connected to server: demo-weather-server (version 1.26.0)

Available tools:
 - get_weather: Get the current weather for a city. should get support cities from resource weather://citys first {'properties': {'city': {'title': 'City', 'type': 'string'}}, 'required': ['city'], 'title': 'get_weatherArguments', 'type': 'object'}
Available resources (static):
 - weather://citys: Get the list of cities.

==================================================

get_weather result: meta=None content=[TextContent(type='text', text='{\n  "city": "guangzhou",\n  "temp": 22,\n  "condition": "sunny"\n}', annotations=None, meta=None)] structuredContent=None isError=False
Citys result: [TextResourceContents(uri=AnyUrl('weather://citys'), mimeType='text/plain', meta=None, text='[\n  "guangzhou",\n  "shanghai",\n  "beijing"\n]')]
```

## Agent中接入

如何将mcp的信息转换成`completion`的`tools`参数可以参考我的[demo](https://github.com/bluesky466/SimpleAgent/blob/feature/mcp/tool/mcp_tool.py),本质上是一种json格式到另一种json格式的转换

接入完成之后就询问`广州天气怎样`就可以看到agent先通过`weather://citys`查询城市列表再通过`get_weather`获取到天气信息。然后我还加入了智谱的联网搜索api,在[配置文件](https://github.com/bluesky466/SimpleAgent/blob/feature/mcp/config.json)里填上你的key就可以通过网络调用到智谱的api去进行智能搜索:

<video src="https://blog.islinjw.cn/AIAgent-MCP/agent_mcp.mp4" controls width="600" style="display: block; margin: 0 auto;"></video>
