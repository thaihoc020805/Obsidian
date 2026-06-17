# SSE Server

SSE (Server Sent Events) is a ==standard== for ==server-to-client streaming==, ==allowing servers to push real-time updates to clients over HTTP==. This is particularly(đặc biệt) useful for ==applications that require live updates==, such as ==chat applications==, ==notifications==, or ==real-time data feeds==. Also, your ==server== can be ==used== by ==multiple clients at the same time== as it lives on a server that can be ==run somewhere== in the cloud for example.

## Overview

This lesson covers how to build and consume SSE Servers.

## Learning Objectives

By the end of this lesson, you will be able to:

- ==Build an SSE Server.==
- ==Debug an SSE Server using the Inspector.==
- ==Consume an SSE Server using Visual Studio Code.==


## SSE, ==how it works==

SSE is one of two supported transport types. You've already seen the first one stdio being used in previous lessons. The difference is the following:

- ==SSE needs you to handle two things; connection and messages.==
- As this is a ==server that can live anywhere==, you need that to reflect in how you work with tools like the Inspector and Visual Studio Code. What that means is that ==instead of pointing out how to start the server==, you instead ==point to the endpoint where it can establish a connection==. See below example code:

### TypeScript

```typescript
app.get("/sse", async (_: Request, res: Response) => {
    const transport = new SSEServerTransport('/messages', res);
    transports[transport.sessionId] = transport;
    res.on("close", () => {
        delete transports[transport.sessionId];
    });
    await server.connect(transport);
});

app.post("/messages", async (req: Request, res: Response) => {
    const sessionId = req.query.sessionId as string;
    const transport = transports[sessionId];
    if (transport) {
        await transport.handlePostMessage(req, res);
    } else {
        res.status(400).send('No transport found for sessionId');
    }
});
```

In the preceding code:

- ==`/sse`== is set up as a ==route==. When a ==request== is made ==towards this route==, a ==new transport instance is created== and ==the server *connects* using this transport==
- ==`/messages`==, this is the ==route== that ==handles incoming messages.==

### Python

```python
mcp = FastMCP("My App")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

# Mount the SSE server to the existing ASGI server
app = Starlette(
    routes=[
        Mount('/', app=mcp.sse_app()),
    ]
)

```

In the preceding code we:

- ==Create an instance of an ASGI server== (using Starletter specifically) and ==mount the default route ==

  What happens ==behind the scenes== is that the routes ==`/sse` and `/messages`== are setup to ==handle connections and messages respectively==. The rest of the app, like adding features like tools, happens like with stdio servers.

### .NET    

```csharp
    var builder = WebApplication.CreateBuilder(args);
    builder.Services
        .AddMcpServer()
        .WithTools<Tools>();


    builder.Services.AddHttpClient();

    var app = builder.Build();

    app.MapMcp();
    ```

    There are two methods that helps us go from a web server to a web server supporting SSE and that is:

    - `AddMcpServer`, this method adds capabilities.
    - `MapMcp`, this adds routes like `/SSE` and `/messages`.


Now that we know a little bit more about SSE, let's build an SSE server next.

## Exercise: Creating an SSE Server

To create our server, we need to keep two things in mind:

- ==We need to use a web server to expose endpoints for connection and messages.==
- ==Build our server like we normally do with tools, resources and prompts when we were using stdio.==

### -1- Create a server instance

To create our server, we use the same types as with stdio. However, for the transport, we need to choose SSE.

### TypeScript

```typescript
import { Request, Response } from "express";
import express from "express";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";

const server = new McpServer({
  name: "example-server",
  version: "1.0.0"
});

const app = express();

const transports: {[sessionId: string]: SSEServerTransport} = {};
```

In the preceding code we've:

- Created a server instance.
- Defined an app using the web framework express.
- Created a transports variable that we will use to store incoming connections.

### Python

```python
from starlette.applications import Starlette
from starlette.routing import Mount, Host
from mcp.server.fastmcp import FastMCP


mcp = FastMCP("My App")
```

