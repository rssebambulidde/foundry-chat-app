# Foundry AI Tools Exercise Guide

## Exercise Overview

This exercise teaches you how to build a **generative AI application that uses tools** to extend the model's capabilities. You'll create a travel assistant that combines:
- **web_search tool** - Find current information from the internet
- **file_search tool** - Query knowledge from uploaded PDF documents
- **Responses API** - Modern API with built-in tool support

**Duration:** ~30 minutes  
**Difficulty:** Intermediate  
**Key Learning:** Tool-augmented AI applications with grounded knowledge

---

## Table of Contents

1. [Understanding AI Tools](#understanding-ai-tools)
2. [Tools vs. Traditional APIs](#tools-vs-traditional-apis)
3. [Key Concepts](#key-concepts)
4. [Vector Stores and File Search](#vector-stores-and-file-search)
5. [Implementation Details](#implementation-details)
6. [Tool Integration Patterns](#tool-integration-patterns)
7. [Real-World Applications](#real-world-applications)
8. [Exam Preparation](#exam-preparation)

---

## Understanding AI Tools

### What Are AI Tools?

Tools are functions that a language model can **decide to call** to accomplish tasks. Instead of relying only on training data, the model can:
1. Recognize when it needs external information
2. Call the appropriate tool
3. Receive the result
4. Incorporate it into its response

### Why Tools Matter

**Without tools:**
- Model can only use training data (potentially outdated)
- Can't access real-time information
- Can't query specific documents or databases
- Limited to what the model already "knows"

**With tools:**
- ✅ Access current information (web_search)
- ✅ Query specific documents (file_search)
- ✅ Perform calculations or lookups
- ✅ Integrate external APIs
- ✅ Provide grounded, accurate responses

---

## Tools vs. Traditional APIs

### Traditional API (ChatCompletions)

```python
# Model generates text based on prompt only
response = openai_client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "What's happening in SF next month?"}]
)
# Response: Generic information from training data (possibly outdated)
```

**Limitations:**
- ❌ Can't access real-time events
- ❌ Can't search the web
- ❌ Limited to training data cutoff

### Tools-Augmented API (Responses with Tools)

```python
# Model can call tools as needed
response = openai_client.responses.create(
    model="gpt-4",
    instructions="You are a travel assistant...",
    input="What's happening in SF next month?",
    tools=[
        {"type": "web_search"},
        {"type": "file_search", "vector_store_ids": [store_id]}
    ]
)
# Response: Uses web_search tool to get current events
```

**Advantages:**
- ✅ Model uses tools autonomously
- ✅ Gets real-time, accurate information
- ✅ Responses grounded in facts
- ✅ Combines multiple knowledge sources

---

## Key Concepts

### 1. Vector Store

**Purpose:** A database of embeddings that enables semantic search

**How it works:**
1. PDFs are uploaded
2. Content is split into chunks
3. Each chunk is converted to an embedding (vector)
4. Vectors are stored for similarity search

**Why vectors?**
- Allows semantic search (meaning-based, not keyword-based)
- Model can find relevant content even with different words
- Example: "hotels in SF" matches "accommodation in San Francisco"

### 2. file_search Tool

**Purpose:** Search documents in a vector store

```python
{
    # Tell the model we're using file_search tool
    "type": "file_search",
    # List the vector stores to search (can have multiple)
    "vector_store_ids": [vector_store.id]  # Pass the store ID
}
```

**How it works (step by step):**
1. User asks: "What hotels in SF?"
2. Model converts question to a vector (embedding)
3. Vector store finds similar document chunks
4. Results are passed back to the model
5. Model reads the results and incorporates into response

### 3. web_search Tool

**Purpose:** Search the internet for current information

```python
{
    # Tell the model we're using web_search tool
    "type": "web_search"
    # No configuration needed - searches the whole internet
}
```

**How it works (step by step):**
1. Model recognizes: "This question needs current info"
2. Model calls web_search tool automatically
3. Tool searches the internet and gets results
4. Results passed back to model
5. Model reads results and incorporates into response

**Use cases:**
- Current events and dates
- Weather and conditions
- News and breaking information
- Real-time prices and availability

---

## Vector Stores and File Search

### Creating a Vector Store

```python
# Create an empty vector store (database for embeddings)
# This is like creating an empty filing cabinet
vector_store = openai_client.vector_stores.create(
    name="travel-brochures"  # Give it a descriptive name
)
# After this, we have an empty vector store waiting for files
```

### Uploading Files

```python
# Get list of all PDF files from brochures folder
# glob.glob() searches for files matching a pattern
# "brochures/*.pdf" means "all PDF files in brochures folder"
file_streams = [open(f, "rb") for f in glob.glob("brochures/*.pdf")]

# Upload files and automatically create embeddings
# "upload_and_poll" means: upload and wait until all files are processed
file_batch = openai_client.vector_stores.file_batches.upload_and_poll(
    vector_store_id=vector_store.id,  # Which vector store to put files in
    files=file_streams  # List of files to upload
)

# Print how many files were successfully processed
print(f"Uploaded {file_batch.file_counts.completed} files")
```

**Key parameters (what they mean):**
- `upload_and_poll()` - Upload files AND wait for processing to finish
- `file_streams` - List of open files (PDFs in this case)
- `file_batch.file_counts.completed` - How many files finished processing

### Vector Store Lifecycle

```
1. Create → Empty store created
2. Upload → Files processed, embeddings created
3. Query → file_search tool searches embeddings
4. Retrieve → Relevant chunks returned to model
```

---

## Implementation Details

### Complete Tools App Structure

```python
# Import required libraries
import os  # For system operations
from dotenv import load_dotenv  # For loading .env configuration
import glob  # For finding files by pattern
from openai import OpenAI  # OpenAI SDK
from azure.identity import DefaultAzureCredential, get_bearer_token_provider  # Azure authentication

def main():
    # === STEP 1: LOAD CONFIGURATION ===
    load_dotenv()  # Read .env file
    # Get the server address where our model lives
    azure_openai_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
    # Get the model name (e.g., "gpt-4.1")
    model_deployment = os.getenv("MODEL_DEPLOYMENT")
    
    # === STEP 2: INITIALIZE OPENAI CLIENT ===
    # Create token provider using Azure login
    token_provider = get_bearer_token_provider(
        DefaultAzureCredential(), "https://ai.azure.com/.default"
    )
    # Create connection to the model
    openai_client = OpenAI(
        base_url=azure_openai_endpoint,  # Server address
        api_key=token_provider  # Authentication tokens
    )
    
    # === STEP 3: CREATE VECTOR STORE AND UPLOAD FILES ===
    # Create an empty vector store (database for documents)
    vector_store = openai_client.vector_stores.create(
        name="travel-brochures"  # Name for this document collection
    )
    # Find all PDF files in the brochures folder
    file_streams = [open(f, "rb") for f in glob.glob("brochures/*.pdf")]
    # Upload files and create embeddings automatically
    file_batch = openai_client.vector_stores.file_batches.upload_and_poll(
        vector_store_id=vector_store.id,  # Which store to upload to
        files=file_streams  # Files to upload
    )
    # Close all open file handles (clean up)
    for f in file_streams:
        f.close()
    
    # === STEP 4: CHAT LOOP WITH TOOLS ===
    # Variable to track previous response (for conversation context)
    last_response_id = None
    
    while True:
        # Get input from user
        input_text = input('\nEnter a question: ')
        # Check if user wants to exit
        if input_text.lower() == "quit":
            break  # Exit the loop
        
        # === STEP 5: CALL API WITH TOOLS ===
        # Request response with both file_search and web_search tools
        response = openai_client.responses.create(
            model=model_deployment,  # Use GPT-4.1
            instructions="You are a travel assistant...",  # AI's personality/role
            input=input_text,  # User's question
            previous_response_id=last_response_id,  # Link for conversation context
            tools=[
                # Tool 1: Search uploaded documents (our brochures)
                {
                    "type": "file_search",  # This tool searches files
                    "vector_store_ids": [vector_store.id]  # Search in our brochures
                },
                # Tool 2: Search the internet
                {
                    "type": "web_search"  # This tool searches the web
                }
            ]
        )
        # Print the AI's response
        print(response.output_text)
        # Save this response's ID for next turn (maintains conversation context)
        last_response_id = response.id
```

### Configuration (.env)

**What is .env?** 
A configuration file that stores settings like a computer's contact list. It tells your program:
- Where to find the AI model (server address)
- Which version of the model to use

```env
# Server address where the AI model lives
# This tells your program how to connect to Microsoft Foundry
# Example: https://samabrains-ai-lab-resource.openai.azure.com/openai/v1
AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/openai/v1"

# The name of the model deployment to use
# Think of this as \"which AI version\" (like GPT-4.1 vs GPT-3.5)
MODEL_DEPLOYMENT="gpt-4.1"
```

**Why use .env instead of writing settings in the code?**
- 🔒 **Security:** Don't expose server addresses in code that gets shared
- 🔄 **Flexibility:** Easy to change settings without editing code
- 📝 **Organization:** Keeps configuration separate from code
- 🚀 **Deployment:** Different settings for testing vs production

---

## Tool Integration Patterns

### Pattern 1: Single Tool (file_search)

```python
# Give the model only ONE tool: search documents
tools=[
    {
        "type": "file_search",  # This tool searches files/documents
        "vector_store_ids": [vector_store.id]  # Which vector store to search
    }
]
```

**Use case:** Document-only Q&A (company knowledge base, internal documents only)
**Example:** Customer asking "What's in your employee handbook?"

### Pattern 2: Single Tool (web_search)

```python
# Give the model only ONE tool: search the internet
tools=[
    {
        "type": "web_search"  # This tool searches the internet
    }
]
```

**Use case:** Current events, news, real-time information only
**Example:** User asking "What's happening in SF next month?"

### Pattern 3: Multiple Tools (Recommended)

```python
tools=[
    {
        "type": "file_search",
        "vector_store_ids": [vector_store.id]
    },
    {
        "type": "web_search"
    }
]
```

**Use case:** Hybrid - combine internal knowledge with current info

**Advantages:**
- Model chooses best tool for each query
- file_search for company-specific info
- web_search for current information
- More accurate and comprehensive responses

### Tool Selection Logic

The model automatically decides:
- "What SF hotels?" → file_search (specific company data)
- "What events next month?" → web_search (current info)
- "Margie's hotels vs events?" → Both tools (hybrid)

---

## Real-World Applications

### 1. Customer Support

```python
tools=[
    {
        "type": "file_search",
        "vector_store_ids": [kb_store, policies_store, faq_store]
    },
    {
        "type": "web_search"  # For external references
    }
]
```

Queries:
- "What's your return policy?" → file_search (KB)
- "How do I track my order?" → file_search (FAQ)
- "Is your competitor cheaper?" → web_search (current prices)

### 2. Medical/Legal Assistant

```python
tools=[
    {
        "type": "file_search",
        "vector_store_ids": [medical_journals, case_law]
    }
]
```

Ensures responses grounded in authoritative documents

### 3. Travel Planning

```python
tools=[
    {
        "type": "file_search",
        "vector_store_ids": [brochures, policies, packages]
    },
    {
        "type": "web_search"  # Weather, events, prices
    }
]
```

Combines company offerings with real-time information

### 4. Research Assistant

```python
tools=[
    {
        "type": "file_search",
        "vector_store_ids": [papers_store, data_store]
    },
    {
        "type": "web_search"  # Latest publications
    }
]
```

---

## Comparison: With vs Without Tools

### Scenario: "What's happening in San Francisco next month?"

**Without Tools:**
```
Response: "San Francisco hosts many events including concerts, festivals, 
and sports games. Popular activities include visiting the Golden Gate Bridge 
and Alcatraz..."
```
❌ Generic, possibly outdated, no specific dates

**With web_search Tool:**
```
Response: "In June 2026, San Francisco has:
- San Francisco Pride (late June, final weekend)
- Union Street Festival (June 6-7)
- Stern Grove Festival (starts June 14)
- SF Giants home games (June 8-10, 12-14, 23-25, 26-28)
- San Francisco Design Week (June 2-12)
..."
```
✅ Specific, current, dates and events

### Scenario: "What hotels does Margie's Travel offer in San Francisco?"

**Without Tools:**
```
Response: "Major hotels in San Francisco include the Fairmont, Hilton, 
Marriott, Four Seasons..."
```
❌ Generic hotels, not company-specific

**With file_search Tool:**
```
Response: "Margie's Travel offers:
- The Lombard Hotel (family-run, near Golden Gate Bridge, free parking)
- The Wharf Hotel (in Fisherman's Wharf, continental breakfast included)"
```
✅ Company-specific offerings, accurate information

---

## Exam Preparation Tips

### Key Concepts to Master

1. **Tool Purpose**
   - Tools extend model capabilities beyond training data
   - Enable real-time, accurate, grounded responses

2. **file_search vs web_search**
   - **file_search**: Query uploaded documents (vector store)
   - **web_search**: Query internet (real-time info)

3. **Vector Store Workflow**
   - Create → Upload → Search → Retrieve → Include

4. **Tool Definition**
   ```python
   {
       "type": "file_search",  # or "web_search"
       "vector_store_ids": [...]  # Required for file_search
   }
   ```

5. **Tool Autonomy**
   - Model decides which tool to use
   - No explicit tool calling by developer
   - Model incorporates tool results automatically

6. **Configuration**
   - `.env` stores endpoint and deployment
   - Vector store ID must be passed to file_search
   - Web_search requires no configuration

### Practice Questions

1. **Q: Why would you use file_search instead of just web_search?**
   - A: For company-specific, proprietary, or internal knowledge that isn't on the web

2. **Q: What happens if you upload PDFs but don't reference the vector store in tools?**
   - A: The vector store exists but isn't used; the model won't search those documents

3. **Q: Can a single response use both file_search and web_search?**
   - A: Yes! The model autonomously chooses which tool(s) best answer the query

4. **Q: What's the relationship between embeddings and file_search?**
   - A: Embeddings convert text to vectors; file_search uses semantic similarity of embeddings to find relevant content

5. **Q: How does the model know to use a tool?**
   - A: The tools are specified in the API request; model recognizes it needs that data type and calls the tool

### Common Mistakes to Avoid

❌ **Mistake 1:** Uploading files but forgetting to create vector store
✅ **Fix:** Create vector_store first, then upload_and_poll files to it

❌ **Mistake 2:** Not including vector_store_ids in file_search tool
✅ **Fix:** Pass vector_store_ids as required parameter

❌ **Mistake 3:** Expecting model to use tools without specifying them
✅ **Fix:** Include tools parameter in responses.create()

❌ **Mistake 4:** Using file_search for real-time information
✅ **Fix:** Use web_search for current events, use file_search for static documents

❌ **Mistake 5:** Closing file streams before upload_and_poll completes
✅ **Fix:** Close files AFTER upload_and_poll returns

### Testing Your Understanding

**Test 1:** Modify the app to add a third vector store for competitor pricing. What changes needed?
```python
# Add another vector store to tools
"vector_store_ids": [travel_brochures, competitor_data, pricing_data]
```

**Test 2:** How would you prevent the model from using web_search (only company knowledge)?
```python
# Remove web_search, keep only file_search
tools=[
    {
        "type": "file_search",
        "vector_store_ids": [vector_store.id]
    }
]
```

**Test 3:** What if you want to use multiple file_search stores?
```python
# Pass multiple vector store IDs
{
    "type": "file_search",
    "vector_store_ids": [store1.id, store2.id, store3.id]
}
```

---

## Architecture Diagram

```
User Query
    ↓
Responses API
    ↓
Model decides: Do I need tools?
    ├─→ YES: Call appropriate tool(s)
    │   ├─→ file_search → Vector Store → Semantic Search → Results
    │   └─→ web_search → Internet Search → Current Info → Results
    │
    └─→ NO: Use training data
    
    ↓
Model incorporates results
    ↓
Final Response to User
```

---

## Summary

Tools transform language models from **closed systems** (training data only) to **open systems** (connected to real-time data, documents, and APIs).

**Key Takeaway:** The Responses API with tools enables building intelligent, grounded applications that combine:
- **Company knowledge** (file_search on brochures)
- **Current information** (web_search)
- **Smart tool selection** (model decides autonomously)

This results in accurate, relevant, up-to-date responses that users can trust.

---

## Additional Resources

- **Vector Embeddings:** Understand how text becomes searchable vectors
- **RAG Pattern:** Retrieval-Augmented Generation for enhanced responses
- **Tool Calling:** Advanced pattern for structured tool use
- **Prompt Engineering:** Crafting instructions that guide tool usage

Good luck with your exam! 🚀
