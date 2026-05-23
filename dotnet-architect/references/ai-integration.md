# AI Integration in .NET with Semantic Kernel and OpenAI

## When to Use Semantic Kernel vs. Direct OpenAI SDK

**Use Semantic Kernel when:**
- Building multi-step AI workflows (planning, function calling, chaining)
- You need model portability (swap OpenAI for Azure OpenAI or another provider)
- You're building agents or RAG pipelines
- The AI capability is a core part of the product

**Use the OpenAI SDK directly when:**
- You need a single completion or embedding call
- Semantic Kernel's abstraction adds friction without benefit
- You're doing a quick integration and SK feels like overkill

## Semantic Kernel Setup

```csharp
// Program.cs
builder.Services.AddSingleton<Kernel>(sp =>
{
    var kernelBuilder = Kernel.CreateBuilder();
    kernelBuilder.AddAzureOpenAIChatCompletion(
        deploymentName: config["AzureOpenAI:DeploymentName"],
        endpoint: config["AzureOpenAI:Endpoint"],
        apiKey: config["AzureOpenAI:ApiKey"]);
    return kernelBuilder.Build();
});
```

## RAG Pattern (Retrieval-Augmented Generation)

The standard RAG pipeline in .NET:

1. **Ingestion** — chunk documents, generate embeddings, store in a vector DB
2. **Retrieval** — embed the user query, find top-K similar chunks
3. **Augmentation** — inject retrieved chunks into the prompt as context
4. **Generation** — send augmented prompt to the LLM

```csharp
public async Task<string> AskAsync(string userQuery, CancellationToken ct)
{
    // 1. Embed the query
    var queryEmbedding = await _embeddingService.GenerateEmbeddingAsync(userQuery, ct);

    // 2. Retrieve relevant chunks from vector store
    var relevantChunks = await _vectorStore.SearchAsync(queryEmbedding, topK: 5, ct);

    // 3. Build the augmented prompt
    var context = string.Join("\n\n", relevantChunks.Select(c => c.Content));
    var prompt = $"""
        Answer the question using only the context below.
        Context: {context}
        Question: {userQuery}
        """;

    // 4. Generate response
    var result = await _kernel.InvokePromptAsync(prompt, cancellationToken: ct);
    return result.GetValue<string>();
}
```

**Vector DB options:**
- Azure AI Search (easiest if you're already on Azure)
- pgvector (PostgreSQL extension — great if you want to stay in one DB)
- Qdrant or Weaviate (purpose-built, more features)
- In-memory (for dev/testing only)

## Semantic Kernel Plugins (Function Calling)

Expose your application capabilities as SK plugins for the AI to call:

```csharp
public class OrderPlugin
{
    private readonly IOrderRepository _orders;

    [KernelFunction, Description("Get order details by order ID")]
    public async Task<string> GetOrderAsync(
        [Description("The order ID")] string orderId,
        CancellationToken ct)
    {
        var order = await _orders.GetByIdAsync(Guid.Parse(orderId), ct);
        return JsonSerializer.Serialize(order);
    }
}

// Register
kernel.Plugins.AddFromType<OrderPlugin>();
```

## Streaming Responses

For chat UIs, stream tokens as they arrive rather than waiting for the full response:

```csharp
await foreach (var chunk in kernel.InvokePromptStreamingAsync(prompt, ct: ct))
{
    await hubContext.Clients.User(userId).SendAsync("ReceiveChunk", chunk.ToString(), ct);
}
```

Use SignalR or SSE (Server-Sent Events) to push chunks to the browser.

## Cost and Latency Considerations

- Cache embeddings — they're deterministic; the same text always produces the same embedding
- Cache frequent LLM responses with a semantic similarity threshold (not just exact key match)
- Use GPT-4o-mini for classification/routing tasks; GPT-4o for complex reasoning
- Set `max_tokens` explicitly — unbounded responses burn budget and add latency
- Instrument every AI call: log the model, prompt tokens, completion tokens, latency, and cost
- Implement retry with exponential backoff for rate limit errors (429s are expected at scale)

## Security Concerns

- Never expose your OpenAI key to the client. All AI calls go through your backend.
- Sanitize user input before embedding in prompts — prompt injection is a real attack vector
- Don't store raw prompts if they contain PII — log a hash or anonymized version
- Validate and limit what SK plugins can do — a plugin that can delete records is dangerous in an agent loop