In the preceding code we've:

- Imported the libraries we're going to need with Starlette (an ASGI framework) being pulled in.
- Created an MCP server instance `mcp`.

### .NET

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services
    .AddMcpServer();


builder.Services.AddHttpClient();

var app = builder.Build();

// TODO: add routes 
```

At this point, we've:

- Created a web app
- Added support for MCP features through `AddMcpServer`.

Let's add the needed routes next.

### -2- ==Add routes==

Let's ==add routes== next that ==handle the connection and incoming messages:==

### TypeScript

```typescript
app.get("/sse", async (_: Request, res: Response) => {
  const transport = new SSEServerTransport('/messages', res);
  transports[transport.sessionId] = transport;
  res.on("close", () => {
    delete transports[transport.sessionId];
  });
  await server.connect(transport);
});

app.post("/messages", async (req: Request, res: Response) => {
  const sessionId = req.query.sessionId as string;
  const transport = transports[sessionId];
  if (transport) {
    await transport.handlePostMessage(req, res);
  } else {
    res.status(400).send('No transport found for sessionId');
  }
});

app.listen(3001);
```

In the preceding code we've defined:

- An `/sse` route that instantiates a transport of type SSE and ends up calling `connect` on the MCP server.
- A `/messages` route that takes care of incoming messages.

### Python

```python
app = Starlette(
    routes=[
        Mount('/', app=mcp.sse_app()),
    ]
)

or

app = FastAPI()
app.mount('/', mcp.sse_app())

```

In the preceding code we've:

- ==Created an ASGI app instance== using the Starlette framework. As part of that we passes `mcp.sse_app()` to ==it's list of routes==. T==hat ends up mounting an `/sse` and `/messages` route on the app instance.==
	
### .NET

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services
    .AddMcpServer();

builder.Services.AddHttpClient();

var app = builder.Build();

app.MapMcp();
```

We've added one line of code at the end `add.MapMcp()` this means we now have routes `/SSE` and `/messages`. 

Let's add capabilties to the server next.

### -3- ==Adding server capabilities==

Now that we've got everything SSE specific defined, let's ==add server capabilities like tools, prompts and resources.==

### TypeScript

```typescript
server.tool("random-joke", "A joke returned by the chuck norris api", {},
  async () => {
    const response = await fetch("https://api.chucknorris.io/jokes/random");
    const data = await response.json();

    return {
      content: [
        {
          type: "text",
          text: data.value
        }
      ]
    };
  }
);
```

Here's how you can add a tool for example. This specific tool creates a tool call "random-joke" that calls a Chuck Norris API and returns a JSON response.

### Python

```python
@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b
```

Now your server has one tool.

### TypeScript

```typescript
// server-sse.ts
import { Request, Response } from "express";
import express from "express";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";

// Create an MCP server
const server = new McpServer({
  name: "example-server",
  version: "1.0.0",
});

const app = express();

const transports: { [sessionId: string]: SSEServerTransport } = {};

app.get("/sse", async (_: Request, res: Response) => {
  const transport = new SSEServerTransport("/messages", res);
  transports[transport.sessionId] = transport;
  res.on("close", () => {
    delete transports[transport.sessionId];
  });
  await server.connect(transport);
});

app.post("/messages", async (req: Request, res: Response) => {
  const sessionId = req.query.sessionId as string;
  const transport = transports[sessionId];
  if (transport) {
    await transport.handlePostMessage(req, res);
  } else {
    res.status(400).send("No transport found for sessionId");
  }
});

server.tool("random-joke", "A joke returned by the chuck norris api", {}, async () => {
  const response = await fetch("https://api.chucknorris.io/jokes/random");
  const data = await response.json();

  return {
    content: [
      {
        type: "text",
        text: data.value,
      },
    ],
  };
});

app.listen(3001);
```

### Python

```python
from starlette.applications import Starlette
from starlette.routing import Mount, Host
from mcp.server.fastmcp import FastMCP


mcp = FastMCP("My App")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

# Mount the SSE server to the existing ASGI server
app = Starlette(
    routes=[
        Mount('/', app=mcp.sse_app()),
    ]
)
```

