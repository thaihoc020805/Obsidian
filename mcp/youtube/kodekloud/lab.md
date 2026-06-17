complete client

```python
#!/usr/bin/env python3
"""
Complete MCP Client - Full featured client with all MCP capabilities
"""
import asyncio
from mcp.client.session import ClientSession
from mcp.client.streamable_http import streamablehttp_client
from mcp.types import (
    ListRootsResult, Root,
    CreateMessageRequestParams, CreateMessageResult, TextContent,
    ElicitRequestParams, ElicitResult
)

# === CALLBACK IMPLEMENTATIONS ===

async def list_roots_callback(context):
    """Provide project roots to server"""
    project_roots = [
        "file:///home/lab-user/",
        "file:///home/lab-user/flight-booking-server/",
        "file:///home/lab-user/mcp-client/"
    ]
    
    print(f"📁 Providing {len(project_roots)} project roots to server")
    return ListRootsResult(roots=[Root(uri=root) for root in project_roots])

async def handle_sampling(context, params: CreateMessageRequestParams) -> CreateMessageResult:
    """Handle LLM sampling requests from server"""
    print("🤖 Server requests LLM generation!")
    
    messages = params.messages
    user_messages = [msg for msg in messages if msg.role == "user"]
    
    if user_messages:
        prompt = user_messages[-1].content.text if hasattr(user_messages[-1].content, 'text') else str(user_messages[-1].content)
        print(f"📝 Prompt: {prompt[:100]}...")
        
        # Context-aware response generation
        if "poem" in prompt.lower():
            response = "Flights soar through azure skies,\nConnecting distant hearts and minds,\nTravel dreams come true."
        elif "explain" in prompt.lower():
            response = "Flight booking involves searching for available flights, comparing prices and schedules, selecting preferred options, and completing the reservation process with payment."
        elif "recommend" in prompt.lower():
            response = "I recommend booking flights 2-3 months in advance for domestic travel, being flexible with dates, and considering nearby airports for better deals."
        else:
            response = f"Here's a thoughtful response about travel and flight booking: {prompt}"
        
        return CreateMessageResult(
            role="assistant",
            content=TextContent(type="text", text=response),
            model="travel-assistant-llm",
            stopReason="endTurn"
        )
    
    return CreateMessageResult(
        role="assistant",
        content=TextContent(type="text", text="I couldn't generate a response."),
        model="travel-assistant-llm",
        stopReason="endTurn"
    )

async def handle_elicitation(context, params: ElicitRequestParams) -> ElicitResult:
    """Handle user input requests from server"""
    print(f"🔔 Server requests user input: {params.message}")
    
    message_lower = params.message.lower()
    
    # Intelligent response based on request type
    if "name" in message_lower:
        response = {"name": "Sarah Wilson", "title": "Dr."}
    elif "email" in message_lower:
        response = {"email": "sarah.wilson@example.com", "notifications": True}
    elif "preference" in message_lower:
        response = {"seat": "aisle", "class": "economy", "meal": "vegetarian"}
    elif "budget" in message_lower:
        response = {"budget": 750, "currency": "USD", "flexible": True}
    elif "date" in message_lower:
        response = {"departure": "2024-12-30", "return": "2025-01-05", "flexible": True}
    else:
        response = {"response": "confirmed", "user_id": "user_789"}
    
    print(f"👤 User response: {response}")
    return ElicitResult(action="accept", content=response)

# === MAIN CLIENT FUNCTIONALITY ===

async def test_complete_client():
    """Test all MCP client capabilities in one comprehensive session"""
    print("🌟 COMPLETE MCP CLIENT")
    print("=" * 50)
    print("🎯 Goal: Demonstrate full MCP client capabilities")
    print("=" * 50)
    
    try:
        async with streamablehttp_client("http://localhost:8000/mcp/") as (read, write, _):
            # Initialize with ALL callbacks
            async with ClientSession(
                read, write,
                list_roots_callback=list_roots_callback,
                sampling_callback=handle_sampling,
                elicitation_callback=handle_elicitation
            ) as client:
                
                await client.initialize()
                print("✅ Connected with full MCP capabilities!")
                print()
                
                # === 1. DISCOVERY PHASE ===
                print("🔍 PHASE 1: Discovery")
                print("-" * 30)
                
                # List tools
                tools = await client.list_tools()
                print(f"🔧 Found {len(tools.tools)} tools:")
                for tool in tools.tools:
                    print(f"  • {tool.name}: {tool.description}")
                print()
                
                # List resources
                resources = await client.list_resources()
                print(f"📊 Found {len(resources.resources)} resources:")
                for resource in resources.resources:
                    print(f"  • {resource.uri}")
                print()
                
                # List prompts
                prompts = await client.list_prompts()
                print(f"💬 Found {len(prompts.prompts)} prompts:")
                for prompt in prompts.prompts:
                    print(f"  • {prompt.name}: {prompt.description}")
                print()
                
                # === 2. TOOLS TESTING PHASE ===
                print("🛠️ PHASE 2: Tools Testing")
                print("-" * 30)
                
                # Test flight search
                print("✈️ Testing flight search...")
                try:
                    search_result = await client.call_tool("search_flights", {
                        "origin": "SFO",
                        "destination": "NYC"
                    })
                    if search_result.content:
                        print(f"   ✅ Result: {search_result.content[0].text}")
                    print()
                except Exception as e:
                    print(f"   ❌ Error: {e}")
                    print()
                
                # Test booking creation
                print("🎫 Testing booking creation...")
                try:
                    booking_result = await client.call_tool("create_booking", {
                        "flight_id": "UA789",
                        "passenger_name": "Sarah Wilson"
                    })
                    if booking_result.content:
                        print(f"   ✅ Result: {booking_result.content[0].text}")
                    print()
                except Exception as e:
                    print(f"   ❌ Error: {e}")
                    print()
                
                # === 3. RESOURCES TESTING PHASE ===
                print("📊 PHASE 3: Resources Testing")
                print("-" * 30)
                
                # Test airport resource
                print("🏢 Testing airport resource...")
                try:
                    airport_result = await client.read_resource("file://airports")
                    if airport_result.contents:
                        content = airport_result.contents[0].text
                        print(f"   ✅ Retrieved {len(content)} characters of airport data")
                    print()
                except Exception as e:
                    print(f"   ❌ Error: {e}")
                    print()
                
                # === 4. PROMPTS TESTING PHASE ===
                print("💡 PHASE 4: Prompts Testing")
                print("-" * 30)
                
                # Test flight recommendation prompt
                print("🎯 Testing flight recommendation prompt...")
                try:
                    prompt_result = await client.get_prompt("find_best_flight", {
                        "budget": "1000.0",  # Changed from 1000.0 to "1000.0"
                        "preferences": "first class, morning departure"
                    })
                    if prompt_result.messages:
                        print("   ✅ Prompt generated successfully")
                        print(f"   📝 Message count: {len(prompt_result.messages)}")
                    print()
                except Exception as e:
                    print(f"   ❌ Error: {e}")
                    print()
                
                # Test disruption handling prompt
                print("🚨 Testing disruption handling prompt...")
                try:
                    disruption_result = await client.get_prompt("handle_disruption", {
                        "original_flight": "UA789",
                        "reason": "mechanical issue"
                    })
                    if disruption_result.messages:
                        print("   ✅ Disruption prompt generated successfully")
                    print()
                except Exception as e:
                    print(f"   ❌ Error: {e}")
                    print()
                
                # === 5. SUMMARY PHASE ===
                print("🎊 PHASE 5: Test Summary")
                print("-" * 30)
                print("✅ Client Capabilities Demonstrated:")
                print("   🔌 Basic connection and initialization")
                print("   🔧 Tool discovery and execution")
                print("   📊 Resource access and reading")
                print("   💬 Prompt generation and parameterization")
                print("   📁 Project roots provision")
                print("   🤖 LLM sampling response handling")
                print("   🔔 User input elicitation handling")
                print()
                print("🌟 Complete MCP client implementation successful!")
                print("✨ Ready for production use and integration!")
                
    except Exception as e:
        print(f"❌ Complete client test failed: {e}")
        print("💡 Make sure the flight booking server is running on port 8000")

if __name__ == "__main__":
    asyncio.run(test_complete_client()) 
```

