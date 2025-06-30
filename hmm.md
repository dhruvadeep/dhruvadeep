I'll help you create a proper MCP client that lists all available tools from your server, and also improve your server implementation.

## Updated MCP Client (Tool Lister)

Here's your updated client that will connect and list all available tools:

```python
"""MCP Streamable HTTP Client - Tool Discovery"""

import argparse
import asyncio
from contextlib import AsyncExitStack
from typing import Optional
import json
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

import os
from dotenv import load_dotenv

# Load environment variables
load_dotenv() 

HTTP_PROXY = os.getenv("HTTP_PROXY", "http://webproxy.ext.ti.com:80")
HTTPS_PROXY = os.getenv("HTTPS_PROXY", "http://webproxy.ext.ti.com:80")
NO_PROXY = os.getenv("NO_PROXY", "localhost,127.0.0.1,.ti.com")

os.environ["HTTP_PROXY"] = HTTP_PROXY
os.environ["HTTPS_PROXY"] = HTTPS_PROXY
os.environ["NO_PROXY"] = NO_PROXY

class MCPToolLister:
    """MCP Client for discovering and listing tools from an MCP server"""

    def __init__(self):
        self.session: Optional[ClientSession] = None
        self.exit_stack = AsyncExitStack()

    async def connect_to_streamable_http_server(
        self, server_url: str, headers: Optional[dict] = None
    ):
        """Connect to an MCP server running with HTTP Streamable transport"""
        self._streams_context = streamablehttp_client(
            url=server_url,
            headers=headers or {},
        )
        read_stream, write_stream, _ = await self._streams_context.__aenter__()

        self._session_context = ClientSession(read_stream, write_stream)
        self.session: ClientSession = await self._session_context.__aenter__()

        # Initialize the connection
        await self.session.initialize()
        print("✅ Connected to MCP Streamable HTTP server successfully.")

    async def list_tools(self):
        """List all available tools from the MCP server"""
        if not self.session:
            print("❌ No active session. Connect to server first.")
            return

        try:
            # Get the list of tools
            result = await self.session.list_tools()
            tools = result.tools

            if not tools:
                print("📭 No tools available on this server.")
                return

            print(f"\n🔧 Found {len(tools)} tools on the server:\n")
            print("=" * 80)

            for i, tool in enumerate(tools, 1):
                print(f"\n[{i}] Tool: {tool.name}")
                print(f"    Description: {tool.description or 'No description provided'}")
                
                # Display input schema
                if hasattr(tool, 'inputSchema') and tool.inputSchema:
                    schema = tool.inputSchema
                    print(f"    Input Schema:")
                    print(f"      Type: {schema.get('type', 'Not specified')}")
                    
                    if 'properties' in schema:
                        print(f"      Parameters:")
                        for param_name, param_info in schema['properties'].items():
                            param_type = param_info.get('type', 'unknown')
                            param_desc = param_info.get('description', 'No description')
                            required = param_name in schema.get('required', [])
                            required_text = " (required)" if required else " (optional)"
                            print(f"        - {param_name}: {param_type}{required_text}")
                            if param_desc != 'No description':
                                print(f"          Description: {param_desc}")

                # Display annotations if available
                if hasattr(tool, 'annotations') and tool.annotations:
                    print(f"    Annotations:")
                    annotations = tool.annotations
                    if hasattr(annotations, 'title') and annotations.title:
                        print(f"      Title: {annotations.title}")
                    if hasattr(annotations, 'readOnlyHint'):
                        print(f"      Read Only: {annotations.readOnlyHint}")
                    if hasattr(annotations, 'destructiveHint'):
                        print(f"      Destructive: {annotations.destructiveHint}")
                    if hasattr(annotations, 'idempotentHint'):
                        print(f"      Idempotent: {annotations.idempotentHint}")
                    if hasattr(annotations, 'openWorldHint'):
                        print(f"      Open World: {annotations.openWorldHint}")

                print("-" * 40)

        except Exception as e:
            print(f"❌ Failed to list tools: {e}")

    async def list_resources(self):
        """List all available resources from the MCP server"""
        if not self.session:
            print("❌ No active session. Connect to server first.")
            return

        try:
            # Get the list of resources
            result = await self.session.list_resources()
            resources = result.resources

            if not resources:
                print("📭 No resources available on this server.")
                return

            print(f"\n📚 Found {len(resources)} resources on the server:\n")
            print("=" * 80)

            for i, resource in enumerate(resources, 1):
                print(f"\n[{i}] Resource: {resource.uri}")
                print(f"    Name: {resource.name or 'No name provided'}")
                print(f"    Description: {resource.description or 'No description provided'}")
                print(f"    MIME Type: {resource.mimeType or 'Not specified'}")
                print("-" * 40)

        except Exception as e:
            print(f"❌ Failed to list resources: {e}")

    async def get_server_info(self):
        """Get server information"""
        if not self.session:
            print("❌ No active session. Connect to server first.")
            return

        try:
            # Server info is available after initialization
            print(f"\n🖥️  Server Information:")
            print(f"    Name: {self.session.server_name}")
            print(f"    Version: {self.session.server_version}")
            print(f"    Protocol Version: {self.session.protocol_version}")
            
            # Check capabilities
            capabilities = self.session.server_capabilities
            print(f"    Capabilities:")
            if hasattr(capabilities, 'tools') and capabilities.tools:
                print(f"      ✅ Tools supported")
            if hasattr(capabilities, 'resources') and capabilities.resources:
                print(f"      ✅ Resources supported")
            if hasattr(capabilities, 'logging') and capabilities.logging:
                print(f"      ✅ Logging supported")
            if hasattr(capabilities, 'prompts') and capabilities.prompts:
                print(f"      ✅ Prompts supported")

        except Exception as e:
            print(f"❌ Failed to get server info: {e}")

    async def cleanup(self):
        """Properly clean up the session and streams"""
        try:
            if hasattr(self, '_session_context') and self._session_context:
                await self._session_context.__aexit__(None, None, None)
            if hasattr(self, '_streams_context') and self._streams_context:
                await self._streams_context.__aexit__(None, None, None)
            print("🧹 Client cleanup completed.")
        except Exception as e:
            print(f"❌ Error during cleanup: {e}")


async def main():
    """Main function to run the MCP tool discovery client"""
    parser = argparse.ArgumentParser(description="Discover and list MCP tools and resources")
    parser.add_argument(
        "--mcp-localhost-port", 
        type=int, 
        default=8002, 
        help="Localhost port to connect to"
    )
    parser.add_argument(
        "--show-resources", 
        action="store_true", 
        help="Also show available resources"
    )
    parser.add_argument(
        "--show-server-info", 
        action="store_true", 
        help="Show server information"
    )
    
    args = parser.parse_args()

    client = MCPToolLister()
    try:
        # Connect to the server
        server_url = f"http://localhost:{args.mcp_localhost_port}/mcp"
        print(f"🔗 Connecting to MCP server at: {server_url}")
        
        await client.connect_to_streamable_http_server(server_url)
        
        # Show server info if requested
        if args.show_server_info:
            await client.get_server_info()
        
        # List all tools
        await client.list_tools()
        
        # List resources if requested
        if args.show_resources:
            await client.list_resources()
            
    except Exception as e:
        print(f"❌ Failed to connect or list tools: {e}")
        print(f"   Make sure your MCP server is running on port {args.mcp_localhost_port}")
    finally:
        await client.cleanup()


if __name__ == "__main__":
    asyncio.run(main())
```

