---
title: Creating a Weather Agent with BedrockLLMAgent and Custom Tools
description: Understanding Tool use in Bedrock LLM Agent
---

This guide demonstrates how to create a specialized weather agent using BedrockLLMAgent and a custom weather tool. We'll walk through the process of defining the tool, setting up the agent, and integrating it into your Multi-Agent Orchestrator system.

<br>


1. **Define the Weather Tool**

Let's break down the weather tool definition into its key components:

A. Tool Description

```typescript
export const weatherToolDescription = [
  {
    toolSpec: {
      name: "Weather_Tool",
      description: "Get the current weather for a given location, based on its WGS84 coordinates.",
      inputSchema: {
        json: {
          type: "object",
          properties: {
            latitude: {
              type: "string",
              description: "Geographical WGS84 latitude of the location.",
            },
            longitude: {
              type: "string",
              description: "Geographical WGS84 longitude of the location.",
            },
          },
          required: ["latitude", "longitude"],
        }
      },
    }
  }
];
```

**Explanation:**
- This describes the tool's interface to the LLM.
- `name`: Identifies the tool to the LLM.
- `description`: Explains the tool's purpose to the LLM.
- `inputSchema`: Defines the expected input format.
  - Requires `latitude` and `longitude` as strings.
  - This schema helps the LLM understand how to use the tool correctly.

<br>
<br>

B. Custom Prompt

```typescript
export const WEATHER_PROMPT = `
You are a weather assistant that provides current weather data for user-specified locations using only
the Weather_Tool, which expects latitude and longitude. Infer the coordinates from the location yourself.
If the user provides coordinates, infer the approximate location and refer to it in your response.
To use the tool, you strictly apply the provided tool specification.

- Explain your step-by-step process, and give brief updates before each step.
- Only use the Weather_Tool for data. Never guess or make up information.
- Repeat the tool use for subsequent requests if necessary.
- If the tool errors, apologize, explain weather is unavailable, and suggest other options.
- Report temperatures in °C (°F) and wind in km/h (mph). Keep weather reports concise. Sparingly use
  emojis where appropriate.
- Only respond to weather queries. Remind off-topic users of your purpose.
- Never claim to search online, access external data, or use tools besides Weather_Tool.
- Complete the entire process until you have all required data before sending the complete response.
`;
```

**Explanation:**
- This prompt sets the behavior and limitations for the LLM.
- It instructs the LLM to:
  - Use only the Weather_Tool for data.
  - Infer coordinates from location names.
  - Provide step-by-step explanations.
  - Handle errors gracefully.
  - Format responses consistently (units, conciseness).
  - Stay on topic and use only the provided tool.

<br>
<br>

C. Tool Handler

```typescript

import { ConversationMessage, ParticipantRole } from "multi-agent-orchestrator";


export async function weatherToolHandler(response, conversation: ConversationMessage[]):Promse<ConversationMessage> {
  const responseContentBlocks = response.content as any[];
  let toolResults: any = [];

  if (!responseContentBlocks) {
    throw new Error("No content blocks in response");
  }

  for (const contentBlock of response.content) {
    if ("toolUse" in contentBlock) {
      const toolUseBlock = contentBlock.toolUse;
      if (toolUseBlock.name === "Weather_Tool") {
        const response = await fetchWeatherData({
          latitude: toolUseBlock.input.latitude,
          longitude: toolUseBlock.input.longitude
        });
        toolResults.push({
          "toolResult": {
            "toolUseId": toolUseBlock.toolUseId,
            "content": [{ json: { result: response } }],
          }
        });
      }
    }
  }

  const message: ConversationMessage = { role: ParticipantRole.USER, content: toolResults };
  return message;
}
```

**Explanation:**
- This handler processes the LLM's request to use the Weather_Tool.
- It iterates through the response content, looking for tool use blocks.
- When it finds a Weather_Tool use:
  - It calls `fetchWeatherData` with the provided coordinates.
  - It formats the result into a tool result object.
