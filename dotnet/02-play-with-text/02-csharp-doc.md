# Text Summarization and Categorization with Azure OpenAI SDK (.NET)

This exercise will guide you through implementing text summarization and categorization using the Azure OpenAI SDK in .NET.

## Prerequisites

- [.NET 6.0 SDK](https://dotnet.microsoft.com/download/dotnet/6.0) or later
- An Azure account with an Azure OpenAI resource
- Visual Studio 2022 or Visual Studio Code

## Setup

### 1. Create a new .NET Console Application

```bash
dotnet new console -n AzureOpenAITextProcessing
cd AzureOpenAITextProcessing
```

### 2. Add the required NuGet packages

```bash
dotnet add package Azure.AI.OpenAI --version 2.1.0
dotnet add package OpenAI --version 2.1.0
dotnet add package Microsoft.Extensions.Configuration.Json --version 9.0.4
```

### 3. Create configuration settings

Create an `appsettings.json` file in your project root:

```json
{
  "AzureOpenAI": {
    "Endpoint": "https://YOUR_AZURE_OPENAI_RESOURCE_NAME.services.ai.azure.com/",
    "Key": "YOUR_AZURE_OPENAI_API_KEY",
    "DeploymentName": "YOUR_DEPLOYMENT_NAME",
    "Model": "YOUR_MODEL_NAME"
  }
}
```

### 4. Create a model for text processing

Create a new file called `TextProcessingModel.cs`:

```csharp
namespace AzureOpenAITextProcessing
{
    public class TextProcessingModel
    {
        public string OriginalText { get; set; } = "";
        public string Summary { get; set; } = "";
        public string Category { get; set; } = "";
        public List<string> Keywords { get; set; } = new List<string>();
        public string SentimentAnalysis { get; set; } = "";
    }
}
```

### 5. Create a text processing service

Create a new file called `TextProcessingService.cs`:

```csharp
using Azure;
using Azure.AI.OpenAI;
using System.Text.Json;
using OpenAI.Chat;

namespace AzureOpenAITextProcessing
{
    public class TextProcessingService
    {
        private readonly AzureOpenAIClient _openAIClient;
        private readonly string _deploymentName;

        public TextProcessingService(string endpoint, string key, string deploymentName)
        {
            _openAIClient = new AzureOpenAIClient(
                new Uri(endpoint),
                new AzureKeyCredential(key));
            _deploymentName = deploymentName;
        }

        public async Task<string> SummarizeTextAsync(string text, int maxTokens = 150)
        {
            ChatClient chatClient = _openAIClient.GetChatClient(_deploymentName);

            var requestOptions = new ChatCompletionOptions()
            {
              MaxOutputTokenCount = maxTokens,
              Temperature = 1.0f,
              TopP = 1.0f,
            };

            List<ChatMessage> messages =
            [
              new SystemChatMessage("You are a helpful AI assistant that summarizes text concisely."),
              new UserChatMessage($"Please summarize the following text in about 2-3 sentences: \n\n{text}")
            ];

            var response = await chatClient.CompleteChatAsync(messages, requestOptions);
            var completion = response.Value;

            return completion.Content[0].Text;
        }        public async Task<string> CategorizeTextAsync(string text, int maxTokens = 30)
        {
            ChatClient chatClient = _openAIClient.GetChatClient(_deploymentName);

            var requestOptions = new ChatCompletionOptions()
            {
              MaxOutputTokenCount = maxTokens,
              Temperature = 0.0f, // Lower temperature for more deterministic outputs
              TopP = 1.0f,
            };

            List<ChatMessage> messages =
            [
              new SystemChatMessage("You are a helpful AI assistant that categorizes text into a single appropriate category."),
              new UserChatMessage($"Please categorize the following text into exactly one category (like technology, business, politics, health, etc.). Return only the category name, nothing else.\n\n{text}")
            ];

            var response = await chatClient.CompleteChatAsync(messages, requestOptions);
            var completion = response.Value;

            return completion.Content[0].Text.Trim();
        }        public async Task<List<string>> ExtractKeywordsAsync(string text, int maxTokens = 50)
        {
            ChatClient chatClient = _openAIClient.GetChatClient(_deploymentName);

            var requestOptions = new ChatCompletionOptions()
            {
              MaxOutputTokenCount = maxTokens,
              Temperature = 0.0f, // Lower temperature for more deterministic outputs
              TopP = 1.0f,
            };

            List<ChatMessage> messages =
            [
              new SystemChatMessage("You are a helpful AI assistant that extracts keywords from text."),
              new UserChatMessage($"Extract 5-7 important keywords from the following text. Return only a JSON array of strings, nothing else:\n\n{text}")
            ];

            var response = await chatClient.CompleteChatAsync(messages, requestOptions);
            var completion = response.Value;

            var content = completion.Content[0].Text.Trim();

            // Parse the JSON array
            try
            {
                return JsonSerializer.Deserialize<List<string>>(content) ?? new List<string>();
            }
            catch
            {
                // If parsing fails, try to extract keywords manually
                return content.Replace("[", "").Replace("]", "").Replace("\"", "")
                    .Split(',').Select(k => k.Trim()).Where(k => !string.IsNullOrEmpty(k)).ToList();
            }
        }        public async Task<string> AnalyzeSentimentAsync(string text, int maxTokens = 50)
        {
            ChatClient chatClient = _openAIClient.GetChatClient(_deploymentName);

            var requestOptions = new ChatCompletionOptions()
            {
              MaxOutputTokenCount = maxTokens,
              Temperature = 0.0f, // Lower temperature for more deterministic outputs
              TopP = 1.0f,
            };

            List<ChatMessage> messages =
            [
              new SystemChatMessage("You are a helpful AI assistant that analyzes sentiment."),
              new UserChatMessage($"Analyze the sentiment of the following text. Respond with only one word: 'positive', 'negative', or 'neutral'.\n\n{text}")
            ];

            var response = await chatClient.CompleteChatAsync(messages, requestOptions);
            var completion = response.Value;

            return completion.Content[0].Text.Trim().ToLower();
        }

        public async Task<TextProcessingModel> ProcessTextAsync(string text)
        {
            var result = new TextProcessingModel
            {
                OriginalText = text
            };

            // Process text in parallel for efficiency
            var summaryTask = SummarizeTextAsync(text);
            var categoryTask = CategorizeTextAsync(text);
            var keywordsTask = ExtractKeywordsAsync(text);
            var sentimentTask = AnalyzeSentimentAsync(text);

            await Task.WhenAll(summaryTask, categoryTask, keywordsTask, sentimentTask);

            result.Summary = await summaryTask;
            result.Category = await categoryTask;
            result.Keywords = await keywordsTask;
            result.SentimentAnalysis = await sentimentTask;

            return result;
        }
    }
}
```

### 6. Update Program.cs

Replace the contents of `Program.cs` with the following code:

```csharp
using AzureOpenAITextProcessing;
using Microsoft.Extensions.Configuration;

// Load configuration
var configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .Build();

// Get Azure OpenAI service configuration values
var endpoint = configuration["AzureOpenAI:Endpoint"] ?? throw new InvalidOperationException("Endpoint is not configured");
var key = configuration["AzureOpenAI:Key"] ?? throw new InvalidOperationException("API Key is not configured");
var deploymentName = configuration["AzureOpenAI:DeploymentName"] ?? throw new InvalidOperationException("Deployment name is not configured");
var model = configuration["AzureOpenAI:Model"] ?? throw new InvalidOperationException("Model is not configured");

// Initialize the text processing service
var textProcessingService = new TextProcessingService(endpoint, key, deploymentName);

// Sample texts for processing
var sampleTexts = new List<string>
{
    "The European Union has approved new regulations on artificial intelligence that will require AI companies to disclose training data and conduct risk assessments. The regulations, which focus on high-risk AI applications, aim to ensure AI systems are safe, transparent, and respect fundamental rights while still promoting innovation. Critics argue the rules may stifle growth in the industry, while proponents say they'll help build trust in AI technology.",

    "A new study published in Nature Medicine suggests that regular moderate exercise may reduce the risk of severe COVID-19 outcomes. Researchers analyzed data from over 50,000 patients and found that those who engaged in at least 150 minutes of moderate activity per week had 31% lower odds of hospitalization when infected with COVID-19. The study controlled for age, BMI, and pre-existing conditions.",

    "The latest smartphone from Apple features a groundbreaking camera system with a 108-megapixel sensor, allowing for unprecedented detail in photos even in low light conditions. The phone also includes a new AI-powered image processing system that can recognize objects in real-time and adjust settings accordingly. Battery life has been improved by 25% compared to the previous model."
};

// Process the sample texts
Console.WriteLine("Processing sample texts...\n");

foreach (var text in sampleTexts)
{
    Console.WriteLine("Original Text:");
    Console.WriteLine(text);
    Console.WriteLine();

    try
    {
        var result = await textProcessingService.ProcessTextAsync(text);

        Console.WriteLine("Summary:");
        Console.WriteLine(result.Summary);
        Console.WriteLine();

        Console.WriteLine("Category:");
        Console.WriteLine(result.Category);
        Console.WriteLine();

        Console.WriteLine("Keywords:");
        Console.WriteLine(string.Join(", ", result.Keywords));
        Console.WriteLine();

        Console.WriteLine("Sentiment:");
        Console.WriteLine(result.SentimentAnalysis);
        Console.WriteLine();
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error processing text: {ex.Message}");
    }

    Console.WriteLine(new string('-', 80));
    Console.WriteLine();
}

// Interactive mode
Console.WriteLine("Enter your own text to process (or type 'exit' to quit):");
while (true)
{
    Console.WriteLine();
    var userText = Console.ReadLine();

    if (string.IsNullOrWhiteSpace(userText) || userText.ToLower() == "exit")
        break;

    try
    {
        var result = await textProcessingService.ProcessTextAsync(userText);

        Console.WriteLine("Summary:");
        Console.WriteLine(result.Summary);
        Console.WriteLine();

        Console.WriteLine("Category:");
        Console.WriteLine(result.Category);
        Console.WriteLine();

        Console.WriteLine("Keywords:");
        Console.WriteLine(string.Join(", ", result.Keywords));
        Console.WriteLine();

        Console.WriteLine("Sentiment:");
        Console.WriteLine(result.SentimentAnalysis);
        Console.WriteLine();
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error processing text: {ex.Message}");
    }
}
```

### 7. Run the application

```bash
dotnet run
```

## Expected Output

For each sample text, you should see an output that includes:
- The original text
- A concise summary
- A single category classification
- A list of extracted keywords
- A sentiment analysis (positive, negative, or neutral)

After processing the sample texts, you can enter your own text for processing.

## Exercises

1. Add more categorization options or create a custom category list.
2. Implement a function to compare two texts and determine their similarity.
3. Extend the application to process text from a file or URL.
4. Create a more advanced sentiment analysis that provides a confidence score or multiple emotions.
5. Implement a function to extract key entities (people, organizations, locations) from the text.

## Troubleshooting

- **Authentication errors**: Double-check your API key and endpoint in `appsettings.json`.
- **JSON parsing errors**: The JSON output format may sometimes vary; improve the parsing logic to handle edge cases.
- **Token limit errors**: Reduce the input text length or increase the maxTokens parameter for large texts.