# Deriving Insights from CSV Data using Azure OpenAI SDK (TypeScript)

This exercise will guide you through using the Azure OpenAI SDK to analyze and derive insights from CSV data in a TypeScript application.

## Prerequisites

- [Node.js](https://nodejs.org/) (v16.x or later)
- npm (included with Node.js)
- An Azure account with an Azure OpenAI resource
- Visual Studio Code (recommended)

## Setup

### 1. Create a new Node.js project

```bash
mkdir azure-openai-csv-insights
cd azure-openai-csv-insights
npm init -y
```

### 2. Install required dependencies

```bash
npm install openai dotenv typescript ts-node @types/node readline-sync @types/readline-sync csv-parser fs-extra @types/fs-extra
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
    "skipLibCheck": true,
    "resolveJsonModule": true
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

### 5. Create a sample CSV file

Create a folder called `data` and add a file called `sales_data.csv` with the following content:

```csv
Date,Region,Product,Category,Sales,Units,CustomerType
2023-01-05,East,Laptop X1,Electronics,1200.50,2,Business
2023-01-12,West,Smartphone Y2,Electronics,899.99,3,Consumer
2023-01-15,North,Office Chair,Furniture,349.95,5,Business
2023-01-18,South,Desk Lamp,Furniture,75.00,10,Consumer
2023-01-20,East,Monitor Z3,Electronics,450.00,3,Business
2023-01-22,West,Coffee Maker,Appliances,89.99,7,Consumer
2023-01-25,North,Bluetooth Speaker,Electronics,129.95,6,Consumer
2023-01-28,South,Standing Desk,Furniture,799.00,2,Business
2023-02-02,East,Tablet A1,Electronics,599.99,4,Consumer
2023-02-05,West,Wireless Mouse,Electronics,45.99,15,Consumer
2023-02-08,North,Ergonomic Keyboard,Electronics,149.99,6,Business
2023-02-10,South,Air Purifier,Appliances,279.95,3,Consumer
2023-02-15,East,Laptop X1,Electronics,1200.50,3,Business
2023-02-18,West,Smartphone Y2,Electronics,899.99,5,Consumer
2023-02-20,North,Bookshelf,Furniture,210.50,4,Consumer
2023-02-22,South,Desk Lamp,Furniture,75.00,8,Consumer
2023-02-25,East,Monitor Z3,Electronics,450.00,2,Business
2023-02-28,West,Microwave Oven,Appliances,189.95,3,Consumer
2023-03-03,North,Bluetooth Speaker,Electronics,129.95,10,Consumer
2023-03-06,South,Standing Desk,Furniture,799.00,1,Business
2023-03-10,East,Tablet A1,Electronics,599.99,6,Consumer
2023-03-12,West,Wireless Mouse,Electronics,45.99,20,Business
2023-03-15,North,Ergonomic Keyboard,Electronics,149.99,8,Business
2023-03-18,South,Air Purifier,Appliances,279.95,4,Consumer
2023-03-20,East,Laptop X1,Electronics,1200.50,4,Consumer
2023-03-23,West,Smartphone Y2,Electronics,899.99,7,Consumer
2023-03-25,North,Office Chair,Furniture,349.95,6,Business
2023-03-28,South,Coffee Maker,Appliances,89.99,15,Consumer
```

### 6. Create the data models

Create a file named `models/salesRecord.ts`:

```typescript
export interface SalesRecord {
  Date: string;
  Region: string;
  Product: string;
  Category: string;
  Sales: number;
  Units: number;
  CustomerType: string;
}
```

### 7. Create a CSV data service

Create a file named `services/csvDataService.ts`:

```typescript
import * as fs from 'fs';
import * as path from 'path';
import * as csv from 'csv-parser';

export class CsvDataService {
  async readCsvFile<T>(filePath: string): Promise<T[]> {
    return new Promise((resolve, reject) => {
      const results: T[] = [];

      fs.createReadStream(filePath)
        .pipe(csv())
        .on('data', (data) => {
          // Convert numeric fields
          Object.keys(data).forEach(key => {
            if (!isNaN(Number(data[key]))) {
              data[key] = Number(data[key]);
            }
          });
          results.push(data as T);
        })
        .on('end', () => {
          resolve(results);
        })
        .on('error', (error) => {
          reject(error);
        });
    });
  }

  getCsvSummary<T>(records: T[]): string {
    if (!records || records.length === 0) {
      return "No data available";
    }

    const firstRecord = records[0];
    const columns = Object.keys(firstRecord);

    let summary = `Total records: ${records.length}\n`;
    summary += "Columns:\n";

    columns.forEach(column => {
      const type = typeof firstRecord[column as keyof T];
      summary += `- ${column} (${type})\n`;
    });

    return summary;
  }

  convertRecordsToString<T>(records: T[], limit: number = 10): string {
    if (!records || records.length === 0) {
      return "No data available";
    }

    const columns = Object.keys(records[0]);
    const recordsToShow = records.slice(0, limit);

    let result = columns.join(',') + '\n';

    recordsToShow.forEach(record => {
      const values = columns.map(column => {
        const value = record[column as keyof T];
        return value !== undefined ? value.toString() : '';
      });
      result += values.join(',') + '\n';
    });

    return result;
  }
}
```

### 8. Create a data analytics service

Create a file named `services/dataAnalyticsService.ts`:

```typescript
import { AzureOpenAI } from "openai";
import { CsvDataService } from "./csvDataService";

export class DataAnalyticsService {
  private client: AzureOpenAI;
  private modelName: string;
  private csvDataService: CsvDataService;

  constructor(endpoint: string, apiKey: string, deploymentName: string, modelName: string, apiVersion: string) {
    this.client = new AzureOpenAI({
      endpoint,
      apiKey,
      deploymentName,
      apiVersion
    });
    this.modelName = modelName;
    this.csvDataService = new CsvDataService();
  }

  async generateDataSummary<T>(data: T[], dataDescription: string): Promise<string> {
    const dataString = this.csvDataService.convertRecordsToString(data, 25); // Show up to 25 records
    const dataSummary = this.csvDataService.getCsvSummary(data);

    const prompt = `
I have a dataset with the following summary:
${dataSummary}

Here's a sample of the data:
${dataString}

Please provide a comprehensive summary of this ${dataDescription} data, including:
1. Overall data description
2. Key observations
3. Any notable patterns or trends

Keep your response focused on factual observations from the data.
`;

    const messages = [
      { role: "system", content: "You are a helpful data analyst assistant that provides insights from data." },
      { role: "user", content: prompt }
    ];

    const response = await this.client.chat.completions.create({
      messages,
      temperature: 0,
      max_tokens: 1000,
      model: this.modelName
    });

    return response.choices[0].message?.content || "No response generated.";
  }

  async analyzeData<T>(data: T[], question: string): Promise<string> {
    const dataString = this.csvDataService.convertRecordsToString(data, 25); // Show up to 25 records
    const dataSummary = this.csvDataService.getCsvSummary(data);

    const prompt = `
I have a dataset with the following summary:
${dataSummary}

Here's a sample of the data:
${dataString}

Question: ${question}

Please analyze the data to answer this question. If calculations are needed, explain your methodology clearly. If the data is insufficient to answer the question completely, note what additional data would be helpful.
`;

    const messages: ChatRequestMessage[] = [
      { role: "system", content: "You are a helpful data analyst assistant that provides insights from data." },
      { role: "user", content: prompt }
    ];

    const response = await this.client.chat.completions.create({
      messages,
      temperature: 0,
      max_tokens: 1000,
      model: this.modelName
    });

    return response.choices[0].message?.content || "No response generated.";
  }

  async generateVisualizationCode<T>(data: T[], visualizationType: string, description: string): Promise<string> {
    const dataString = this.csvDataService.convertRecordsToString(data, 25); // Show up to 25 records
    const dataSummary = this.csvDataService.getCsvSummary(data);

    const prompt = `
I have a dataset with the following summary:
${dataSummary}

Here's a sample of the data:
${dataString}

I want to create a ${visualizationType} visualization that shows ${description}.

Please provide TypeScript code that would create this visualization using a library like Chart.js, D3.js, or similar. The code should be complete and ready to run. Include any necessary package references.

Also, explain why this visualization is appropriate for the data and what insights it might reveal.
`;    const messages = [
      { role: "system", content: "You are a helpful data visualization expert that provides data visualization code." },
      { role: "user", content: prompt }
    ];

    const response = await this.client.chat.completions.create({
      messages,
      temperature: 0,
      max_tokens: 1500,
      model: this.modelName
    });

    return response.choices[0].message?.content || "No response generated.";
  }

  async getDataStatistics<T>(data: T[]): Promise<Record<string, any>> {
    const dataString = this.csvDataService.convertRecordsToString(data, 25); // Show up to 25 records
    const dataSummary = this.csvDataService.getCsvSummary(data);

    const prompt = `
I have a dataset with the following summary:
${dataSummary}

Here's a sample of the data:
${dataString}

Calculate and provide the following statistical metrics for the numerical columns in this dataset:
1. Min, Max, Mean, and Median values
2. Standard deviation
3. Total and sum where appropriate

Format your response as a valid JSON object where the keys are the column names and the values are objects with the calculated statistics. Round numerical values to 2 decimal places.
`;

    const messages: ChatRequestMessage[] = [
      { role: "system", content: "You are a helpful data analyst assistant that provides data statistics in JSON format." },
      { role: "user", content: prompt }
    ];    const response = await this.client.chat.completions.create({
      messages,
      temperature: 0,
      max_tokens: 1000,
      model: this.modelName
    });

    const content = response.choices[0].message?.content || "{}";

    try {
      // Extract JSON from the response (in case there's explanatory text)
      const jsonStart = content.indexOf('{');
      const jsonEnd = content.lastIndexOf('}');

      if (jsonStart >= 0 && jsonEnd >= 0) {
        const jsonContent = content.substring(jsonStart, jsonEnd + 1);
        return JSON.parse(jsonContent);
      }
    } catch (err) {
      console.error('Error parsing JSON response:', err);
    }

    return {};
  }
}
```

### 9. Create the main application file

Create a file named `index.ts`:

```typescript
import * as dotenv from 'dotenv';
import * as path from 'path';
import * as readline from 'readline-sync';
import { CsvDataService } from './services/csvDataService';
import { DataAnalyticsService } from './services/dataAnalyticsService';
import { SalesRecord } from './models/salesRecord';

// Load environment variables
dotenv.config();

// Get configuration from environment variables
const openAIEndpoint = process.env.AZURE_OPENAI_ENDPOINT || "";
const openAIKey = process.env.AZURE_OPENAI_KEY || "";
const deploymentName = process.env.AZURE_OPENAI_DEPLOYMENT_NAME || "";
const apiVersion = process.env.AZURE_OPENAI_API_VERSION || "";
const modelName = process.env.AZURE_OPENAI_MODEL_NAME || "";

if (!openAIEndpoint || !openAIKey || !deploymentName || !apiVersion || !modelName) {
  throw new Error("Missing required environment variables");
}

async function main() {  // Initialize services
  const csvDataService = new CsvDataService();
  const dataAnalyticsService = new DataAnalyticsService(openAIEndpoint, openAIKey, deploymentName, modelName, apiVersion);

  // Load the CSV data
  const dataFilePath = path.join(__dirname, 'data', 'sales_data.csv');
  console.log(`Loading data from ${dataFilePath}`);

  let salesData: SalesRecord[];
  try {
    salesData = await csvDataService.readCsvFile<SalesRecord>(dataFilePath);
    console.log(`Loaded ${salesData.length} records from the CSV file.`);
    console.log();
  } catch (err) {
    console.error('Error loading CSV data:', err);
    return;
  }

  // Main menu loop
  while (true) {
    console.log("\n=== Sales Data Analysis ===");
    console.log("1. View Data Summary");
    console.log("2. Analyze Sales by Region");
    console.log("3. Analyze Sales by Category");
    console.log("4. Analyze Sales by Customer Type");
    console.log("5. Generate Visualization Code");
    console.log("6. Get Data Statistics");
    console.log("7. Ask a Custom Question");
    console.log("8. Exit");
    console.log();

    const choice = readline.questionInt("Select an option: ");

    try {
      switch (choice) {
        case 1: // Data Summary
          console.log("\nGenerating data summary...");
          const summary = await dataAnalyticsService.generateDataSummary(salesData, "sales");
          console.log("\n=== Data Summary ===");
          console.log(summary);
          break;

        case 2: // Sales by Region
          console.log("\nAnalyzing sales by region...");
          const regionAnalysis = await dataAnalyticsService.analyzeData(
            salesData,
            "What are the total sales and units sold by region? Which region is performing the best, and are there any trends over time?"
          );
          console.log("\n=== Sales by Region Analysis ===");
          console.log(regionAnalysis);
          break;

        case 3: // Sales by Category
          console.log("\nAnalyzing sales by category...");
          const categoryAnalysis = await dataAnalyticsService.analyzeData(
            salesData,
            "What are the total sales and units sold by product category? Which category is the most profitable, and which has the highest volume of units sold?"
          );
          console.log("\n=== Sales by Category Analysis ===");
          console.log(categoryAnalysis);
          break;

        case 4: // Sales by Customer Type
          console.log("\nAnalyzing sales by customer type...");
          const customerAnalysis = await dataAnalyticsService.analyzeData(
            salesData,
            "How do sales differ between Business and Consumer customer types? Which customer type generates more revenue, and which buys more units?"
          );
          console.log("\n=== Sales by Customer Type Analysis ===");
          console.log(customerAnalysis);
          break;

        case 5: // Generate Visualization Code
          console.log("\nWhat type of visualization would you like?");
          console.log("1. Sales by Region (Bar Chart)");
          console.log("2. Sales by Category (Pie Chart)");
          console.log("3. Sales Trend Over Time (Line Chart)");
          console.log("4. Custom Visualization");
          console.log();

          const vizChoice = readline.questionInt("Select an option: ");

          let vizType = "";
          let vizDescription = "";

          switch (vizChoice) {
            case 1:
              vizType = "bar chart";
              vizDescription = "total sales by region";
              break;
            case 2:
              vizType = "pie chart";
              vizDescription = "sales distribution by product category";
              break;
            case 3:
              vizType = "line chart";
              vizDescription = "sales trend over time (by month)";
              break;
            case 4:
              vizType = readline.question("\nEnter visualization type (e.g., bar chart, pie chart, line chart): ");
              vizDescription = readline.question("Enter what the visualization should show: ");
              break;
            default:
              console.log("Invalid choice. Using default bar chart.");
              vizType = "bar chart";
              vizDescription = "total sales by region";
              break;
          }

          console.log(`\nGenerating ${vizType} code for ${vizDescription}...`);
          const vizCode = await dataAnalyticsService.generateVisualizationCode(
            salesData,
            vizType,
            vizDescription
          );
          console.log("\n=== Visualization Code ===");
          console.log(vizCode);
          break;

        case 6: // Data Statistics
          console.log("\nCalculating data statistics...");
          const statistics = await dataAnalyticsService.getDataStatistics(salesData);
          console.log("\n=== Data Statistics ===");

          for (const [key, value] of Object.entries(statistics)) {
            console.log(`${key}:`);

            if (typeof value === 'object' && value !== null) {
              for (const [statKey, statValue] of Object.entries(value)) {
                console.log(`  ${statKey}: ${statValue}`);
              }
            } else {
              console.log(`  ${value}`);
            }

            console.log();
          }
          break;

        case 7: // Custom Question
          const customQuestion = readline.question("\nEnter your question about the sales data: ");

          if (!customQuestion) {
            console.log("Question cannot be empty.");
            break;
          }

          console.log("\nAnalyzing data to answer your question...");
          const customAnswer = await dataAnalyticsService.analyzeData(salesData, customQuestion);
          console.log("\n=== Analysis Results ===");
          console.log(customAnswer);
          break;

        case 8: // Exit
          console.log("\nExiting program. Goodbye!");
          return;

        default:
          console.log("\nInvalid option. Please select a number between 1 and 8.");
          break;
      }
    } catch (err) {
      console.error("\nError:", err);
    }

    readline.question("\nPress Enter to continue...");
    console.clear();
  }
}

// Call the main function
main().catch(err => {
  console.error("Error in main:", err);
});
```

### 10. Update package.json

Update the `package.json` file to add scripts:

```json
{
  "scripts": {
    "start": "ts-node index.ts",
    "build": "tsc"
  }
}
```

### 11. Set up the directory structure

Ensure your project structure looks like this:

```
azure-openai-csv-insights/
├── .env
├── package.json
├── tsconfig.json
├── index.ts
├── models/
│   └── salesRecord.ts
├── services/
│   ├── csvDataService.ts
│   └── dataAnalyticsService.ts
└── data/
    └── sales_data.csv
```

### 12. Run the application

```bash
npm start
```

## Expected Output

When you run the application, you'll see a menu with options to:

1. View a summary of the sales data
2. Analyze sales by region
3. Analyze sales by category
4. Analyze sales by customer type
5. Generate visualization code for different chart types
6. Get detailed statistics about the data
7. Ask custom questions about the data
8. Exit the application

For each option, the application uses the Azure OpenAI service to analyze the data and provide insights or generate code for visualizations.

## Example Insights

Here are some examples of the types of insights the system might provide:

### Sales by Region Analysis

The analysis reveals that the East region leads in total sales with $4,051.48, followed by West ($3,181.89), North ($1,470.23), and South ($1,318.89). East region not only has the highest total sales but also maintains consistent performance across all three months. The West region shows strong growth over time, with sales increasing each month. North and South regions have lower overall sales but show different patterns - North has more consistent monthly sales while South shows more variability.

### Sales by Category Analysis

Electronics is the dominant category with $9,323.32 in total sales (84% of all sales), followed by Furniture with $1,834.40 (16.5%) and Appliances with $649.88 (5.9%). While Electronics leads in revenue, it's interesting to note that Furniture items have a higher average price per unit. The Appliances category has the lowest total sales but a higher volume of units sold compared to Furniture.

## Example Visualization Code

The following is an example of the visualization code that might be generated for a bar chart showing sales by region:

```typescript
import * as Chart from 'chart.js/auto';
import { CategoryScale, LinearScale, BarController, BarElement, Title, Tooltip, Legend } from 'chart.js';

// Register required components
Chart.register(CategoryScale, LinearScale, BarController, BarElement, Title, Tooltip, Legend);

// Function to create a bar chart showing total sales by region
function createSalesByRegionChart(salesData: any[]): void {
  // Process the data to get total sales by region
  const salesByRegion = salesData.reduce((acc, current) => {
    const region = current.Region;
    if (!acc[region]) {
      acc[region] = 0;
    }
    acc[region] += Number(current.Sales);
    return acc;
  }, {});

  // Prepare data for the chart
  const regions = Object.keys(salesByRegion);
  const sales = regions.map(region => salesByRegion[region]);

  // Create a canvas element to render the chart
  const canvas = document.createElement('canvas');
  canvas.id = 'salesByRegionChart';
  canvas.width = 800;
  canvas.height = 400;
  document.body.appendChild(canvas);

  // Create the chart
  const ctx = canvas.getContext('2d');
  if (ctx) {
    new Chart(ctx, {
      type: 'bar',
      data: {
        labels: regions,
        datasets: [{
          label: 'Total Sales ($)',
          data: sales,
          backgroundColor: [
            'rgba(255, 99, 132, 0.6)',
            'rgba(54, 162, 235, 0.6)',
            'rgba(255, 206, 86, 0.6)',
            'rgba(75, 192, 192, 0.6)'
          ],
          borderColor: [
            'rgba(255, 99, 132, 1)',
            'rgba(54, 162, 235, 1)',
            'rgba(255, 206, 86, 1)',
            'rgba(75, 192, 192, 1)'
          ],
          borderWidth: 1
        }]
      },
      options: {
        responsive: true,
        plugins: {
          legend: {
            position: 'top',
          },
          title: {
            display: true,
            text: 'Total Sales by Region'
          },
          tooltip: {
            callbacks: {
              label: function(context) {
                return `${context.parsed.y.toFixed(2)}`;
              }
            }
          }
        },
        scales: {
          y: {
            beginAtZero: true,
            ticks: {
              callback: function(value) {
                return ' + value;
              }
            }
          }
        }
      }
    });
  }
}

// Call the function with your sales data
createSalesByRegionChart(salesData);

/*
This bar chart visualization is appropriate for showing total sales by region because:
1. Bar charts excel at comparing categorical data (regions) across a quantitative measure (sales)
2. The vertical bars make it easy to visually compare the performance of different regions
3. The clear labeling and color coding helps to distinguish between regions
4. The chart reveals which regions are contributing most to overall sales

From this visualization, we can quickly identify which regions are the top performers and which
may need additional support or marketing efforts to improve sales.
*/
```

## Exercises

1. Extend the application to support multiple data files or different data formats.
2. Implement a feature to export the generated insights to a file.
3. Add functionality to perform time series analysis or forecasting.
4. Create a web interface using Express.js to visualize the data and insights.
5. Implement a function to automatically detect anomalies or outliers in the data.
6. Add support for loading data from external sources like databases or APIs.
7. Implement a caching mechanism to avoid repeated API calls for the same analysis.

## Troubleshooting

- **CSV parsing errors**: Ensure your CSV file format is correct and the column names match what's expected in the `SalesRecord` interface.
- **Authentication errors**: Double-check your API key and endpoint in the `.env` file.
- **TypeScript errors**: Run `npx tsc --noEmit` to check for TypeScript errors without generating output files.
- **JSON parsing errors**: If the model doesn't return properly formatted JSON, improve the error handling in the `getDataStatistics` method.