- Finally, it adds the tool results to the conversation as a new user message.

<br>
<br>

D. Data Fetching Function

```typescript
async function fetchWeatherData(inputData: { latitude: number; longitude: number }) {
  const endpoint = "https://api.open-meteo.com/v1/forecast";
  const params = new URLSearchParams({
    latitude: inputData.latitude.toString(),
    longitude: inputData.longitude.toString(),
    current_weather: "true",
  });

  try {
    const response = await fetch(`${endpoint}?${params}`);
    const data = await response.json();
    if (!response.ok) {
      return { error: 'Request failed', message: data.message || 'An error occurred' };
    }
    return { weather_data: data };
  } catch (error: any) {
    return { error: error.name, message: error.message };
  }
}
```

**Explanation:**
- This function makes the actual API call to get weather data.
- It uses the Open-Meteo API (a free weather API service).
- It constructs the API URL with the provided latitude and longitude.
- It handles both successful responses and errors:
  - On success, it returns the weather data.
  - On failure, it returns an error object.

These components work together to create a functional weather tool:
1. The tool description tells the LLM how to use the tool.
2. The prompt guides the LLM's behavior and response format.
3. The handler processes the LLM's tool use requests.
4. The fetch function retrieves real weather data based on the LLM's input.

This setup allows the BedrockLLMAgent to provide weather information by seamlessly integrating external data into its responses.
<br>
<br>

2. **Create the Weather Agent**

Now that we have our weather tool defined and the code above in a file called `weatherTool.ts`, let's create a BedrockLLMAgent that uses this tool.

```typescript
// weatherAgent.ts

import { BedrockLLMAgent } from 'multi-agent-orchestrator';
import { weatherToolDescription, weatherToolHandler, WEATHER_PROMPT } from './weatherTool';

const weatherAgent = new BedrockLLMAgent({
    name: "Weather Agent",
    description:`Specialized agent for providing comprehensive weather information and forecasts for specific cities worldwide.
    This agent can deliver current conditions, temperature ranges, precipitation probabilities, wind speeds, humidity levels, UV indexes, and extended forecasts.
    It can also offer insights on severe weather alerts, air quality indexes, and seasonal climate patterns.
    The agent is capable of interpreting user queries related to weather, including natural language requests like 'Do I need an umbrella today?' or 'What's the best day for outdoor activities this week?'.
    It can handle location-specific queries and time-based weather predictions, making it ideal for travel planning, event scheduling, and daily decision-making based on weather conditions.`,
    streaming: false,
    inferenceConfig: {
        temperature: 0.1,
    },
    toolConfig: {
        useToolHandler: weatherToolHandler,
        tool: weatherToolDescription,
        toolMaxRecursions: 5
    }
});

weatherAgent.setSystemPrompt(WEATHER_PROMPT);

```

3. **Add the Weather Agent to the Orchestrator**

Now we can add our weather agent to the Multi-Agent Orchestrator:

```typescript

import { MultiAgentOrchestrator } from "multi-agent-orchestrator";

const orchestrator = new MultiAgentOrchestrator();

orchestrator.addAgent(weatherAgent);

```

## 4. Using the Weather Agent

Now that our weather agent is set up and added to the orchestrator, we can use it to get weather information:

```typescript

const response = await orchestrator.routeRequest(
  "What's the weather like in New York City?",
  "user123",
  "session456"
);
```

### How It Works

1. When a weather query is received, the orchestrator routes it to the Weather Agent.
2. The Weather Agent processes the query using the custom system prompt (WEATHER_PROMPT).
3. The agent uses the Weather_Tool to fetch weather data for the specified location.
4. The weatherToolHandler processes the tool use, fetches real weather data, and adds it to the conversation.
5. The agent then formulates a response based on the weather data and the original query.

This setup allows for a specialized weather agent that can handle various weather-related queries while using real-time data from an external API.

---

By following this guide, you can create a powerful, context-aware weather agent using BedrockLLMAgent and custom tools within your Multi-Agent Orchestrator system.