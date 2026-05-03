# Microsoft Foundry Chat Application Exercise Guide

## Exercise Overview

This exercise teaches you how to build a generative AI chat application using the **OpenAI SDK** connected to a **GPT-4.1 model** deployed in **Microsoft Foundry**. The application demonstrates modern API patterns, conversation management, streaming responses, and asynchronous programming.

**Duration:** ~45 minutes  
**Difficulty:** Intermediate  
**Key Technologies:** Python, OpenAI SDK, Azure Identity, Microsoft Foundry

---

## Table of Contents

1. [Prerequisites & Setup](#prerequisites--setup)
2. [Understanding Microsoft Foundry](#understanding-microsoft-foundry)
3. [Project Structure](#project-structure)
4. [Key Technologies Explained](#key-technologies-explained)
5. [Implementation Details](#implementation-details)
6. [API Evolution](#api-evolution)
7. [Exam Preparation Tips](#exam-preparation-tips)

---

## Prerequisites & Setup

### Required Tools
- **Azure Subscription** - Provides cloud resources for Foundry
- **Visual Studio Code** - Development environment
- **Python 3.13.xx** - Runtime (3.14 has dependency compilation issues)
- **Git** - Version control
- **Azure CLI** - Command-line tool for Azure authentication

### Required Python Packages
```
python-dotenv      # Load environment variables from .env file
aiohttp            # Async HTTP client (used in async operations)
azure-identity     # Azure authentication and credentials
openai             # OpenAI SDK for model interaction
```

---

## Understanding Microsoft Foundry

### What is Microsoft Foundry?

Microsoft Foundry (formerly Azure AI Studio) is a **platform for building, evaluating, and deploying AI applications**. It provides:

- **Model Deployment** - Deploy pre-trained models (GPT-4, GPT-3.5, etc.)
- **Unified Endpoints** - Single point of access for all models via Azure OpenAI Service
- **Resource Management** - Organize models, datasets, and configurations
- **Role-Based Access Control** - Manage team permissions and access

### Why Use Foundry?

1. **Enterprise Integration** - Works seamlessly with Azure ecosystem
2. **Authentication** - Uses Azure Entra ID for secure access
3. **Cost Management** - Built-in monitoring and quota management
4. **Model Flexibility** - Easy switching between different model versions
5. **Compliance** - Data residency and security controls

### Key Concepts

| Concept | Definition |
|---------|-----------|
| **Project** | Container for organizing models, data, and resources |
| **Deployment** | Instance of a model running and serving requests |
| **Endpoint** | URL used to access your deployed model |
| **Azure OpenAI Endpoint** | `https://{resource-name}.openai.azure.com/openai/v1` |

---

## Project Structure

```
labfiles/foundry-chat/python/chat-app/
├── .env                  # Configuration file (endpoint, deployment name)
├── requirements.txt      # Python package dependencies
├── chat-app.py          # Synchronous chat application
└── chat-async.py        # Asynchronous chat application
```

### Configuration File (.env)

```env
AZURE_OPENAI_ENDPOINT="https://samabrains-ai-lab-resource.openai.azure.com/openai/v1"
MODEL_DEPLOYMENT="gpt-4.1"
```

**Important:** 
- `AZURE_OPENAI_ENDPOINT` is the **service endpoint** (not the project endpoint)
- `MODEL_DEPLOYMENT` is the exact name assigned when deploying the model

---

## Key Technologies Explained

### 1. OpenAI SDK

The **OpenAI SDK** provides a Python interface to communicate with language models.

```python
from openai import OpenAI
from openai import AsyncOpenAI
```

**Two main classes:**
- `OpenAI()` - Synchronous client (blocking calls)
- `AsyncOpenAI()` - Asynchronous client (non-blocking calls)

### 2. Azure Identity Authentication

Instead of using API keys, modern applications use **token-based authentication** via Azure Entra ID.

```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
```

**How it works:**
1. `DefaultAzureCredential()` - Automatically finds available credentials (user login, managed identity, etc.)
2. `get_bearer_token_provider()` - Generates bearer tokens for authentication
3. Token scope: `https://ai.azure.com/.default` - Authorizes access to Azure AI services

**Advantages:**
- ✅ No API key exposure in code
- ✅ Works with Azure roles and RBAC
- ✅ Automatic token refresh
- ✅ Audit trail of who accessed what

### 3. Environment Variables (.env)

The `python-dotenv` package loads configuration from `.env` files:

```python
from dotenv import load_dotenv
import os

load_dotenv()
endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
deployment = os.getenv("MODEL_DEPLOYMENT")
```

**Best Practice:** Never hardcode sensitive configuration—use `.env` files (and add to `.gitignore`).

---

## Implementation Details

### Part 1: Basic Setup & Client Initialization

#### Step 1: Import Namespaces
```python
from openai import OpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from dotenv import load_dotenv
import os
```

#### Step 2: Load Configuration
```python
load_dotenv()
azure_openai_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
model_deployment = os.getenv("MODEL_DEPLOYMENT")
```

#### Step 3: Create OpenAI Client
```python
token_provider = get_bearer_token_provider(
    DefaultAzureCredential(), "https://ai.azure.com/.default"
)

openai_client = OpenAI(
    base_url=azure_openai_endpoint,
    api_key=token_provider
)
```

**What's happening:**
- `DefaultAzureCredential()` - Gets credentials from current Azure login
- `token_provider` - Creates a function that generates bearer tokens
- `base_url` - Points to your Foundry model endpoint
- `api_key` - Uses token provider instead of static API key

---

### Part 2: API Evolution - From ChatCompletions to Responses API

#### ChatCompletions API (Legacy Pattern)

```python
completion = openai_client.chat.completions.create(
    model=model_deployment,
    messages=[
        {
            "role": "system",
            "content": "You are a helpful AI assistant..."
        },
        {
            "role": "user",
            "content": input_text
        }
    ]
)
print(completion.choices[0].message.content)
```

**Characteristics:**
- Uses `messages` array with roles (system, user, assistant)
- System prompt defines the model's behavior
- Returns complete response at once
- Stateless - no built-in conversation tracking

#### Responses API (Modern Pattern)

```python
response = openai_client.responses.create(
    model=model_deployment,
    instructions="You are a helpful AI assistant...",
    input=input_text
)
print(response.output_text)
```

**Improvements:**
- Simpler syntax - `instructions` instead of system message
- Direct `input` parameter instead of messages array
- Built-in support for response IDs for conversation tracking
- More intuitive for single-turn interactions

**Key Difference:**
| Aspect | ChatCompletions | Responses API |
|--------|-----------------|----------------|
| Syntax | More verbose (messages) | Simpler (instructions) |
| System Prompt | `messages[0]` | `instructions` |
| User Input | `messages[-1]` | `input` |
| Response Tracking | Manual | Built-in (response IDs) |

---

### Part 3: Conversation Tracking

#### The Problem
Without tracking, the model has no memory of previous exchanges:

```
User: "Tell me about ELIZA"
Assistant: [Explains ELIZA]

User: "How does it compare to modern LLMs?"
Assistant: "I don't know what 'it' refers to..." ❌
```

#### The Solution: Response IDs

```python
# Initialize response tracking
last_response_id = None

# First interaction
response = openai_client.responses.create(
    model=model_deployment,
    instructions="You are a helpful AI assistant...",
    input="Tell me about ELIZA",
    previous_response_id=last_response_id  # None on first call
)
print(response.output_text)
last_response_id = response.id  # Save for next interaction

# Second interaction
response = openai_client.responses.create(
    model=model_deployment,
    instructions="You are a helpful AI assistant...",
    input="How does it compare to modern LLMs?",
    previous_response_id=last_response_id  # Pass previous ID
)
print(response.output_text)
last_response_id = response.id
```

**How it works:**
1. Each response has a unique `response.id`
2. Pass `previous_response_id=last_response_id` to link responses
3. Model receives context from the previous response
4. Conversation context is maintained

**Advanced Usage:**
You can also reference any previous response, not just the immediately preceding one:
```python
# Resume conversation from 5 interactions ago
response = openai_client.responses.create(
    model=model_deployment,
    instructions="...",
    input=user_input,
    previous_response_id=stored_response_ids[some_previous_index]
)
```

---

### Part 4: Streaming Responses

#### Why Stream?

Large responses can take several seconds to arrive completely. Without streaming, the app appears frozen:

```python
# Without streaming - app waits silently
response = openai_client.responses.create(...)
print(response.output_text)  # Long pause before anything prints
```

#### Streaming Implementation

```python
# With streaming - response appears incrementally
stream = openai_client.responses.create(
    model=model_deployment,
    instructions="You are a helpful AI assistant...",
    input=input_text,
    previous_response_id=last_response_id,
    stream=True  # Enable streaming
)

for event in stream:
    if event.type == "response.output_text.delta":
        print(event.delta, end="")  # Print each chunk
    elif event.type == "response.completed":
        last_response_id = event.response.id  # Save ID from completed event
print()  # Final newline
```

**Event Types:**
- `response.output_text.delta` - Partial text chunk (stream this)
- `response.completed` - Response finished, contains final `response.id`

**Stream Workflow:**
1. Request with `stream=True`
2. Iterate through events as they arrive
3. Print text chunks immediately (delta events)
4. Capture response ID from completed event
5. Add final newline after stream ends

**Benefits:**
- ✅ Immediate visual feedback to user
- ✅ Appears responsive during long responses
- ✅ Can process chunks while more arrive
- ✅ Better UX for production applications

---

### Part 5: Asynchronous Programming

#### Why Async?

Async allows your program to handle multiple operations without blocking:

```python
# Without async - if one operation takes 10 seconds, entire app waits
response = openai_client.responses.create(...)  # Blocks for 10 seconds
print(response.output_text)

# With async - program can do other things while waiting
response = await async_client.responses.create(...)  # Non-blocking
```

#### Async Implementation

```python
import asyncio
from openai import AsyncOpenAI
from azure.identity.aio import DefaultAzureCredential, get_bearer_token_provider

# Setup async client
credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(
    credential, "https://ai.azure.com/.default"
)

async_client = AsyncOpenAI(
    base_url=azure_openai_endpoint,
    api_key=token_provider
)

# Use async/await in main function
async def main():
    # Track responses
    last_response_id = None
    
    while True:
        input_text = input('\nEnter a prompt (or type "quit" to exit): ')
        if input_text.lower() == "quit":
            break
        
        # Await non-blocking response
        response = await async_client.responses.create(
            model=model_deployment,
            instructions="You are a helpful AI assistant...",
            input=input_text,
            previous_response_id=last_response_id
        )
        
        print("Assistant:", response.output_text)
        last_response_id = response.id

# Cleanup
finally:
    await credential.close()

# Run async main
if __name__ == '__main__':
    asyncio.run(main())
```

**Key Async Concepts:**

| Concept | Meaning |
|---------|---------|
| `async def` | Declares an asynchronous function |
| `await` | Pauses execution until async operation completes |
| `asyncio.run()` | Runs async main function |
| `AsyncOpenAI` | Async version of OpenAI client |
| `.aio` imports | Async versions from azure.identity |

**When to Use Async:**
- ✅ Building web servers handling many requests
- ✅ Long-running operations (API calls, file I/O)
- ✅ Applications needing responsive UI
- ❌ Simple sequential scripts (added complexity)

---

## API Evolution Summary

### Evolution Path

1. **ChatCompletions API** → Basic, well-established
2. **Responses API** → Newer, simpler, with built-in tracking
3. **Streaming** → Real-time, incremental responses
4. **Async** → Non-blocking, responsive applications

### Decision Tree

```
Choose API based on your needs:

Need conversation tracking?
├─ YES → Use Responses API with previous_response_id
└─ NO  → Use ChatCompletions API (simpler for single-turn)

Need real-time response display?
├─ YES → Add stream=True
└─ NO  → Use regular (non-streaming)

Need responsive UI/multiple operations?
├─ YES → Use AsyncOpenAI
└─ NO  → Use OpenAI (simpler)
```

---

## Complete Implementation Reference

### Synchronous Chat App (chat-app.py)

```python
import os
from dotenv import load_dotenv
from openai import OpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

def main():
    os.system('cls' if os.name == 'nt' else 'clear')
    
    try:
        # Load configuration
        load_dotenv()
        azure_openai_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
        model_deployment = os.getenv("MODEL_DEPLOYMENT")
        
        # Initialize OpenAI client with token-based auth
        token_provider = get_bearer_token_provider(
            DefaultAzureCredential(), "https://ai.azure.com/.default"
        )
        openai_client = OpenAI(
            base_url=azure_openai_endpoint,
            api_key=token_provider
        )
        
        # Track conversation
        last_response_id = None
        
        # Conversation loop
        while True:
            input_text = input('\nEnter a prompt (or type "quit" to exit): ')
            if input_text.lower() == "quit":
                break
            if len(input_text) == 0:
                print("Please enter a prompt.")
                continue
            
            # Stream response with context tracking
            stream = openai_client.responses.create(
                model=model_deployment,
                instructions="You are a helpful AI assistant...",
                input=input_text,
                previous_response_id=last_response_id,
                stream=True
            )
            
            for event in stream:
                if event.type == "response.output_text.delta":
                    print(event.delta, end="")
                elif event.type == "response.completed":
                    last_response_id = event.response.id
            print()
    
    except Exception as ex:
        print(ex)

if __name__ == '__main__':
    main()
```

### Asynchronous Chat App (chat-async.py)

```python
import os
import asyncio
from dotenv import load_dotenv
from openai import AsyncOpenAI
from azure.identity.aio import DefaultAzureCredential, get_bearer_token_provider

async def main():
    os.system('cls' if os.name == 'nt' else 'clear')
    
    try:
        # Load configuration
        load_dotenv()
        azure_openai_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
        model_deployment = os.getenv("MODEL_DEPLOYMENT")
        
        # Initialize async OpenAI client
        credential = DefaultAzureCredential()
        token_provider = get_bearer_token_provider(
            credential, "https://ai.azure.com/.default"
        )
        async_client = AsyncOpenAI(
            base_url=azure_openai_endpoint,
            api_key=token_provider
        )
        
        # Track conversation
        last_response_id = None
        
        # Conversation loop
        while True:
            input_text = input('\nEnter a prompt (or type "quit" to exit): ')
            if input_text.lower() == "quit":
                break
            if len(input_text) == 0:
                print("Please enter a prompt.")
                continue
            
            # Await async response with context
            response = await async_client.responses.create(
                model=model_deployment,
                instructions="You are a helpful AI assistant...",
                input=input_text,
                previous_response_id=last_response_id
            )
            
            print("Assistant:", response.output_text)
            last_response_id = response.id
    
    except Exception as ex:
        print(ex)
    
    finally:
        # Close async client
        await credential.close()

if __name__ == '__main__':
    asyncio.run(main())
```

---

## Exam Preparation Tips

### Key Concepts to Master

1. **Microsoft Foundry Architecture**
   - What is it? Platform for building AI applications
   - Key components: Projects, Deployments, Endpoints
   - Why use Foundry? Enterprise integration, security, cost control

2. **Authentication Patterns**
   - Token-based vs API key authentication
   - `DefaultAzureCredential()` - Finds available credentials
   - `get_bearer_token_provider()` - Generates tokens
   - Scope: `https://ai.azure.com/.default`

3. **OpenAI SDK Patterns**
   - `OpenAI()` class for synchronous operations
   - `AsyncOpenAI()` class for async operations
   - Base URL points to Azure OpenAI endpoint

4. **API Design Evolution**
   - ChatCompletions API → Responses API
   - Why switch? Simpler syntax, built-in tracking, streaming support
   - System prompt → Instructions parameter

5. **Conversation Management**
   - Problem: Stateless APIs lose context
   - Solution: Track response IDs
   - How: Pass `previous_response_id` to link responses

6. **Streaming Implementation**
   - Purpose: Real-time response display
   - How: `stream=True` parameter
   - Events: `response.output_text.delta` and `response.completed`
   - Usage: Print delta events as they arrive

7. **Async/Await Patterns**
   - When to use: Multiple long-running operations
   - How: Use `async def`, `await`, `asyncio.run()`
   - Benefits: Non-blocking, responsive applications
   - Caution: Adds complexity; use only when needed

### Practice Questions

1. **What is the purpose of Microsoft Foundry?**
   - A: Platform for deploying and managing AI models with enterprise features

2. **Why use `DefaultAzureCredential()` instead of hardcoded API keys?**
   - A: Better security, no key exposure, automatic token refresh, audit trail

3. **What's the difference between ChatCompletions and Responses APIs?**
   - A: Responses API is newer, simpler syntax, has built-in support for conversation tracking

4. **How do you maintain conversation context across multiple requests?**
   - A: Track response IDs and pass `previous_response_id` to link responses

5. **What does `stream=True` do?**
   - A: Returns streaming events, allowing real-time display of partial responses

6. **When should you use AsyncOpenAI instead of OpenAI?**
   - A: When handling multiple long-running operations without blocking

7. **What file should contain your Azure endpoint and deployment name?**
   - A: `.env` file (never commit to git)

### Common Mistakes to Avoid

❌ **Mistake 1:** Using project endpoint instead of Azure OpenAI endpoint  
✅ **Fix:** Copy endpoint from Foundry home page → `https://{resource}.openai.azure.com/openai/v1`

❌ **Mistake 2:** Hardcoding API keys in code  
✅ **Fix:** Use `.env` file + `python-dotenv` + `.gitignore`

❌ **Mistake 3:** Not tracking response IDs for multi-turn conversations  
✅ **Fix:** Save `last_response_id = response.id` and pass to next request

❌ **Mistake 4:** Using synchronous API for high-concurrency scenarios  
✅ **Fix:** Use AsyncOpenAI for multiple simultaneous requests

❌ **Mistake 5:** Forgetting to close async credentials  
✅ **Fix:** Use `finally` block with `await credential.close()`

### Real-World Applications

This pattern is used in:
- 💬 **Chatbots** - Customer support, virtual assistants
- 📊 **Analytics** - Document analysis, summarization
- 🤖 **Workflow Automation** - Process complex tasks
- 🎮 **Game AI** - NPC dialogue, behavior
- 📝 **Code Generation** - GitHub Copilot-style features

---

## Glossary

| Term | Definition |
|------|-----------|
| **Bearer Token** | Authentication credential passed in HTTP headers |
| **Entra ID** | Microsoft's cloud-based identity platform (formerly Azure AD) |
| **Stream Event** | Individual chunk of data in a streaming response |
| **Async/Await** | Python pattern for non-blocking operations |
| **Response ID** | Unique identifier for each model response |
| **Token Scope** | Permissions granted by a token (e.g., `ai.azure.com`) |
| **RBAC** | Role-Based Access Control - manages who can do what |

---

## Additional Resources

- **Microsoft Learn:** Microsoft Foundry fundamentals
- **OpenAI Documentation:** Python SDK and API reference
- **Azure Identity Docs:** Authentication patterns and best practices
- **Python Async Guide:** Comprehensive async/await tutorial

---

## Summary

This exercise teaches the complete lifecycle of building a modern AI chat application:

1. ✅ Set up authentication with Azure Entra ID
2. ✅ Initialize OpenAI client pointing to Foundry deployment
3. ✅ Implement single-turn and multi-turn conversations
4. ✅ Add streaming for real-time response display
5. ✅ Implement async patterns for responsive applications

**Key Takeaway:** The Responses API with streaming and conversation tracking provides a modern, user-friendly way to build interactive AI applications while maintaining architectural best practices around authentication, configuration management, and scalability.

Good luck on your exam! 🚀
