# Hello World with Azure OpenAI SDK (.NET)

This exercise will guide you through setting up a basic "Hello World" application using the Azure OpenAI SDK in .NET.

## Prerequisites

- [.NET 6.0 SDK](https://dotnet.microsoft.com/download/dotnet/6.0) or later
- An Azure account with an Azure OpenAI resource
- Visual Studio 2022 or Visual Studio Code

## Setup

### 1. Create a new .NET Console Application

```bash
dotnet new console -n AzureOpenAIHelloWorld
cd AzureOpenAIHelloWorld
```

### 2. Add the Azure OpenAI SDK NuGet package

```bash
dotnet add package Azure.AI.OpenAI --version 2.1.0
dotnet add package Microsoft.Extensions.Configuration.Json --version 9.0.4
```

### 3. Create configuration settings

Create an `appsettings.json` file in your project root and add your Azure OpenAI service details:

```bash
dotnet add package Microsoft.Extensions.Configuration.Json
```

Add a new file called `appsettings.json`:

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

### 4. Update Program.cs

Replace the contents of `Program.cs` with the following code:

```csharp
using Azure;
using Azure.AI.OpenAI;
using Microsoft.Extensions.Configuration;
using OpenAI.Chat;

// Load configuration
var configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .AddJsonFile("appsettings.development.json", optional: true)
    .Build();

// Get Azure OpenAI service configuration values
var endpoint = configuration["AzureOpenAI:Endpoint"] ?? throw new InvalidOperationException("Endpoint is not configured");
var key = configuration["AzureOpenAI:Key"] ?? throw new InvalidOperationException("API Key is not configured");
var deploymentName = configuration["AzureOpenAI:DeploymentName"] ?? throw new InvalidOperationException("Deployment name is not configured");
var model = configuration["AzureOpenAI:Model"] ?? throw new InvalidOperationException("Model is not configured");

// Initialize the Azure OpenAI client
var openAIClient = new AzureOpenAIClient(
    new Uri(endpoint),
    new AzureKeyCredential(key));

ChatClient chatClient = openAIClient.GetChatClient(deploymentName);

var requestOptions = new ChatCompletionOptions()
{
  MaxOutputTokenCount = 4096,
  Temperature = 1.0f,
  TopP = 1.0f,
};

List<ChatMessage> messages =
[
  new SystemChatMessage("You are a helpful assistant."),
  new UserChatMessage("Hello, world! What can you do for me?"),
];

try
{
  // Send the completion request
  var response = await chatClient.CompleteChatAsync(messages, requestOptions);
  var completion = response.Value;

  // Display the response
  Console.WriteLine("Azure OpenAI Response:\n");
  Console.WriteLine(completion.Content[0].Text);
}
catch (Exception ex)
{
  Console.WriteLine($"Error: {ex.Message}");
}
```

### 5. Run the application

```bash
dotnet run
```

## Expected Output

You should see a response from the Azure OpenAI service to your "Hello, world!" message, similar to:

```
Azure OpenAI Response:

Hello! I'm an AI assistant, and I'm here to help you with a variety of tasks. I can answer questions, provide information on various topics, assist with writing tasks, brainstorm ideas, explain concepts, and more. Feel free to ask me anything, and I'll do my best to assist you!
```

## Exercises

1. Try modifying the system message to give the AI a different personality or role.
2. Add a loop to allow for multiple conversation turns with the AI.
3. Experiment with different parameters like temperature and max tokens to see how they affect the model's responses.

## Troubleshooting

- **Authentication errors**: Double-check your API key and endpoint in `appsettings.json`.
- **Deployment errors**: Make sure you've created a deployment for one of the models (e.g., gpt-4, gpt-35-turbo) in the Azure OpenAI service.
- **Missing packages**: Run `dotnet restore` to ensure all packages are installed.