## Updated MCP Server (Arithmetic Server)

Here's your improved server with better documentation and annotations:

```python
"""
Enhanced Arithmetic MCP Server
=============================

A Model Context Protocol (MCP) server providing basic arithmetic operations
and help resources. This server demonstrates proper tool and resource
implementation with comprehensive annotations.

Features:
- Basic arithmetic operations (add, subtract, multiply, divide)
- Help resource for usage documentation
- Proper error handling and validation
- Tool annotations for better UX integration

Usage:
    python arithmetic_server.py

The server will start on port 8002 using streamable HTTP transport.
"""

from mcp.server.fastmcp import FastMCP
import os
from dotenv import load_dotenv
import logging

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# Load environment variables
load_dotenv() 

HTTP_PROXY = os.getenv("HTTP_PROXY", "http://webproxy.ext.ti.com:80")
HTTPS_PROXY = os.getenv("HTTPS_PROXY", "http://webproxy.ext.ti.com:80")
NO_PROXY = os.getenv("NO_PROXY", "localhost,127.0.0.1,.ti.com")

os.environ["HTTP_PROXY"] = HTTP_PROXY
os.environ["HTTPS_PROXY"] = HTTPS_PROXY
os.environ["NO_PROXY"] = NO_PROXY

# Create an MCP server with enhanced configuration
mcp = FastMCP(
    name="Enhanced Arithmetic Server",
    version="1.0.0",
    port=8002,
    host="127.0.0.1"  # Bind to localhost only for security
)

@mcp.tool(
    annotations={
        "title": "Add Numbers",
        "readOnlyHint": True,
        "destructiveHint": False,
        "idempotentHint": True,
        "openWorldHint": False
    }
)
def add(a: float, b: float) -> float:
    """
    Add two numbers together.
    
    This tool performs basic addition of two floating-point numbers.
    It's a read-only operation that doesn't modify any external state.
    
    Args:
        a: The first number to add
        b: The second number to add
        
    Returns:
        The sum of a and b
        
    Example:
        add(5.0, 3.0) returns 8.0
    """
    result = a + b
    logger.info(f"Addition: {a} + {b} = {result}")
    return result

@mcp.tool(
    annotations={
        "title": "Subtract Numbers",
        "readOnlyHint": True,
        "destructiveHint": False,
        "idempotentHint": True,
        "openWorldHint": False
    }
)
def subtract(a: float, b: float) -> float:
    """
    Subtract the second number from the first number.
    
    This tool performs basic subtraction of two floating-point numbers.
    It's a read-only operation that doesn't modify any external state.
    
    Args:
        a: The number to subtract from (minuend)
        b: The number to subtract (subtrahend)
        
    Returns:
        The difference of a and b (a - b)
        
    Example:
        subtract(10.0, 4.0) returns 6.0
    """
    result = a - b
    logger.info(f"Subtraction: {a} - {b} = {result}")
    return result

@mcp.tool(
    annotations={
        "title": "Multiply Numbers",
        "readOnlyHint": True,
        "destructiveHint": False,
        "idempotentHint": True,
        "openWorldHint": False
    }
)
def multiply(a: float, b: float) -> float:
    """
    Multiply two numbers together.
    
    This tool performs basic multiplication of two floating-point numbers.
    It's a read-only operation that doesn't modify any external state.
    
    Args:
        a: The first number to multiply (multiplicand)
        b: The second number to multiply (multiplier)
        
    Returns:
        The product of a and b
        
    Example:
        multiply(2.0, 7.0) returns 14.0
    """
    result = a * b
    logger.info(f"Multiplication: {a} × {b} = {result}")
    return result

@mcp.tool(
    annotations={
        "title": "Divide Numbers",
        "readOnlyHint": True,
        "destructiveHint": False,
        "idempotentHint": True,
        "openWorldHint": False
    }
)
def divide(a: float, b: float) -> float:
    """
    Divide the first number by the second number.
    
    This tool performs basic division of two floating-point numbers.
    It includes proper error handling for division by zero.
    It's a read-only operation that doesn't modify any external state.
    
    Args:
        a: The number to be divided (dividend)
        b: The number to divide by (divisor)
        
    Returns:
        The quotient of a and b (a / b)
        
    Raises:
        ValueError: If b is zero (division by zero)
        
    Example:
        divide(20.0, 5.0) returns 4.0
        divide(10.0, 0.0) raises ValueError
    """
    if b == 0:
        error_msg = f"Cannot divide by zero: {a} ÷ {b}"
        logger.error(error_msg)
        raise ValueError(error_msg)
    
    result = a / b
    logger.info(f"Division: {a} ÷ {b} = {result}")
    return result

@mcp.tool(
    annotations={
        "title": "Power Operation",
        "readOnlyHint": True,
        "destructiveHint": False,
        "idempotentHint": True,
        "openWorldHint": False
    }
)
def power(base: float, exponent: float) -> float:
    """
    Raise a number to the power of another number.
    
    This tool calculates base raised to the power of exponent.
    It's a read-only operation that doesn't modify any external state.
    
    Args:
        base: The base number
        exponent: The exponent (power to raise the base to)
        
    Returns:
        The result of base^exponent
        
    Example:
        power(2.0, 3.0) returns 8.0
        power(9.0, 0.5) returns 3.0 (square root)
    """
    result = base ** exponent
    logger.info(f"Power: {base}^{exponent} = {result}")
    return result

@mcp.tool(
    annotations={
        "title": "Square Root",
        "readOnlyHint": True,
        "destructiveHint": False,
        "idempotentHint": True,
        "openWorldHint": False
    }
)
def square_root(number: float) -> float:
    """
    Calculate the square root of a number.
    
    This tool calculates the square root of a positive number.
    It includes proper error handling for negative numbers.
    
    Args:
        number: The number to find the square root of
        
    Returns:
        The square root of the number
        
    Raises:
        ValueError: If the number is negative
        
    Example:
        square_root(16.0) returns 4.0
        square_root(2.0) returns ~1.414
    """
    if number < 0:
        error_msg = f"Cannot calculate square root of negative number: {number}"
        logger.error(error_msg)
        raise ValueError(error_msg)
    
    result = number ** 0.5
    logger.info(f"Square root: √{number} = {result}")
    return result

@mcp.resource("help://arithmetic")
def arithmetic_help() -> str:
    """
    Get comprehensive help on using the Enhanced Arithmetic Server.
    
    This resource provides detailed documentation about all available
    arithmetic operations, their parameters, and usage examples.
    """
    return """
Enhanced Arithmetic Server Help
===============================

This server provides comprehensive arithmetic operations through the Model Context Protocol (MCP).

🔧 Available Tools:
==================

1. add(a, b) - Add two numbers
   • Parameters: a (float), b (float)
   • Returns: Sum of a and b
   • Example: add(5.0, 3.0) → 8.0

2. subtract(a, b) - Subtract b from a
   • Parameters: a (float), b (float)  
   • Returns: Difference (a - b)
   • Example: subtract(10.0, 4.0) → 6.0

3. multiply(a, b) - Multiply two numbers
   • Parameters: a (float), b (float)
   • Returns: Product of a and b
   • Example: multiply(2.0, 7.0) → 14.0

4. divide(a, b) - Divide a by b
   • Parameters: a (float), b (float)
   • Returns: Quotient (a / b)
   • Error: Raises ValueError if b = 0
   • Example: divide(20.0, 5.0) → 4.0

5. power(base, exponent) - Calculate base^exponent
   • Parameters: base (float), exponent (float)
   • Returns: base raised to the power of exponent
   • Example: power(2.0, 3.0) → 8.0

6. square_root(number) - Calculate square root
   • Parameters: number (float)
   • Returns: Square root of number
   • Error: Raises ValueError if number < 0
   • Example: square_root(16.0) → 4.0

🚀 Usage Notes:
===============

• All operations accept floating-point numbers
• Division by zero is properly handled with error messages
• Square root of negative numbers raises appropriate errors
• All tools are read-only and don't modify server state
• Operations are idempotent (same inputs = same outputs)
• Comprehensive logging is provided for debugging

📚 Resources:
=============

• help://arithmetic - This help documentation

🔒 Security Features:
====================

• Server binds to localhost only (127.0.0.1)
• Input validation for all operations
• Proper error handling and logging
• No external system interactions

For technical support or feature requests, please refer to the server documentation.
    """

@mcp.resource("api://arithmetic/status")
def server_status() -> str:
    """Get current server status and statistics."""
    return """
Server Status: ✅ ACTIVE
======================

Service: Enhanced Arithmetic Server
Version: 1.0.0
Transport: Streamable HTTP
Port: 8002
Host: 127.0.0.1 (localhost only)

Available Endpoints:
• Tools: 6 arithmetic operations
• Resources: 2 help and status resources

Server Capabilities:
• ✅ Tools supported
• ✅ Resources supported  
• ✅ Error handling
• ✅ Input validation
• ✅ Comprehensive logging

Last Started: Server runtime information
Memory Usage: Optimized for lightweight operations
Security: Localhost-only binding, input validation
    """

if __name__ == "__main__":
    print("🚀 Starting Enhanced Arithmetic MCP Server...")
    print(f"   Server: Enhanced Arithmetic Server v1.0.0")
    print(f"   Transport: Streamable HTTP")
    print(f"   Host: 127.0.0.1 (localhost only)")
    print(f"   Port: 8002")
    print(f"   Endpoint: http://127.0.0.1:8002/mcp")
    print("   Status: Ready to accept connections")
    print("\n📋 Available Tools: add, subtract, multiply, divide, power, square_root")
    print("📚 Available Resources: help://arithmetic, api://arithmetic/status")
    print("\n🔧 Use the MCP client to discover and interact with tools.")
    
    try:
        mcp.run("streamable-http")
    except KeyboardInterrupt:
        print("\n🛑 Server shutdown requested")
    except Exception as e:
        logger.error(f"Server error: {e}")
        print(f"❌ Server error: {e}")
```

