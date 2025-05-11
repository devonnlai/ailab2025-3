# Hello World with Azure OpenAI SDK (TypeScript)

This exercise will guide you through setting up a basic "Hello World" application using the Azure OpenAI SDK in TypeScript.

## Prerequisites

- [Node.js](https://nodejs.org/) (v16.x or later)
- npm (included with Node.js)
- An Azure account with an Azure OpenAI resource
- Visual Studio Code (recommended)

## Setup

### 1. Create a new Node.js project

```bash
mkdir azure-openai-ts-hello
cd azure-openai-ts-hello
npm init -y
```

### 2. Install required dependencies

```bash
npm install openai dotenv typescript ts-node @types/node @azure/core-auth @azure/identity
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
AZURE_OPENAI_ENDPOINT=https://YOUR_AZURE_OPENAI_RESOURCE_NAME.openai.azure.com/
AZURE_OPENAI_KEY=YOUR_AZURE_OPENAI_API_KEY
AZURE_OPENAI_DEPLOYMENT_NAME=YOUR_DEPLOYMENT_NAME
AZURE_OPENAI_MODEL_NAME=YOUR_MODEL_NAME
AZURE_OPENAI_API_VERSION=2024-04-01-preview
```

### 5. Create the main application file

Create a file named `index.ts` with the following content:

```typescript
import { AzureOpenAI } from "openai";
import * as dotenv from "dotenv";

// Load environment variables
dotenv.config();

async function main() {
  // Get configuration from environment variables
  const endpoint = process.env.AZURE_OPENAI_ENDPOINT || "";
  const apiKey = process.env.AZURE_OPENAI_KEY || "";
  const deploymentName = process.env.AZURE_OPENAI_DEPLOYMENT_NAME || "";
  const apiVersion = process.env.AZURE_OPENAI_API_VERSION || "";
  const modelName = process.env.AZURE_OPENAI_MODEL_NAME || "";

  if (!endpoint || !apiKey || !deploymentName || !apiVersion || !modelName) {
    throw new Error("Missing required environment variables");
  }

  const options = { endpoint, apiKey, deploymentName, apiVersion };

  // Initialize the Azure OpenAI client
  const client = new AzureOpenAI(options);

  try {
    // Send a completion request
    const response = await client.chat.completions.create({
      messages: [
        { role: "system", content: "You are a helpful AI assistant." },
        { role: "user", content: "Hello, world! What can you do for me?" }
      ],
      max_tokens: 4096,
      temperature: 1,
      top_p: 1,
      model: modelName
    });

    // Display the response
    console.log("Azure OpenAI Response:\n");
    console.log(response.choices[0].message?.content || "No response");
  } catch (error) {
    console.error("Error:", error);
  }
}

// Call the main function
main().catch((error) => {
  console.error("Error in main:", error);
});
```

### 6. Add a script to run the application

Update the `package.json` file to add a script:

```json
{
  "scripts": {
    "start": "ts-node index.ts"
  }
}
```

### 7. Run the application

```bash
npm start
```

## Expected Output

You should see a response from the Azure OpenAI service to your "Hello, world!" message, similar to:

```
Azure OpenAI Response:

Hello! I'm an AI assistant, and I'm here to help you with a variety of tasks. I can answer questions, provide information on various topics, assist with writing tasks, brainstorm ideas, explain concepts, and more. Feel free to ask me anything, and I'll do my best to assist you!
```

## Exercises

1. Try modifying the system message to give the AI a different personality or role.
2. Create a simple CLI chat interface that allows for multiple conversation turns.
3. Experiment with different parameters like temperature and max tokens to see how they affect the model's responses.
4. Add error handling for different types of API errors.

## Troubleshooting

- **Authentication errors**: Double-check your API key and endpoint in the `.env` file.
- **Deployment errors**: Make sure you've created a deployment for one of the models (e.g., gpt-4o-mini, gpt-4, gpt-35-turbo) in the Azure OpenAI service.
- **Module not found errors**: Run `npm install` to ensure all packages are installed.
- **API version errors**: Verify that the API version is correctly set (e.g., '2024-04-01-preview'). Different features may require specific API versions.
- **Model name errors**: Ensure the model name matches exactly with the model name in your Azure OpenAI resource.