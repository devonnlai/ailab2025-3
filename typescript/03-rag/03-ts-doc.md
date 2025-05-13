# RAG Implementation with Azure OpenAI SDK (TypeScript)

This exercise will guide you through implementing a Retrieval-Augmented Generation (RAG) system using the Azure OpenAI SDK in TypeScript.

## Prerequisites

- [Node.js](https://nodejs.org/) (v16.x or later)
- npm (included with Node.js)
- An Azure account with an Azure OpenAI resource
- An Azure Cognitive Search resource (for the vector store)
- Visual Studio Code (recommended)

## Setup

### 1. Create a new Node.js project

```bash
mkdir azure-openai-rag-ts
cd azure-openai-rag-ts
npm init -y
```

### 2. Install required dependencies

```bash
npm install openai @azure/search-documents dotenv typescript ts-node @types/node readline-sync @types/readline-sync uuid @types/uuid
```

### 3. Initialize TypeScript configuration

```bash
npx tsc --init
```

Update the `tsconfig.json` file with the following content:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

### 4. Create environment variables

Create a `.env` file in the project root:

```
# Azure OpenAI settings for completions
AZURE_OPENAI_ENDPOINT=https://YOUR_AZURE_OPENAI_RESOURCE_NAME.openai.azure.com/
AZURE_OPENAI_KEY=YOUR_AZURE_OPENAI_API_KEY
AZURE_OPENAI_API_VERSION=2024-04-01-preview

# Specific deployment and model names for RAG implementation
AZURE_OPENAI_DEPLOYMENT_NAME=YOUR_COMPLETION_DEPLOYMENT_NAME
AZURE_OPENAI_EMBEDDING_ENDPOINT=YOUR_EMBEDDING_ENDPOINT
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=YOUR_EMBEDDING_DEPLOYMENT_NAME
AZURE_OPENAI_EMBEDDING_KEY=YOUR_EMBEDDING_KEY
AZURE_OPENAI_EMBEDDING_API_VERSION=2023-05-15

# Azure Cognitive Search settings
AZURE_SEARCH_ENDPOINT=https://YOUR_SEARCH_SERVICE_NAME.search.windows.net
AZURE_SEARCH_KEY=YOUR_SEARCH_SERVICE_KEY
AZURE_SEARCH_INDEX_NAME=knowledge-index
```

### 5. Update package.json

Update the `package.json` file to add scripts:

```json
{
  "scripts": {
    "start": "ts-node index.ts",
    "build": "tsc"
  }
}
```

### 6. Set up the directory structure

Ensure your project structure looks like this:

```
azure-openai-rag-ts/
├── .env
├── package.json
├── tsconfig.json
├── index.ts
├── models/
│   └── document.ts
├── services/
│   ├── embeddingService.ts
│   ├── vectorStoreService.ts
│   └── ragService.ts
└── data/
    └── sampleDataLoader.ts
```

### 7. Create the document model

Create a file named `models/document.ts`:

```typescript
export interface Document {
  id: string;
  title: string;
  content: string;
  embedding?: number[];
  category: string;
  source: string;
}
```

### 8. Create the embedding service

Create a file named `services/embeddingService.ts`:

```typescript
import { AzureOpenAI } from "openai";

export class EmbeddingService {
  private client: AzureOpenAI;
  private modelName: string;

  constructor(endpoint: string, apiKey: string, deploymentName: string, modelName: string, apiVersion: string) {
    this.client = new AzureOpenAI({
      endpoint,
      apiKey,
      deployment: deploymentName,
      apiVersion
    });
    this.modelName = modelName;
  }

  async generateEmbeddings(text: string): Promise<number[]> {
    const response = await this.client.embeddings.create({
      input: [text],
      model: this.modelName
    });
    return Array.from(response.data[0].embedding);
  }
}
```

### 9. Create the vector store service

Create a file named `services/vectorStoreService.ts`:

```typescript
import {
    SearchClient,
    SearchIndexClient,
    AzureKeyCredential,
    SearchIndex,
    VectorSearchAlgorithmConfiguration,
    VectorSearchProfile
} from "@azure/search-documents";
import { Document } from "../models/document";

export class VectorStoreService {
    private searchIndexClient: SearchIndexClient;
    private searchClient: SearchClient<Document>;
    private indexName: string;

    constructor(endpoint: string, apiKey: string, indexName: string) {
        this.indexName = indexName;
        this.searchIndexClient = new SearchIndexClient(
            endpoint,
            new AzureKeyCredential(apiKey)
        );
        this.searchClient = new SearchClient(
            endpoint,
            indexName,
            new AzureKeyCredential(apiKey)
        );
    }
    async createIndexIfNotExists(): Promise<void> {
        // Check if index exists
        const indexNamesIterator = await this.searchIndexClient.listIndexesNames();
        const indexNames = [];
        for await (const indexName of indexNamesIterator) {
            indexNames.push(indexName);
        }

        if (indexNames.includes(this.indexName)) {
            console.log(`Index ${this.indexName} already exists`);
            return;
        }

        // Create the vector search algorithm and profile
        const algorithm: VectorSearchAlgorithmConfiguration = {
            name: "hnsw",
            kind: "hnsw",
            parameters: {
                metric: "cosine",
                m: 4,
                efConstruction: 400,
                efSearch: 500
            }
        };

        const vectorSearchProfile: VectorSearchProfile = {
            name: "my-vector-profile",
            algorithmConfigurationName: "hnsw"
        };

        // Create the index with vector search capabilities
        const index: SearchIndex = {
            name: this.indexName,
            fields: [
                {
                    name: "id",
                    type: "Edm.String",
                    key: true,
                    filterable: true
                },
                {
                    name: "title",
                    type: "Edm.String",
                    searchable: true,
                    filterable: true
                },
                {
                    name: "content",
                    type: "Edm.String",
                    searchable: true
                },
                {
                    name: "embedding",
                    type: "Collection(Edm.Single)",
                    searchable: true,
                    vectorSearchDimensions: 1536, // Adjust for your embedding model
                    vectorSearchProfileName: "my-vector-profile"
                },
                {
                    name: "category",
                    type: "Edm.String",
                    filterable: true,
                    facetable: true
                },
                {
                    name: "source",
                    type: "Edm.String",
                    filterable: true,
                    facetable: true
                }
            ],
            vectorSearch: {
                algorithms: [algorithm],
                profiles: [vectorSearchProfile]
            }
        };

        await this.searchIndexClient.createOrUpdateIndex(index);
        console.log(`Index ${this.indexName} created successfully`);
    }

    async addDocuments(documents: Document[]): Promise<void> {
        // Add or update documents in the search index
        const response = await this.searchClient.uploadDocuments(documents);
        console.log(`Added ${documents.length} documents to the search index`);
    }

    async searchSimilarDocuments(queryEmbedding: number[], top: number = 3): Promise<Document[]> {
        // Perform a vector search using the query embedding
        const results = await this.searchClient.search("", {
            vectorSearchOptions: {
                queries: [
                    {
                        vector: queryEmbedding,
                        fields: ["embedding"],
                        kNearestNeighborsCount: top,
                        kind: 'vector'
                    }
                ]
            },
            select: ["id", "title", "content", "category", "source"],
            top: top
        });

        const documents: Document[] = [];
        for await (const result of results.results) {
            documents.push(result.document as Document);
        }

        return documents;
    }
}
```

### 10. Create the RAG service

Create a file named `services/ragService.ts`:

```typescript
import { AzureOpenAI } from "openai";
import { Document } from "../models/document";
import { EmbeddingService } from "./embeddingService";
import { VectorStoreService } from "./vectorStoreService";
import { ResponseInput } from 'openai/resources/responses/responses.mjs';

export class RagService {
  private client: AzureOpenAI;
  private modelName: string;
  private embeddingService: EmbeddingService;
  private vectorStoreService: VectorStoreService;

  constructor(
    openAIEndpoint: string,
    openAIKey: string,
    completionDeploymentName: string,
    modelName: string,
    apiVersion: string,
    embeddingService: EmbeddingService,
    vectorStoreService: VectorStoreService
  ) {
    this.client = new AzureOpenAI({
      endpoint: openAIEndpoint,
      apiKey: openAIKey,
      deployment: completionDeploymentName,
      apiVersion: apiVersion
    });
    this.modelName = modelName;
    this.embeddingService = embeddingService;
    this.vectorStoreService = vectorStoreService;
  }

  async query(query: string, topK: number = 3): Promise<string> {
    // Step 1: Generate embeddings for the query
    const queryEmbedding = await this.embeddingService.generateEmbeddings(query);

    // Step 2: Retrieve similar documents from the vector store
    const similarDocuments = await this.vectorStoreService.searchSimilarDocuments(queryEmbedding, topK);

    // Step 3: Prepare context from retrieved documents
    const context = similarDocuments.map(doc =>
      `Title: ${doc.title}\nCategory: ${doc.category}\nSource: ${doc.source}\nContent: ${doc.content}`
    ).join("\n\n");

    // Step 4: Generate completion with the context and query
    const systemMessage = `
You are a helpful assistant that answers questions based on the provided context.
If the context doesn't contain information to answer the question, say 'I don't have enough information to answer that question.'
Always cite the source of the information in your answer.
`;

    const userMessage = `
Context:
${context}

Question: ${query}

Answer the question based on the context provided. If the context doesn't contain relevant information, say 'I don't have enough information to answer that question.'
`;    const messages: ResponseInput = [
      { role: "system", content: systemMessage },
      { role: "user", content: userMessage }
    ];

    const response = await this.client.responses.create({
      input: messages,
      max_output_tokens: 500,
      temperature: 0.5,
      model: this.modelName
    });

    return response.output_text || "No response generated.";
  }

  async addDocuments(documents: Document[]): Promise<void> {
    // Step 1: Generate embeddings for each document
    for (const document of documents) {
      document.embedding = await this.embeddingService.generateEmbeddings(document.content);
    }

    // Step 2: Add documents with embeddings to the vector store
    await this.vectorStoreService.addDocuments(documents);
  }
}
```

### 11. Create the sample data loader

Create a file named `data/sampleDataLoader.ts`:

```typescript
import { Document } from "../models/document";
import { v4 as uuidv4 } from "uuid";

export function getSampleDocuments(): Document[] {
  return [
    {
      id: uuidv4(),
      title: "What is Azure OpenAI Service?",
      content: "Azure OpenAI Service provides REST API access to OpenAI's powerful language models including GPT-4, GPT-35-Turbo, and Embeddings model series. These models can be easily adapted to your specific task including content generation, summarization, semantic search, and natural language to code translation. Users can access the service through REST APIs, Python SDK, or our web-based interface in the Azure OpenAI Studio.",
      category: "Product Information",
      source: "Azure OpenAI Documentation"
    },
    {
      id: uuidv4(),
      title: "Azure OpenAI Service Pricing",
      content: "Azure OpenAI Service offers various pricing tiers based on the model and usage. GPT-4 models are charged at a premium rate compared to GPT-35-Turbo models. Embedding models are charged at the lowest rate. Prices are calculated based on the number of tokens processed, with tokens roughly corresponding to 4 characters in English text. For the most current pricing information, visit the Azure OpenAI pricing page.",
      category: "Pricing",
      source: "Azure OpenAI Pricing Page"
    },
    {
      id: uuidv4(),
      title: "Implementing RAG Pattern with Azure OpenAI",
      content: "Retrieval-Augmented Generation (RAG) is a pattern that enhances Large Language Models (LLMs) with additional context from external sources. In Azure OpenAI, implementing RAG typically involves using the Embeddings model to convert documents and queries into vector embeddings, storing these vectors in a vector database like Azure Cognitive Search, retrieving relevant documents based on query similarity, and then providing these documents as context to the GPT model along with the user's query. This approach grounds the LLM's responses in specific information, reducing hallucinations and providing more accurate answers based on your data.",
      category: "Technical Guide",
      source: "Azure AI Blog"
    },
    {
      id: uuidv4(),
      title: "Advantages of Azure OpenAI over Direct OpenAI API",
      content: "Azure OpenAI Service offers several advantages over using the OpenAI API directly. These include enterprise-grade security with Azure Active Directory integration, private networking, and data encryption. Azure OpenAI also provides compliance certifications that may be required for regulated industries. Additionally, it offers regional availability, allowing you to deploy models closer to your applications for reduced latency. For organizations already using Azure services, it provides integrated billing and support through existing Azure subscriptions.",
      category: "Comparison",
      source: "Microsoft Learn"
    },
    {
      id: uuidv4(),
      title: "Best Practices for Prompt Engineering in Azure OpenAI",
      content: "When working with Azure OpenAI models, effective prompt engineering can significantly improve results. Key best practices include: 1) Be specific and clear in your instructions, 2) Use examples to demonstrate desired outputs (few-shot learning), 3) Structure complex tasks into steps, 4) Experiment with system messages to set the AI's tone and behavior, 5) Use temperature settings to control randomness (lower for factual responses, higher for creative tasks), and 6) Implement validation to ensure outputs meet your requirements. For production applications, always implement human review processes for AI-generated content.",
      category: "Technical Guide",
      source: "Azure AI Documentation"
    }
  ];
}
```

### 12. Create the main application file

Create a file named `index.ts`:

```typescript
import * as dotenv from "dotenv";
import * as readline from "readline-sync";
import { EmbeddingService } from "./services/embeddingService";
import { VectorStoreService } from "./services/vectorStoreService";
import { RagService } from "./services/ragService";
import { getSampleDocuments } from "./data/sampleDataLoader";

// Load environment variables
dotenv.config();

async function main() {  // Get configuration from environment variables
    const openAIEndpoint = process.env.AZURE_OPENAI_ENDPOINT || "";
    const openAIKey = process.env.AZURE_OPENAI_KEY || "";
    const apiVersion = process.env.AZURE_OPENAI_API_VERSION || "";

    // Get deployment and model names
    const completionDeploymentName = process.env.AZURE_OPENAI_DEPLOYMENT_NAME || "";
    const completionModelName = process.env.AZURE_OPENAI_MODEL_NAME || "";
    const embeddingDeploymentName = process.env.AZURE_OPENAI_EMBEDDING_DEPLOYMENT || "";
    const embeddingModelName = process.env.AZURE_OPENAI_EMBEDDING_MODEL || "";
    const embeddingApiVersion = process.env.AZURE_OPENAI_EMBEDDING_API_VERSION || "";
    const embeddingEndpoint = process.env.AZURE_OPENAI_EMBEDDING_ENDPOINT || "";
    const embeddingKey = process.env.AZURE_OPENAI_EMBEDDING_KEY || "";

    const searchEndpoint = process.env.AZURE_SEARCH_ENDPOINT || "";
    const searchKey = process.env.AZURE_SEARCH_KEY || "";
    const indexName = process.env.AZURE_SEARCH_INDEX_NAME || "";
    if (!openAIEndpoint || !openAIKey || !apiVersion ||
        !completionDeploymentName || !completionModelName ||
        !embeddingDeploymentName || !embeddingModelName ||
        !searchEndpoint || !searchKey || !indexName || !embeddingEndpoint || !embeddingKey) {
        throw new Error("Missing required environment variables");
    }
    // Initialize services
    const embeddingService = new EmbeddingService(embeddingEndpoint, embeddingKey, embeddingDeploymentName, embeddingModelName, embeddingApiVersion);
    const vectorStoreService = new VectorStoreService(searchEndpoint, searchKey, indexName);
    const ragService = new RagService(
        openAIEndpoint,
        openAIKey,
        completionDeploymentName,
        completionModelName,
        apiVersion,
        embeddingService,
        vectorStoreService
    );

    try {
        // Create the search index if it doesn't exist
        await vectorStoreService.createIndexIfNotExists();

        // Add sample documents to the vector store
        const sampleDocuments = getSampleDocuments();
        await ragService.addDocuments(sampleDocuments);

        console.log("Sample documents added to the vector store.");
        console.log();

        // Sample queries to test the RAG system
        const sampleQueries = [
            "What is Azure OpenAI Service?",
            "How much does Azure OpenAI cost?",
            "How does the RAG pattern work?",
            "Why should I use Azure OpenAI instead of direct OpenAI API?",
            "What are some best practices for prompt engineering?"
        ];

        for (const query of sampleQueries) {
            console.log(`Query: ${query}`);
            console.log();

            try {
                const answer = await ragService.query(query);
                console.log(`Answer: ${answer}`);
            } catch (error) {
                console.error("Error:", error);
            }

            console.log("-".repeat(80));
            console.log();
        }

        // Interactive mode
        console.log("Enter your own queries (or type 'exit' to quit):");

        while (true) {
            console.log();
            const userQuery = readline.question("> ");

            if (!userQuery || userQuery.toLowerCase() === "exit") {
                break;
            }

            try {
                const answer = await ragService.query(userQuery);
                console.log();
                console.log(`Answer: ${answer}`);
            } catch (error) {
                console.error("Error:", error);
            }
        }
    } catch (error) {
        console.error("Error in main:", error);
    }
}

// Call the main function
main();
```

### 13. Run the application

```bash
npm start
```

## Expected Output

The application will:
1. Create a search index in Azure Cognitive Search (if it doesn't exist)
2. Add sample documents to the index with vector embeddings
3. Process a set of sample queries using the RAG pattern
4. Allow you to enter your own queries in interactive mode

For each query, you should see an answer that incorporates information from the relevant documents retrieved from the vector store.

## Exercises

1. Implement document chunking functionality to break large texts into smaller, more manageable pieces.
2. Add a hybrid search capability that combines vector search with traditional keyword search for better results.
3. Implement a history mechanism to maintain context across multiple user questions.
4. Create a simple Express.js API to expose the RAG functionality as a web service.
5. Add support for loading documents from files or web URLs.
6. Implement a stream response option to display the generated answer incrementally.
7. Add a feature to explain which documents were used for generating the answer.

## Troubleshooting

- **Authentication errors**: Double-check your API keys and endpoints in the `.env` file.
- **Vector search errors**: Ensure that your Azure Cognitive Search service supports vector search (requires Standard tier or above).
- **Dependency issues**: Make sure all required packages are installed with `npm install`.
- **TypeScript errors**: Use `npx tsc --noEmit` to check for TypeScript errors without generating output files.
