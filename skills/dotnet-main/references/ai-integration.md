# AI Integration with .NET Reference

## Semantic Kernel — Setup
```csharp
var kernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion(
        deploymentName: "gpt-4",
        endpoint: config["AzureOpenAI:Endpoint"]!,
        apiKey: config["AzureOpenAI:Key"]!)
    .Build();
```

## Chat Completion
```csharp
var chat = kernel.GetRequiredService<IChatCompletionService>();
var history = new ChatHistory("You are a helpful assistant.");
history.AddUserMessage("Summarize this order: " + orderJson);

var response = await chat.GetChatMessageContentAsync(history);
Console.WriteLine(response.Content);
```

## Semantic Kernel Plugins (tools)
```csharp
public class OrderPlugin
{
    [KernelFunction, Description("Get order details by ID")]
    public async Task<string> GetOrder(
        [Description("The order ID")] string orderId)
    {
        // fetch from DB
        return JsonSerializer.Serialize(order);
    }
}

kernel.Plugins.AddFromType<OrderPlugin>();
```

## Azure OpenAI vs OpenAI SDK
| | Azure OpenAI | OpenAI SDK |
|---|---|---|
| Data residency | Yes (EU, US) | No |
| Enterprise SLA | Yes | No |
| Private endpoint | Yes | No |
| Cost | Same or higher | Lower |

## ML.NET (on-device, no API)
```csharp
// Binary classification
var pipeline = mlContext.Transforms.Text.FeaturizeText("Features", "ReviewText")
    .Append(mlContext.BinaryClassification.Trainers.SdcaLogisticRegression());

var model = pipeline.Fit(trainingData);
var predictor = mlContext.Model.CreatePredictionEngine<Review, Prediction>(model);
var result = predictor.Predict(new Review { ReviewText = "Great product!" });
```

## Common Mistakes
1. Putting API keys in code or appsettings (use Key Vault)
2. Not setting max tokens (runaway costs)
3. Not handling rate limit exceptions with retry policy
4. Sending too much context (use chunking / RAG patterns)
5. Forgetting that AI responses are non-deterministic — always validate output
