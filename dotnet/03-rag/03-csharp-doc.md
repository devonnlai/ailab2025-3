# RAG Implementation with Azure OpenAI SDK (.NET)

This exercise will guide you through implementing a Retrieval-Augmented Generation (RAG) system using the Azure OpenAI SDK in .NET.

## Prerequisites

- [.NET 6.0 SDK](https://dotnet.microsoft.com/download/dotnet/6.0) or later
- An Azure account with an Azure OpenAI resource
- An Azure Cognitive Search resource (for the vector store)
- Visual Studio 2022 or Visual Studio Code

## Setup

### 1. Create a new .NET Console Application

```bash
dotnet new console -n AzureOpenAIRagImplementation
cd AzureOpenAIRagImplementation
```

### 2. Add the required NuGet packages

```bash
dotnet add package Azure.AI.OpenAI --version 2.2.0-beta.4
dotnet add package OpenAI --version 2.2.0-beta.4
dotnet add package Microsoft.Extensions.Configuration.Json --version 9.0.4
dotnet add package Azure.Search.Documents --version 11.6.0
dotnet add package Microsoft.SemanticKernel --version 1.49.0
```

### 3. Create configuration settings

Create an `appsettings.json` file in your project root and make entry in the project file to copy the settings files to output directory:

```json
{
  "AzureOpenAI": {
    "Endpoint": "https://YOUR_AZURE_OPENAI_RESOURCE_NAME.services.ai.azure.com/",
    "Key": "YOUR_AZURE_OPENAI_API_KEY",
    "CompletionDeploymentName": "YOUR_COMPLETION_DEPLOYMENT_NAME",
    "EmbeddingDeploymentName": "YOUR_EMBEDDING_DEPLOYMENT_NAME",
    "EmbeddingEndpoint": "YOUR_EMBEDDING_ENDPOINT",
    "EmbeddingKey": "YOUR_EMBEDDING_KEY"
  },
  "AzureCognitiveSearch": {
    "Endpoint": "https://YOUR_SEARCH_SERVICE_NAME.search.windows.net",
    "Key": "YOUR_SEARCH_SERVICE_KEY",
    "IndexName": "knowledge-index"
  }
}
```

### 4. Create the document model

Create a new file called `Document.cs`:

```csharp
using System.Text.Json.Serialization;

namespace AzureOpenAIRagImplementation
{
    public class Document
    {
        [JsonPropertyName("id")]
        public string Id { get; set; } = Guid.NewGuid().ToString();

        [JsonPropertyName("title")]
        public string Title { get; set; } = "";

        [JsonPropertyName("content")]
        public string Content { get; set; } = "";

        [JsonPropertyName("embedding")]
        public float[]? Embedding { get; set; }

        [JsonPropertyName("category")]
        public string Category { get; set; } = "";

        [JsonPropertyName("source")]
        public string Source { get; set; } = "";

        public override string ToString()
        {
            return $"Title: {Title}\nCategory: {Category}\nSource: {Source}\nContent: {Content}";
        }
    }
}
```

### 5. Create the vector embedding service

Create a new file called `EmbeddingService.cs`:

```csharp
using Azure;
using Azure.AI.OpenAI;

namespace AzureOpenAIRagImplementation
{
    public class EmbeddingService
    {
        private readonly AzureOpenAIClient _openAIClient;
        private readonly string _embeddingDeploymentName;

        public EmbeddingService(string endpoint, string key, string embeddingDeploymentName)
        {
            _openAIClient = new AzureOpenAIClient(
                new Uri(endpoint),
                new AzureKeyCredential(key));
            _embeddingDeploymentName = embeddingDeploymentName;
        }

        public async Task<float[]> GenerateEmbeddingsAsync(string text)
        {
            var embeddingClient = _openAIClient.GetEmbeddingClient(_embeddingDeploymentName);
            var response = await embeddingClient.GenerateEmbeddingsAsync([text]);

            return response.Value.First().ToFloats().ToArray();
        }
    }
}
```

### 6. Create the vector store service

Create a new file called `VectorStoreService.cs`:

```csharp
using Azure;
using Azure.Search.Documents;
using Azure.Search.Documents.Indexes;
using Azure.Search.Documents.Indexes.Models;
using Azure.Search.Documents.Models;

namespace AzureOpenAIRagImplementation
{
  public class VectorStoreService
  {
    private readonly string _searchEndpoint;
    private readonly string _searchKey;
    private readonly string _indexName;
    private readonly SearchIndexClient _searchIndexClient;
    private readonly SearchClient _searchClient;

    public VectorStoreService(string searchEndpoint, string searchKey, string indexName)
    {
      _searchEndpoint = searchEndpoint;
      _searchKey = searchKey;
      _indexName = indexName;

      _searchIndexClient = new SearchIndexClient(
          new Uri(_searchEndpoint),
          new AzureKeyCredential(_searchKey));

      _searchClient = new SearchClient(
          new Uri(_searchEndpoint),
          _indexName,
          new AzureKeyCredential(_searchKey));
    }

    public async Task CreateIndexIfNotExistsAsync()
    {
      // Create the search index with vector search capability
      var vectorSearchProfile = new VectorSearchProfile("my-vector-profile", "hnsw");

      // Create fields for the index
      var fieldBuilder = new FieldBuilder();
      var fields = new List<SearchField>
            {
                new SearchField("id", SearchFieldDataType.String) { IsKey = true, IsFilterable = true },
                new SearchField("title", SearchFieldDataType.String) { IsSearchable = true, IsFilterable = true },
                new SearchField("content", SearchFieldDataType.String) { IsSearchable = true },
                new VectorSearchField("embedding", 1536, "my-vector-profile")  // Adjust for your embedding model
                {
                    VectorSearchDimensions = 1536,  // Adjust for your embedding model
                },
                new SearchField("category", SearchFieldDataType.String) { IsFilterable = true, IsFacetable = true },
                new SearchField("source", SearchFieldDataType.String) { IsFilterable = true, IsFacetable = true }
            };

      // Create the index with the fields
      var index = new SearchIndex(_indexName)
      {
        Fields = fields,
        VectorSearch = new VectorSearch
        {
          Profiles = { vectorSearchProfile },
          Algorithms = { new HnswAlgorithmConfiguration("hnsw") {
              Parameters = new HnswParameters
              {
                  M = 4,
                  EfConstruction = 400,
                  EfSearch = 500,
                  Metric = VectorSearchAlgorithmMetric.Cosine
              }
            }
          }
        }
      };

      await _searchIndexClient.CreateOrUpdateIndexAsync(index);
      Console.WriteLine($"Index {_indexName} created successfully");
    }

    public async Task AddDocumentsAsync(List<Document> documents)
    {
      // Upload documents to the search index
      var batch = IndexDocumentsBatch.Upload(documents);
      var response = await _searchClient.IndexDocumentsAsync(batch);

      Console.WriteLine($"Added {documents.Count} documents to the search index");
    }

    public async Task<List<Document>> SearchSimilarDocumentsAsync(float[] queryEmbedding, int top = 3)
    {
      // Create vector query
      var vectorQuery = new VectorizedQuery(queryEmbedding)
      {
        Exhaustive = true,
        KNearestNeighborsCount = top,
        Fields = { "embedding" }
      };

      // Create the vector search options
      var options = new SearchOptions
      {
        IncludeTotalCount = true,
        Size = top,
        Select = { "id", "title", "content", "category", "source" },
      };

      options.VectorSearch = new VectorSearchOptions
      {
        Queries = { vectorQuery }
      };

      // Perform the vector search
      var response = await _searchClient.SearchAsync<Document>(options);

      var results = new List<Document>();
      await foreach (var result in response.Value.GetResultsAsync())
      {
        results.Add(result.Document);
      }

      return results;
    }
  }
}
```

### 7. Create the RAG service

Create a new file called `RagService.cs`:

```csharp
using Azure;
using Azure.AI.OpenAI;
using OpenAI.Chat;

namespace AzureOpenAIRagImplementation
{
    public class RagService
    {
        private readonly AzureOpenAIClient _openAIClient;
        private readonly string _completionDeploymentName;
        private readonly EmbeddingService _embeddingService;
        private readonly VectorStoreService _vectorStoreService;        public RagService(
            string openAIEndpoint,
            string openAIKey,
            string completionDeploymentName,
            EmbeddingService embeddingService,
            VectorStoreService vectorStoreService)
        {
            _openAIClient = new AzureOpenAIClient(new Uri(openAIEndpoint), new AzureKeyCredential(openAIKey));
            _completionDeploymentName = completionDeploymentName;
            _embeddingService = embeddingService;
            _vectorStoreService = vectorStoreService;
        }

        public async Task<string> QueryAsync(string query, int topK = 3)
        {
            // Step 1: Generate embeddings for the query
            var queryEmbedding = await _embeddingService.GenerateEmbeddingsAsync(query);

            // Step 2: Retrieve similar documents from the vector store
            var similarDocuments = await _vectorStoreService.SearchSimilarDocumentsAsync(queryEmbedding, topK);

            // Step 3: Prepare context from retrieved documents
            var context = string.Join("\n\n", similarDocuments.Select(doc => doc.ToString()));            // Step 4: Generate completion with the context and query
            var systemMessage = @"
You are a helpful assistant that answers questions based on the provided context.
If the context doesn't contain information to answer the question, say 'I don't have enough information to answer that question.'
Always cite the source of the information in your answer.
";

            var userMessage = $@"
Context:
{context}

Question: {query}

Answer the question based on the context provided. If the context doesn't contain relevant information, say 'I don't have enough information to answer that question.'
";

            ChatClient chatClient = _openAIClient.GetChatClient(_completionDeploymentName);

            var requestOptions = new ChatCompletionOptions()
            {
              MaxOutputTokenCount = 500,
              Temperature = 0.5f,
              TopP = 1.0f,
            };

            List<ChatMessage> messages =
            [
              new SystemChatMessage(systemMessage),
              new UserChatMessage(userMessage)
            ];

            var response = await chatClient.CompleteChatAsync(messages, requestOptions);
            var completion = response.Value;

            return completion.Content[0].Text;
        }

        public async Task AddDocumentsAsync(List<Document> documents)
        {
            // Step 1: Generate embeddings for each document
            foreach (var document in documents)
            {
                document.Embedding = await _embeddingService.GenerateEmbeddingsAsync(document.Content);
            }

            // Step 2: Add documents with embeddings to the vector store
            await _vectorStoreService.AddDocumentsAsync(documents);
        }
    }
}
```

### 8. Create the sample data loader

Create a new file called `SampleDataLoader.cs`:

```csharp
namespace AzureOpenAIRagImplementation
{
    public static class SampleDataLoader
    {
        public static List<Document> GetSampleDocuments()
        {
            return new List<Document>
            {
                new Document
                {
                    Title = "What is Azure OpenAI Service?",
                    Content = "Azure OpenAI Service provides REST API access to OpenAI's powerful language models including GPT-4, GPT-35-Turbo, and Embeddings model series. These models can be easily adapted to your specific task including content generation, summarization, semantic search, and natural language to code translation. Users can access the service through REST APIs, Python SDK, or our web-based interface in the Azure OpenAI Studio.",
                    Category = "Product Information",
                    Source = "Azure OpenAI Documentation"
                },
                new Document
                {
                    Title = "Azure OpenAI Service Pricing",
                    Content = "Azure OpenAI Service offers various pricing tiers based on the model and usage. GPT-4 models are charged at a premium rate compared to GPT-35-Turbo models. Embedding models are charged at the lowest rate. Prices are calculated based on the number of tokens processed, with tokens roughly corresponding to 4 characters in English text. For the most current pricing information, visit the Azure OpenAI pricing page.",
                    Category = "Pricing",
                    Source = "Azure OpenAI Pricing Page"
                },
                new Document
                {
                    Title = "Implementing RAG Pattern with Azure OpenAI",
                    Content = "Retrieval-Augmented Generation (RAG) is a pattern that enhances Large Language Models (LLMs) with additional context from external sources. In Azure OpenAI, implementing RAG typically involves using the Embeddings model to convert documents and queries into vector embeddings, storing these vectors in a vector database like Azure Cognitive Search, retrieving relevant documents based on query similarity, and then providing these documents as context to the GPT model along with the user's query. This approach grounds the LLM's responses in specific information, reducing hallucinations and providing more accurate answers based on your data.",
                    Category = "Technical Guide",
                    Source = "Azure AI Blog"
                },
                new Document
                {
                    Title = "Advantages of Azure OpenAI over Direct OpenAI API",
                    Content = "Azure OpenAI Service offers several advantages over using the OpenAI API directly. These include enterprise-grade security with Azure Active Directory integration, private networking, and data encryption. Azure OpenAI also provides compliance certifications that may be required for regulated industries. Additionally, it offers regional availability, allowing you to deploy models closer to your applications for reduced latency. For organizations already using Azure services, it provides integrated billing and support through existing Azure subscriptions.",
                    Category = "Comparison",
                    Source = "Microsoft Learn"
                },
                new Document
                {
                    Title = "Best Practices for Prompt Engineering in Azure OpenAI",
                    Content = "When working with Azure OpenAI models, effective prompt engineering can significantly improve results. Key best practices include: 1) Be specific and clear in your instructions, 2) Use examples to demonstrate desired outputs (few-shot learning), 3) Structure complex tasks into steps, 4) Experiment with system messages to set the AI's tone and behavior, 5) Use temperature settings to control randomness (lower for factual responses, higher for creative tasks), and 6) Implement validation to ensure outputs meet your requirements. For production applications, always implement human review processes for AI-generated content.",
                    Category = "Technical Guide",
                    Source = "Azure AI Documentation"
                }
            };
        }
    }
}
```

### 9. Update Program.cs

Replace the contents of `Program.cs` with the following code:

```csharp
using AzureOpenAIRagImplementation;
using Microsoft.Extensions.Configuration;

// Load configuration
var configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .Build();

// Get Azure OpenAI service configuration values
var openAIEndpoint = configuration["AzureOpenAI:Endpoint"] ?? throw new InvalidOperationException("Endpoint is not configured");
var openAIKey = configuration["AzureOpenAI:Key"] ?? throw new InvalidOperationException("API Key is not configured");
var completionDeploymentName = configuration["AzureOpenAI:CompletionDeploymentName"] ?? throw new InvalidOperationException("Completion deployment name is not configured");
var embeddingDeploymentName = configuration["AzureOpenAI:EmbeddingDeploymentName"] ?? throw new InvalidOperationException("Embedding deployment name is not configured");
var embeddingEndpoint = configuration["AzureOpenAI:EmbeddingEndpoint"] ?? throw new InvalidOperationException("Embedding Endpoint is not configured");
var embeddingKey = configuration["AzureOpenAI:EmbeddingKey"] ?? throw new InvalidOperationException("Embedding API Key is not configured");

// Get Azure Cognitive Search configuration values
var searchEndpoint = configuration["AzureCognitiveSearch:Endpoint"];
var searchKey = configuration["AzureCognitiveSearch:Key"];
var indexName = configuration["AzureCognitiveSearch:IndexName"];

// Initialize services
var embeddingService = new EmbeddingService(embeddingEndpoint, embeddingKey, embeddingDeploymentName);
var vectorStoreService = new VectorStoreService(searchEndpoint, searchKey, indexName);
var ragService = new RagService(openAIEndpoint, openAIKey, completionDeploymentName, embeddingService, vectorStoreService);

// Create the search index if it doesn't exist
await vectorStoreService.CreateIndexIfNotExistsAsync();

// Add sample documents to the vector store
var sampleDocuments = SampleDataLoader.GetSampleDocuments();
await ragService.AddDocumentsAsync(sampleDocuments);

Console.WriteLine("Sample documents added to the vector store.");
Console.WriteLine();

// Sample queries to test the RAG system
var sampleQueries = new List<string>
{
    "What is Azure OpenAI Service?",
    "How much does Azure OpenAI cost?",
    "How does the RAG pattern work?",
    "Why should I use Azure OpenAI instead of direct OpenAI API?",
    "What are some best practices for prompt engineering?"
};

foreach (var query in sampleQueries)
{
    Console.WriteLine($"Query: {query}");
    Console.WriteLine();

    try
    {
        var answer = await ragService.QueryAsync(query);
        Console.WriteLine($"Answer: {answer}");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error: {ex.Message}");
    }

    Console.WriteLine(new string('-', 80));
    Console.WriteLine();
}

// Interactive mode
Console.WriteLine("Enter your own queries (or type 'exit' to quit):");
while (true)
{
    Console.WriteLine();
    Console.Write("> ");
    var userQuery = Console.ReadLine();

    if (string.IsNullOrWhiteSpace(userQuery) || userQuery.ToLower() == "exit")
        break;

    try
    {
        var answer = await ragService.QueryAsync(userQuery);
        Console.WriteLine();
        Console.WriteLine($"Answer: {answer}");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error: {ex.Message}");
    }
}
```

### 10. Run the application

```bash
dotnet run
```

## Expected Output

The application will:
1. Create a search index in Azure Cognitive Search (if it doesn't exist)
2. Add sample documents to the index with vector embeddings
3. Process a set of sample queries using the RAG pattern
4. Allow you to enter your own queries in interactive mode

For each query, you should see an answer that incorporates information from the relevant documents retrieved from the vector store.

## Exercises

1. Implement document chunking to break large documents into smaller pieces before embedding.
2. Add a hybrid search capability that combines vector search with traditional keyword search.
3. Implement conversational memory to maintain context across multiple user queries.
4. Add a feature to load documents from files (e.g., PDFs, Word documents, text files).
5. Implement a caching mechanism to avoid generating embeddings for the same text multiple times.
6. Create a simple web API to expose the RAG functionality as a service.

## Troubleshooting

- **Authentication errors**: Double-check your API keys and endpoints in `appsettings.json`.
- **Vector search errors**: Ensure that your Azure Cognitive Search service supports vector search (requires Standard tier or above).
- **Embedding dimension errors**: Make sure the vector dimensions in the search index match the output of your embedding model.