### .NET

1. Let's create some tools first, for this we will create a file *Tools.cs* with the following content:

  ```csharp
  using System.ComponentModel;
  using System.Text.Json;
  using ModelContextProtocol.Server;

  namespace server;

  [McpServerToolType]
  public sealed class Tools
  {

      public Tools()
      {
      
      }

      [McpServerTool, Description("Add two numbers together.")]
      public async Task<string> AddNumbers(
          [Description("The first number")] int a,
          [Description("The second number")] int b)
      {
          return (a + b).ToString();
      }

  }
  ```

  Here we've added the following:

  - Created a class `Tools` with the decorator `McpServerToolType`.
  - Defined a tool `AddNumbers` by decorating the method with `McpServerTool`. We've also provided parameters and an implementation.

1. Let's leverage the `Tools` class we just created:

  ```csharp
  var builder = WebApplication.CreateBuilder(args);
  builder.Services
      .AddMcpServer()
      .WithTools<Tools>();


  builder.Services.AddHttpClient();

  var app = builder.Build();

  app.MapMcp();
  ```

  We've added a call to `WithTools` that specifies `Tools` as the class containing the tools. That's it, we're ready.

Great, we have a server using SSE, let's take it for a spin next.

## Exercise: Debugging an SSE Server with Inspector

Inspector is a great tool that we saw in a previous lesson [Creating your first server](/03-GettingStarted/01-first-server/README.md). Let's see if we can use the Inspector even here:

### -1- ==Running the inspector==

To run the inspector, you first must have an SSE server running, so let's do that next:

1. ==Run the server== 

    ### TypeScript

    ```sh
    tsx && node ./build/server-sse.ts
    ```

    ### Python

    ```sh
    uvicorn server:app
    ```

    Note how we use the executable `uvicorn` that's installed when we typed `pip install "mcp[cli]"`. Typing ==`server:app` means we're trying to run a file `server.py` and for it to have a Starlette instance called `app`==. 

    ### .NET

    ```sh
    dotnet run
    ```

    This should start the server. To interface with it you need a new terminal.

2. ==Run the inspector==

    > ![NOTE]
    > Run this ==in a separate terminal window than the server is running in==. Also note, you need to adjust the below command to fit the URL where your server runs.

    ```sh
    npx @modelcontextprotocol/inspector --cli http://localhost:8000/sse --method tools/list
    ```

    Running the inspector looks the same in all runtimes. Note how we instead of passing a path to our server and a command for starting the server we ==instead pass the URL where the server is running and we also specify the `/sse` route.==

### -2- Trying out the tool

Connect the server by selecting SSE in the droplist an==d fill in the url field== where your server is running, for example http:localhost:4321/sse. Now click the "Connect" button. As before, select to list tools, select a tool and provide input values. You should see a result like below:

![[sse-inspector.png]]

Great, you're able to work with the inspector, let's see how we can work with Visual Studio Code next.

## Assignment

Try building out your server with more capabilities. See [this page](https://api.chucknorris.io/) to, for example, add a tool that calls an API. You decide what the server should look like. Have fun :)

## Solution

Here's a possible solution with working code.

[[mcp/mcp for beginners microsoft/Building First Server & Client/05-sse-server/solution]]
## ==Key Takeaways==

The key takeaways from this chapter are the following:

- ==SSE is the second supported transport next to stdio.==
- To support SSE, you need to ==manage incoming connections and messages using a web framework.==
- You can use both Inspector and Visual Studio Code to consume an SSE server, just like stdio servers. Note how it differs a little between stdio and SSE. For ==SSE, you need to start up the server separately and then run your inspector tool==. For the ==inspector tool==, there's also ==some differences in that you need to specify the URL.== 

## Additional Resources

- [SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)

## What's Next

- Next: [HTTP Streaming with MCP (Streamable HTTP)](../06-http-streaming/README.md)