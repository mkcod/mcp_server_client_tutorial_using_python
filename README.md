[![MseeP.ai Security Assessment Badge](https://mseep.net/pr/mkcod-mcp-server-client-tutorial-using-python-badge.png)](https://mseep.ai/app/mkcod-mcp-server-client-tutorial-using-python)

# Step-by-Step Guide to Build Your Own MCP Server and MCP Client with UI Using Python

This comprehensive guide walks you through the process of creating both a Model Context Protocol (MCP) server and a client with a graphical user interface using Python. MCP is designed to facilitate communication between language models and tool providers, enabling powerful AI-based applications. By the end of this tutorial, you'll have a functional MCP system with both server and client components that can interact seamlessly.

## Prerequisites and Environment Setup

Before diving into development, ensure your system meets the following requirements:

- A Windows or Mac computer with the latest Python version installed
- Familiarity with Python programming
- Basic understanding of asynchronous programming in Python
- Knowledge of UI development concepts

First, let's set up our development environment with the necessary tools and packages:

```bash
# Install uv tool if not already installed
pip install uv
```

The `uv` tool will help us create and manage our Python projects efficiently. It's a modern Python package manager and virtual environment tool that we'll use throughout this guide[^1][^2].

## Building the MCP Server

### Step 1: Creating the Server Project

Navigate to your desired project directory and create a new MCP server project:

```bash
cd "your_desired_directory"
uvx create-mcp-server
```

During the creation process, you'll be prompted for project details such as name, description, and configuration options. This will generate a project structure with all necessary files and dependencies[^1].

### Step 2: Setting Up the Virtual Environment

After the project is created, navigate to the project directory and set up the virtual environment:

```bash
cd server-project-name
uv sync --dev --all-extras
```

This command ensures all dependencies are properly installed in your virtual environment[^1].

### Step 3: Understanding the Project Structure

The generated server project has a structure similar to this:

```
SERVER-PROJECT-NAME/
├── .venv/
├── Lib/
│ └── site-packages/
├── Scripts/
├── src/
│ └── server_project_name/
│ ├── __pycache__/
│ ├── __init__.py
│ └── server.py
├── .gitignore
├── pyproject.toml
├── README.md
└── uv.lock
```

The key files you need to focus on are in the `src/server_project_name/` directory, particularly `server.py` which contains the server implementation[^1].

### Step 4: Implementing Server Functionality

Open the `server.py` file and implement your server functionality. A basic MCP server should define tools that clients can call. Here's a simple example implementing an addition tool:

```python
from mcp import Server, ToolCall, ToolResult

class MyMCPServer(Server):
    async def handle_tool_call(self, tool_call: ToolCall) -> ToolResult:
        if tool_call.name == "add":
            # Extract arguments
            a = tool_call.arguments.get("a")
            b = tool_call.arguments.get("b")
            
            # Perform addition
            result = a + b
            
            # Return the result
            return ToolResult(content=str(result))
        
        # Handle unknown tool
        return ToolResult(error=f"Unknown tool: {tool_call.name}")
    
    async def list_tools(self):
        # Define available tools
        tools = [
            {
                "name": "add",
                "description": "Adds two numbers together",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "a": {"type": "number"},
                        "b": {"type": "number"}
                    },
                    "required": ["a", "b"]
                }
            }
        ]
        return tools

# Run the server
if __name__ == "__main__":
    server = MyMCPServer()
    server.run()
```

This example creates a server with a simple addition tool that clients can call[^3].

### Step 5: Running the Server

To run your MCP server:

```bash
python -m server_project_name.server
```

By default, the server will run on localhost at port 8000. You can customize this in the server configuration if needed[^3].

## Building the MCP Client

### Step 1: Creating the Client Project

Create a new directory for your MCP client:

```bash
uv init mcp-client
cd mcp-client
```

Set up a virtual environment and install the required packages:

```bash
uv venv
# Activate the virtual environment
# On Windows:
.venv\Scripts\activate
# On macOS/Linux:
source .venv/bin/activate

# Install required packages
uv add mcp anthropic python-dotenv
```

Create a new file for your client implementation:

```bash
touch client.py
```


### Step 2: Implementing the Client

Create a basic MCP client that can connect to the server and call its tools. Here's a sample implementation:

```python
import asyncio
from typing import Optional
from contextlib import AsyncExitStack
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from dotenv import load_dotenv

load_dotenv()  # Load environment variables

class MCPClient:
    def __init__(self):
        # Initialize session and client objects
        self.session: Optional[ClientSession] = None
        self.exit_stack = AsyncExitStack()

    async def connect_to_server(self, server_script_path: str):
        """Connect to an MCP server"""
        is_python = server_script_path.endswith('.py')
        is_js = server_script_path.endswith('.js')
        
        if not (is_python or is_js):
            raise ValueError("Server script must be a .py or .js file")
            
        command = "python" if is_python else "node"
        
        server_params = StdioServerParameters(
            command=command,
            args=[server_script_path],
            env=None
        )
        
        stdio_transport = await self.exit_stack.enter_async_context(stdio_client(server_params))
        self.stdio, self.write = stdio_transport
        
        self.session = await self.exit_stack.enter_async_context(ClientSession(self.stdio, self.write))
        await self.session.initialize()
        
        # List available tools
        response = await self.session.list_tools()
        tools = response.tools
        print("\nConnected to server with tools:", [tool.name for tool in tools])
        return tools

    async def call_add_tool(self, a: int, b: int):
        """Call the add tool on the server"""
        if not self.session:
            raise RuntimeError("Not connected to server")
            
        result = await self.session.call_tool(
            name="add",
            arguments={"a": a, "b": b}
        )
        return result.content[^0].text

    async def cleanup(self):
        """Clean up resources"""
        await self.exit_stack.aclose()

async def main():
    client = MCPClient()
    try:
        await client.connect_to_server("path/to/your/server.py")
        result = await client.call_add_tool(4, 5)
        print(f"Result: {result}")
    finally:
        await client.cleanup()

if __name__ == "__main__":
    asyncio.run(main())
```

This client connects to the MCP server, calls the "add" tool with arguments 4 and 5, and prints the result[^2][^3].

## Adding a UI to the MCP Client Using Tkinter

### Step 1: Setting Up the Basic UI

Now let's enhance our client by adding a graphical user interface using Tkinter. First, install Tkinter if it's not already included in your Python installation:

```bash
uv add tk
```

Now, let's create a new file called `client_ui.py`:

```python
import asyncio
import tkinter as tk
from tkinter import ttk, messagebox
from client import MCPClient  # Import the client class we created earlier

class MCPClientUI:
    def __init__(self, root):
        self.root = root
        self.root.title("MCP Client")
        self.root.geometry("500x400")
        
        self.client = MCPClient()
        self.connected = False
        
        self.setup_ui()
        
    def setup_ui(self):
        # Create main frame
        main_frame = ttk.Frame(self.root, padding="10")
        main_frame.pack(fill=tk.BOTH, expand=True)
        
        # Server connection section
        server_frame = ttk.LabelFrame(main_frame, text="Server Connection", padding="10")
        server_frame.pack(fill=tk.X, pady=5)
        
        ttk.Label(server_frame, text="Server Script Path:").grid(row=0, column=0, sticky=tk.W, pady=5)
        self.server_path = tk.StringVar(value="path/to/your/server.py")
        ttk.Entry(server_frame, textvariable=self.server_path, width=40).grid(row=0, column=1, padx=5, pady=5)
        
        self.connect_btn = ttk.Button(server_frame, text="Connect", command=self.connect_to_server)
        self.connect_btn.grid(row=0, column=2, padx=5, pady=5)
        
        # Tool section
        tool_frame = ttk.LabelFrame(main_frame, text="Add Tool", padding="10")
        tool_frame.pack(fill=tk.X, pady=5)
        
        ttk.Label(tool_frame, text="Number A:").grid(row=0, column=0, sticky=tk.W, pady=5)
        self.num_a = tk.IntVar(value=0)
        ttk.Entry(tool_frame, textvariable=self.num_a, width=10).grid(row=0, column=1, padx=5, pady=5)
        
        ttk.Label(tool_frame, text="Number B:").grid(row=0, column=2, sticky=tk.W, pady=5)
        self.num_b = tk.IntVar(value=0)
        ttk.Entry(tool_frame, textvariable=self.num_b, width=10).grid(row=0, column=3, padx=5, pady=5)
        
        self.calculate_btn = ttk.Button(tool_frame, text="Calculate", command=self.call_add_tool, state=tk.DISABLED)
        self.calculate_btn.grid(row=0, column=4, padx=5, pady=5)
        
        # Results section
        result_frame = ttk.LabelFrame(main_frame, text="Results", padding="10")
        result_frame.pack(fill=tk.BOTH, expand=True, pady=5)
        
        self.result_text = tk.Text(result_frame, height=10, width=50)
        self.result_text.pack(fill=tk.BOTH, expand=True)
        
    def connect_to_server(self):
        server_path = self.server_path.get()
        self.result_text.insert(tk.END, f"Connecting to server at {server_path}...\n")
        
        # Create and run a coroutine to connect to the server
        async def connect():
            try:
                tools = await self.client.connect_to_server(server_path)
                self.connected = True
                self.root.after(0, self.update_ui_after_connect, tools)
            except Exception as e:
                self.root.after(0, lambda: self.show_error(f"Connection error: {str(e)}"))
        
        asyncio.run(connect())
    
    def update_ui_after_connect(self, tools):
        self.result_text.insert(tk.END, f"Connected to server. Available tools: {[tool.name for tool in tools]}\n")
        self.connect_btn.config(text="Connected", state=tk.DISABLED)
        self.calculate_btn.config(state=tk.NORMAL)
    
    def call_add_tool(self):
        if not self.connected:
            self.show_error("Not connected to server")
            return
            
        try:
            a = self.num_a.get()
            b = self.num_b.get()
            
            self.result_text.insert(tk.END, f"Calculating {a} + {b}...\n")
            
            # Create and run a coroutine to call the add tool
            async def add():
                try:
                    result = await self.client.call_add_tool(a, b)
                    self.root.after(0, lambda: self.show_result(result))
                except Exception as e:
                    self.root.after(0, lambda: self.show_error(f"Tool call error: {str(e)}"))
            
            asyncio.run(add())
        except Exception as e:
            self.show_error(f"Input error: {str(e)}")
    
    def show_result(self, result):
        self.result_text.insert(tk.END, f"Result: {result}\n")
    
    def show_error(self, message):
        messagebox.showerror("Error", message)
        self.result_text.insert(tk.END, f"Error: {message}\n")
    
    def cleanup(self):
        asyncio.run(self.client.cleanup())
        self.root.destroy()

if __name__ == "__main__":
    root = tk.Tk()
    app = MCPClientUI(root)
    root.protocol("WM_DELETE_WINDOW", app.cleanup)
    root.mainloop()
```

This UI provides fields to enter the server path and values for the addition operation, buttons to connect to the server and calculate the result, and a text area to display the outputs[^4].

### Step 2: Handling Asynchronous Operations in Tkinter

Tkinter is not inherently designed to work with asyncio, which our MCP client relies on. The solution above uses a simple approach of running asyncio operations in blocking mode, which works for simple operations but isn't ideal for more complex applications.

For a more robust solution, let's create an enhanced version that better handles asynchronous operations:

```python
import asyncio
import tkinter as tk
from tkinter import ttk, messagebox
from client import MCPClient
import threading

class AsyncTkApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Advanced MCP Client")
        self.root.geometry("550x450")
        
        self.client = MCPClient()
        self.connected = False
        self.loop = asyncio.new_event_loop()
        
        # Start a new thread for running the async event loop
        self.thread = threading.Thread(target=self._start_loop, daemon=True)
        self.thread.start()
        
        self.setup_ui()
    
    def _start_loop(self):
        asyncio.set_event_loop(self.loop)
        self.loop.run_forever()
    
    def setup_ui(self):
        # Create main frame with padding
        main_frame = ttk.Frame(self.root, padding="15")
        main_frame.pack(fill=tk.BOTH, expand=True)
        
        # Server connection section
        server_frame = ttk.LabelFrame(main_frame, text="Server Connection", padding="10")
        server_frame.pack(fill=tk.X, pady=8)
        
        ttk.Label(server_frame, text="Server Script Path:").grid(row=0, column=0, sticky=tk.W, pady=5)
        self.server_path = tk.StringVar(value="path/to/your/server.py")
        ttk.Entry(server_frame, textvariable=self.server_path, width=40).grid(row=0, column=1, padx=5, pady=5)
        
        self.connect_btn = ttk.Button(server_frame, text="Connect", command=self.connect_to_server)
        self.connect_btn.grid(row=0, column=2, padx=5, pady=5)
        
        # Tool section with improved layout
        tool_frame = ttk.LabelFrame(main_frame, text="Add Tool", padding="10")
        tool_frame.pack(fill=tk.X, pady=8)
        
        ttk.Label(tool_frame, text="Number A:").grid(row=0, column=0, sticky=tk.W, pady=5)
        self.num_a = tk.IntVar(value=0)
        ttk.Entry(tool_frame, textvariable=self.num_a, width=10).grid(row=0, column=1, padx=5, pady=5)
        
        ttk.Label(tool_frame, text="Number B:").grid(row=0, column=2, sticky=tk.W, pady=5)
        self.num_b = tk.IntVar(value=0)
        ttk.Entry(tool_frame, textvariable=self.num_b, width=10).grid(row=0, column=3, padx=5, pady=5)
        
        self.calculate_btn = ttk.Button(tool_frame, text="Calculate", command=self.call_add_tool, state=tk.DISABLED)
        self.calculate_btn.grid(row=0, column=4, padx=5, pady=5)
        
        # Results section with scrollbar
        result_frame = ttk.LabelFrame(main_frame, text="Results", padding="10")
        result_frame.pack(fill=tk.BOTH, expand=True, pady=8)
        
        # Add a scrollbar to the text widget
        scrollbar = ttk.Scrollbar(result_frame)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        self.result_text = tk.Text(result_frame, height=12, width=50, yscrollcommand=scrollbar.set)
        self.result_text.pack(fill=tk.BOTH, expand=True)
        scrollbar.config(command=self.result_text.yview)
        
        # Status bar
        status_frame = ttk.Frame(main_frame)
        status_frame.pack(fill=tk.X, pady=5)
        self.status_var = tk.StringVar(value="Ready")
        status_label = ttk.Label(status_frame, textvariable=self.status_var, anchor=tk.W)
        status_label.pack(side=tk.LEFT)
    
    def connect_to_server(self):
        server_path = self.server_path.get()
        self.log_message(f"Connecting to server at {server_path}...")
        self.status_var.set("Connecting...")
        
        async def connect_async():
            try:
                tools = await self.client.connect_to_server(server_path)
                self.root.after(0, lambda: self.update_ui_after_connect(tools))
            except Exception as e:
                self.root.after(0, lambda: self.show_error(f"Connection error: {str(e)}"))
        
        # Schedule the coroutine to run in the asyncio event loop
        asyncio.run_coroutine_threadsafe(connect_async(), self.loop)
    
    def update_ui_after_connect(self, tools):
        self.connected = True
        self.log_message(f"Connected to server. Available tools: {[tool.name for tool in tools]}")
        self.connect_btn.config(text="Connected", state=tk.DISABLED)
        self.calculate_btn.config(state=tk.NORMAL)
        self.status_var.set("Connected")
    
    def call_add_tool(self):
        if not self.connected:
            self.show_error("Not connected to server")
            return
            
        try:
            a = self.num_a.get()
            b = self.num_b.get()
            
            self.log_message(f"Calculating {a} + {b}...")
            self.status_var.set("Calculating...")
            
            async def add_async():
                try:
                    result = await self.client.call_add_tool(a, b)
                    self.root.after(0, lambda: self.show_result(result))
                except Exception as e:
                    self.root.after(0, lambda: self.show_error(f"Tool call error: {str(e)}"))
            
            # Schedule the coroutine to run in the asyncio event loop
            asyncio.run_coroutine_threadsafe(add_async(), self.loop)
        except Exception as e:
            self.show_error(f"Input error: {str(e)}")
    
    def log_message(self, message):
        self.result_text.insert(tk.END, f"{message}\n")
        self.result_text.see(tk.END)  # Scroll to the end
    
    def show_result(self, result):
        self.log_message(f"Result: {result}")
        self.status_var.set("Ready")
    
    def show_error(self, message):
        messagebox.showerror("Error", message)
        self.log_message(f"Error: {message}")
        self.status_var.set("Error occurred")
    
    def cleanup(self):
        async def cleanup_async():
            await self.client.cleanup()
            self.loop.stop()
        
        # Schedule cleanup in the event loop
        asyncio.run_coroutine_threadsafe(cleanup_async(), self.loop)
        self.root.after(100, self.root.destroy)  # Give time for cleanup to complete

if __name__ == "__main__":
    root = tk.Tk()
    app = AsyncTkApp(root)
    root.protocol("WM_DELETE_WINDOW", app.cleanup)
    root.mainloop()
```

This enhanced version uses a separate thread for the asyncio event loop, which allows for better handling of asynchronous operations without blocking the UI[^4].

## Testing the Complete System

To test your MCP server and client:

1. Start the MCP server in one terminal:

```bash
cd server-project-directory
python -m server_project_name.server
```

2. Run the MCP client UI in another terminal:

```bash
cd mcp-client
python client_ui.py
```

3. In the client UI:
    - Enter the path to your server script
    - Click "Connect" to establish a connection
    - Enter two numbers in the input fields
    - Click "Calculate" to send the request to the server
    - View the result in the results text area

## Extending the System

Once you have the basic server and client working, you can extend the system by:

1. Adding more tools to the server, such as text processing or data analysis functions
2. Enhancing the client UI with more features and better error handling
3. Implementing authentication and security measures
4. Adding support for multiple concurrent connections
5. Creating a more advanced UI with graphs, charts, or other visualizations

## Conclusion

In this step-by-step guide, we've built a complete MCP server and client system with a graphical user interface. The Model Context Protocol allows for powerful communication between clients and servers, enabling a wide range of applications that leverage AI and other tools.

The system we've built demonstrates the fundamental concepts of MCP: defining tools, calling them from clients, and presenting results in a user-friendly interface. This knowledge provides a solid foundation for developing more complex MCP-based applications for various use cases.

As MCP continues to evolve, staying updated with the latest developments in the protocol will help you build even more powerful and efficient applications. The core concepts learned here will remain applicable even as the technology advances.