server

```python

from mcp.server.fastmcp import FastMCP


# Create an MCP server

mcp = FastMCP("Flight Booking Server")

  

@mcp.resource("file://airports")

def get_airports():

    """Get list of available airports"""

    return {

        "LAX": {"name": "Los Angeles International", "city": "Los Angeles"},

        "JFK": {"name": "John F. Kennedy International", "city": "New York"},

        "LHR": {"name": "London Heathrow", "city": "London"}

    }

  

@mcp.resource("file://airlines")

def get_airlines():

    """Get list of available airlines and their information"""

    return {

        "AA": {"name": "American Airlines", "country": "USA", "fleet_size": 950},

        "BA": {"name": "British Airways", "country": "UK", "fleet_size": 280},

        "DL": {"name": "Delta Air Lines", "country": "USA", "fleet_size": 860},

        "UA": {"name": "United Airlines", "country": "USA", "fleet_size": 790}

    }

  

@mcp.tool()

def search_flights(origin: str, destination: str) -> dict:

    """Search for flights between two airports"""

    return {

        "flights": [

            {"id": "FL123", "origin": origin, "destination": destination, "price": 299},

            {"id": "FL456", "origin": origin, "destination": destination, "price": 399}

        ]

    }

  

@mcp.tool()

def create_booking(flight_id: str, passenger_name: str) -> dict:

    """Create a flight booking"""

    return {

        "booking_id": f"BK{flight_id[-3:]}",

        "flight_id": flight_id,

        "passenger": passenger_name,

        "status": "confirmed"

    }

  

@mcp.prompt()

def find_best_flight(budget: float, preferences: str = "economy") -> str:

    """Generate a prompt for finding the best flight within budget"""

    return f"""Please help me find the best flight within a ${budget} budget.

My preferences: {preferences}

  

Please consider:

- Price (must be under ${budget})

- Flight duration  

- Airline reputation

- Departure times

  

Use the search_flights tool to find available options and provide a recommendation with reasoning."""

  

@mcp.prompt()

def handle_disruption(original_flight: str, reason: str) -> str:

    """Generate a prompt for handling flight disruptions"""

    return f"""A passenger's flight {original_flight} has been disrupted due to: {reason}

  

Please help resolve this by:

1. Understanding the passenger's situation

2. Finding alternative flight options using search_flights

3. Providing clear rebooking steps

4. Offering appropriate compensation if applicable

  

Be empathetic and solution-focused in your response."""

  

if __name__ == "__main__":

    # Run in streamable HTTP mode for client connections

    mcp.run(transport="streamable-http")
```