## How to Use

1. **Start the server:**
   ```bash
   python arithmetic_server.py
   ```

2. **Run the client to list tools:**
   ```bash
   # Basic tool listing
   python mcp_tool_lister.py

   # With server info and resources
   python mcp_tool_lister.py --show-server-info --show-resources

   # Custom port
   python mcp_tool_lister.py --mcp-localhost-port 8002
   ```

## Expected Output

When you run the client, you should see something like:

```
🔗 Connecting to MCP server at: http://localhost:8002/mcp
✅ Connected to MCP Streamable HTTP server successfully.

🔧 Found 6 tools on the server:

================================================================================

[1] Tool: add
    Description: Add two numbers together.
    Input Schema:
      Type: object
      Parameters:
        - a: number (required)
        - b: number (required)
    Annotations:
      Title: Add Numbers
      Read Only: True
      Destructive: False
      Idempotent: True
      Open World: False
----------------------------------------

[2] Tool: subtract
    Description: Subtract the second number from the first number.
    Input Schema:
      Type: object
      Parameters:
        - a: number (required)
        - b: number (required)
    ...

🧹 Client cleanup completed.
```

This setup gives you:
- ✅ Clean tool discovery and listing
- ✅ Detailed parameter information
- ✅ Tool annotations display
- ✅ Resource listing capability
- ✅ Server information display
- ✅ Proper error handling and cleanup
- ✅ Enhanced server with more tools and better documentation
