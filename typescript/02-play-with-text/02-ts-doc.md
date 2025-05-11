# Text Summarization and Categorization with Azure OpenAI SDK (TypeScript)

This exercise will guide you through implementing text summarization and categorization using the Azure OpenAI SDK in TypeScript.

## Prerequisites

- [Node.js](https://nodejs.org/) (v16.x or later)
- npm (included with Node.js)
- An Azure account with an Azure OpenAI resource
- Visual Studio Code (recommended)

## Setup

### 1. Create a new Node.js project

```bash
mkdir azure-openai-text-processing
cd azure-openai-text-processing
npm init -y
```

### 2. Install required dependencies

```bash
npm install openai dotenv typescript ts-node @types/node readline-sync @types/readline-sync
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

### 7. Add a script to run the application

Update the `package.json` file to add a script:

```json
{
  "scripts": {
    "start": "ts-node index.ts"
  }
}
```

### 8. Run the application

```bash
npm start
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

1. Enhance the sentiment analysis to provide a sentiment score (e.g., from -1.0 to 1.0) instead of just a label.
2. Add a function to extract named entities (people, organizations, locations) from the text.
3. Implement a text comparison function that calculates the semantic similarity between two texts.
4. Create a simple web interface using Express.js to expose the text processing functionality as a REST API.
5. Add support for processing text from URLs or files stored on disk.

## Troubleshooting

- **Authentication errors**: Double-check your API key and endpoint in the `.env` file.
- **JSON parsing errors**: Improve the error handling for cases where the model doesn't return valid JSON.
- **Token limit errors**: For long texts, consider chunking the input text into smaller segments for processing.

### 4. Create environment variables

Create a `.env` file in the project root:

```
AZURE_OPENAI_ENDPOINT=https://YOUR_AZURE_OPENAI_RESOURCE_NAME.openai.azure.com/
AZURE_OPENAI_KEY=YOUR_AZURE_OPENAI_API_KEY
AZURE_OPENAI_DEPLOYMENT_NAME=YOUR_DEPLOYMENT_NAME
AZURE_OPENAI_MODEL_NAME=YOUR_MODEL_NAME
AZURE_OPENAI_API_VERSION=2024-04-01-preview
```

### 5. Create the text processing models and service

Create a file named `textProcessingModels.ts`:

```typescript
import * as dotenv from "dotenv";
import * as readline from "readline-sync";
import { TextProcessingService } from "./textProcessingService";

// Load environment variables
export interface TextProcessingResult {
  originalText: string;
  summary: string;
  category: string;
  keywords: string[];
  sentiment: string;
}
```

Create a file named `textProcessingService.ts`:

```typescript
import { AzureOpenAI } from "openai";

export class TextProcessingService {
  private client: AzureOpenAI;
  private deploymentName: string;
  private modelName: string;

  constructor(endpoint: string, apiKey: string, deploymentName: string, modelName: string, apiVersion: string) {
    this.client = new AzureOpenAI({ endpoint, apiKey, deploymentName, apiVersion });
    this.deploymentName = deploymentName;
    this.modelName = modelName;
  }

  async summarizeText(text: string, maxTokens: number = 150): Promise<string> {
    const messages = [
      { role: "system", content: "You are a helpful AI assistant that summarizes text concisely." },
      { role: "user", content: `Please summarize the following text in about 2-3 sentences: \n\n${text}` }
    ];

    const response = await this.client.chat.completions.create({
      messages,
      max_tokens: maxTokens,
      model: this.modelName
    });

    return response.choices[0].message?.content || "";
  }

  async categorizeText(text: string, maxTokens: number = 30): Promise<string> {
    const messages = [
      { role: "system", content: "You are a helpful AI assistant that categorizes text into a single appropriate category." },
      { role: "user", content: `Please categorize the following text into exactly one category (like technology, business, politics, health, etc.). Return only the category name, nothing else.\n\n${text}` }
    ];

    const response = await this.client.chat.completions.create({
      messages,
      max_tokens: maxTokens,
      temperature: 0.0,
      model: this.modelName
    });

    return response.choices[0].message?.content?.trim() || "";
  }

  async extractKeywords(text: string, maxTokens: number = 50): Promise<string[]> {
    const messages = [
      { role: "system", content: "You are a helpful AI assistant that extracts keywords from text." },
      { role: "user", content: `Extract 5-7 important keywords from the following text. Return only a JSON array of strings, nothing else:\n\n${text}` }
    ];

    const response = await this.client.chat.completions.create({
      messages,
      max_tokens: maxTokens,
      temperature: 0.0,
      model: this.modelName
    });

    const content = response.choices[0].message?.content?.trim() || "[]";

    try {
      return JSON.parse(content);
    } catch (error) {
      // If parsing fails, try to extract keywords manually
      return content
        .replace(/[\[\]"]/g, '')
        .split(',')
        .map(k => k.trim())
        .filter(k => k.length > 0);
    }
  }

  async analyzeSentiment(text: string, maxTokens: number = 30): Promise<string> {
    const messages: ChatRequestMessage[] = [
      { role: "system", content: "You are a helpful AI assistant that analyzes sentiment." },
      { role: "user", content: `Analyze the sentiment of the following text. Respond with only one word: 'positive', 'negative', or 'neutral'.\n\n${text}` }
    ];    const response = await this.client.chat.completions.create({
      messages,
      max_tokens: maxTokens,
      temperature: 0.0,
      model: this.modelName
    });

    return response.choices[0].message?.content?.trim().toLowerCase() || "";
  }

  async processText(text: string): Promise<any> {
    // Process text in parallel for efficiency
    const [summary, category, keywords, sentiment] = await Promise.all([
      this.summarizeText(text),
      this.categorizeText(text),
      this.extractKeywords(text),
      this.analyzeSentiment(text)
    ]);

    return {
      originalText: text,
      summary,
      category,
      keywords,
      sentiment
    };
  }
}
```

### 6. Create the main application file

Create a file named `index.ts`:

```typescript
import * as dotenv from "dotenv";
import * as readline from "readline-sync";
import { TextProcessingService } from "./textProcessingService";

// Load environment variables
dotenv.config();

// Get configuration from environment variables
const endpoint = process.env.AZURE_OPENAI_ENDPOINT || "";
const apiKey = process.env.AZURE_OPENAI_KEY || "";
const deploymentName = process.env.AZURE_OPENAI_DEPLOYMENT_NAME || "";
const apiVersion = process.env.AZURE_OPENAI_API_VERSION || "";
const modelName = process.env.AZURE_OPENAI_MODEL_NAME || "";

if (!endpoint || !apiKey || !deploymentName || !apiVersion || !modelName) {
  throw new Error("Missing required environment variables");
}

// Sample texts for processing
const sampleTexts = [
  "The European Union has approved new regulations on artificial intelligence that will require AI companies to disclose training data and conduct risk assessments. The regulations, which focus on high-risk AI applications, aim to ensure AI systems are safe, transparent, and respect fundamental rights while still promoting innovation. Critics argue the rules may stifle growth in the industry, while proponents say they'll help build trust in AI technology.",

  "A new study published in Nature Medicine suggests that regular moderate exercise may reduce the risk of severe COVID-19 outcomes. Researchers analyzed data from over 50,000 patients and found that those who engaged in at least 150 minutes of moderate activity per week had 31% lower odds of hospitalization when infected with COVID-19. The study controlled for age, BMI, and pre-existing conditions.",

  "The latest smartphone from Apple features a groundbreaking camera system with a 108-megapixel sensor, allowing for unprecedented detail in photos even in low light conditions. The phone also includes a new AI-powered image processing system that can recognize objects in real-time and adjust settings accordingly. Battery life has been improved by 25% compared to the previous model."
];

async function main() {  // Initialize the text processing service
  const textProcessingService = new TextProcessingService(endpoint, apiKey, deploymentName, modelName, apiVersion);

  console.log("Processing sample texts...\n");

  // Process each sample text
  for (const text of sampleTexts) {
    console.log("Original Text:");
    console.log(text);
    console.log();

    try {
      const result = await textProcessingService.processText(text);

      console.log("Summary:");
      console.log(result.summary);
      console.log();

      console.log("Category:");
      console.log(result.category);
      console.log();

      console.log("Keywords:");
      console.log(result.keywords.join(", "));
      console.log();

      console.log("Sentiment:");
      console.log(result.sentiment);
      console.log();
    } catch (error) {
      console.error("Error processing text:", error);
    }

    console.log("-".repeat(80));
    console.log();
  }

  // Interactive mode
  console.log("Enter your own text to process (or type 'exit' to quit):");

  while (true) {
    console.log();
    const userText = readline.question("> ");

    if (!userText || userText.toLowerCase() === "exit") {
      break;
    }

    try {
      const result = await textProcessingService.processText(userText);

      console.log("Summary:");
      console.log(result.summary);
      console.log();

      console.log("Category:");
      console.log(result.category);
      console.log();

      console.log("Keywords:");
      console.log(result.keywords.join(", "));
      console.log();

      console.log("Sentiment:");
      console.log(result.sentiment);
      console.log();
    } catch (error) {
      console.error("Error processing text:", error);
    }
  }
}

// Call the main function
main().catch((error) => {
  console.error("Error in main:", error);
});

```