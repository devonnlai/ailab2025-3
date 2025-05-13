# Deriving Insights from CSV Data using Azure OpenAI SDK (.NET)

This exercise will guide you through using the Azure OpenAI SDK to analyze and derive insights from CSV data in a .NET application.

## Prerequisites

- [.NET 6.0 SDK](https://dotnet.microsoft.com/download/dotnet/6.0) or later
- An Azure account with an Azure OpenAI resource
- Visual Studio 2022 or Visual Studio Code

## Setup

### 1. Create a new .NET Console Application

```bash
dotnet new console -n AzureOpenAIDataInsights
cd AzureOpenAIDataInsights
```

### 2. Add the required NuGet packages

```bash
dotnet add package Azure.AI.OpenAI --version 2.1.0
dotnet add package OpenAI --version 2.1.0
dotnet add package Microsoft.Extensions.Configuration.Json --version 9.0.4
dotnet add package CsvHelper --version 33.0.0
```

### 3. Create configuration settings

Create an `appsettings.json` file in your project root:

```json
{
  "AzureOpenAI": {
    "Endpoint": "https://YOUR_AZURE_OPENAI_RESOURCE_NAME.services.ai.azure.com/",
    "Key": "YOUR_AZURE_OPENAI_API_KEY",
    "DeploymentName": "YOUR_DEPLOYMENT_NAME"
  }
}
```

### 9. Add the CSV file to the project

Create a folder called `Data` in your project directory and add the `sales_data.csv` file there. Make sure to set the file properties to copy to the output directory:

```xml
<ItemGroup>
  <None Update="Data\sales_data.csv">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </None>
</ItemGroup>
```

### 10. Build and run the application

```bash
dotnet build
dotnet run
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

## Exercises

1. Extend the application to support multiple data files or different data formats.
2. Implement a feature to save the generated insights to a file.
3. Add a function to perform more complex analyses, such as correlation between different variables.
4. Create a feature to automatically generate and display visualizations instead of just providing the code.
5. Implement a function to make predictions based on the data (e.g., sales forecasting).
6. Add support for dynamic data loading from a database or API endpoint.
7. Create a simple web API to expose the data analysis capabilities.

## Troubleshooting

- **CSV parsing errors**: Check the format of your CSV file and ensure the column names match the property names in the `SalesRecord` class.
- **Authentication errors**: Double-check your API key and endpoint in `appsettings.json`.
- **Token limit errors**: If your data is very large, you may need to sample or summarize it before sending it to the Azure OpenAI service.
- **Invalid JSON errors**: The JSON parsing in the `GetDataStatisticsAsync` method might fail if the model doesn't return properly formatted JSON.

### 4. Create a sample CSV file

Create a folder called `Data` and add a file called `sales_data.csv` with the following content:

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

### 5. Create the data models

Create a new file called `SalesRecord.cs`:

```csharp
using AzureOpenAIDataInsights;
using Microsoft.Extensions.Configuration;
using System.Globalization;
using System.Text;
using System.Text.Json;

// Set culture for consistent number formatting
using System.Globalization;
using CsvHelper.Configuration.Attributes;

namespace AzureOpenAIDataInsights
{
    public class SalesRecord
    {
        [Name("Date")]
        public DateTime Date { get; set; }

        [Name("Region")]
        public string Region { get; set; } = "";

        [Name("Product")]
        public string Product { get; set; } = "";

        [Name("Category")]
        public string Category { get; set; } = "";

        [Name("Sales")]
        public decimal Sales { get; set; }

        [Name("Units")]
        public int Units { get; set; }

        [Name("CustomerType")]
        public string CustomerType { get; set; } = "";
    }
}
```

### 6. Create a CSV data service

Create a new file called `CsvDataService.cs`:

```csharp
using System.Globalization;
using System.Text;
using CsvHelper;
using CsvHelper.Configuration;

namespace AzureOpenAIDataInsights
{
  public class CsvDataService
  {
    public List<T> ReadCsvFile<T>(string filePath)
    {
      var config = new CsvConfiguration(CultureInfo.InvariantCulture)
      {
        HeaderValidated = null,
        MissingFieldFound = null,
      };

      using var streamReader = new StreamReader(filePath);
      using var csvReader = new CsvReader(streamReader, config);

      return csvReader.GetRecords<T>().ToList();
    }

    public string GetCsvSummary<T>(List<T> records)
    {
      if (records == null || !records.Any())
        return "No data available";

      var firstRecord = records.First();
      var properties = typeof(T).GetProperties();

      var summary = new StringBuilder();
      summary.AppendLine($"Total records: {records.Count}");
      summary.AppendLine("Columns:");

      foreach (var property in properties)
      {
        summary.AppendLine($"- {property.Name} ({property.PropertyType.Name})");
      }

      return summary.ToString();
    }

    public string ConvertRecordsToString<T>(List<T> records, int limit = 10)
    {
      if (records == null || !records.Any())
        return "No data available";

      var properties = typeof(T).GetProperties();
      var recordsToShow = records.Take(limit).ToList();

      var csv = new StringBuilder();

      // Add header
      csv.AppendLine(string.Join(",", properties.Select(p => p.Name)));

      // Add records
      foreach (var record in recordsToShow)
      {
        var values = properties.Select(p =>
        {
          var value = p.GetValue(record);
          return value?.ToString() ?? "";
        });
        csv.AppendLine(string.Join(",", values));
      }

      return csv.ToString();
    }
  }
}
```

### 7. Create a data analytics service

Create a new file called `DataAnalyticsService.cs`:

```csharp
using Azure;
using Azure.AI.OpenAI;
using System.Text;
using System.Text.Json;
using OpenAI.Chat;

namespace AzureOpenAIDataInsights
{
    public class DataAnalyticsService
    {
        private readonly AzureOpenAIClient _openAIClient;
        private readonly string _deploymentName;

        public DataAnalyticsService(string endpoint, string key, string deploymentName)
        {
            _openAIClient = new AzureOpenAIClient(new Uri(endpoint), new AzureKeyCredential(key));
            _deploymentName = deploymentName;
        }        

        public async Task<string> GenerateDataSummaryAsync<T>(List<T> data, string dataDescription)
        {
            var csvDataService = new CsvDataService();
            var dataString = csvDataService.ConvertRecordsToString(data, 25); // Show up to 25 records
            var dataSummary = csvDataService.GetCsvSummary(data);

            var prompt = $@"
I have a dataset with the following summary:
{dataSummary}

Here's a sample of the data:
{dataString}

Please provide a comprehensive summary of this {dataDescription} data, including:
1. Overall data description
2. Key observations
3. Any notable patterns or trends

Keep your response focused on factual observations from the data.
";

            ChatClient chatClient = _openAIClient.GetChatClient(_deploymentName);

            var requestOptions = new ChatCompletionOptions()
            {
              MaxOutputTokenCount = 1000,
              Temperature = 0.0f,
              TopP = 1.0f,
            };

            List<ChatMessage> messages =
            [
              new SystemChatMessage("You are a helpful data analyst assistant that provides insights from data."),
              new UserChatMessage(prompt)
            ];

            var response = await chatClient.CompleteChatAsync(messages, requestOptions);
            var completion = response.Value;

            return completion.Content[0].Text;
        }

        public async Task<string> AnalyzeDataAsync<T>(List<T> data, string question)
        {
            var csvDataService = new CsvDataService();
            var dataString = csvDataService.ConvertRecordsToString(data, 25); // Show up to 25 records
            var dataSummary = csvDataService.GetCsvSummary(data);

            var prompt = $@"
I have a dataset with the following summary:
{dataSummary}

Here's a sample of the data:
{dataString}

Question: {question}

Please analyze the data to answer this question. If calculations are needed, explain your methodology clearly. If the data is insufficient to answer the question completely, note what additional data would be helpful.
";            
            
            ChatClient chatClient = _openAIClient.GetChatClient(_deploymentName);

            var requestOptions = new ChatCompletionOptions()
            {
              MaxOutputTokenCount = 1000,
              Temperature = 0.0f,
              TopP = 1.0f,
            };

            List<ChatMessage> messages =
            [
              new SystemChatMessage("You are a helpful data analyst assistant that provides insights from data."),
              new UserChatMessage(prompt)
            ];

            var response = await chatClient.CompleteChatAsync(messages, requestOptions);
            var completion = response.Value;

            return completion.Content[0].Text;
        }

        public async Task<string> GenerateVisualizationCodeAsync<T>(List<T> data, string visualizationType, string description)
        {
            var csvDataService = new CsvDataService();
            var dataString = csvDataService.ConvertRecordsToString(data, 25); // Show up to 25 records
            var dataSummary = csvDataService.GetCsvSummary(data);

            var prompt = $@"
I have a dataset with the following summary:
{dataSummary}

Here's a sample of the data:
{dataString}

I want to create a {visualizationType} visualization that shows {description}.

Please provide C# code that would create this visualization using a library like OxyPlot, ScottPlot, or even a simple approach that outputs to the console. The code should be complete and ready to run. Include any necessary package references.

Also, explain why this visualization is appropriate for the data and what insights it might reveal.
";            
            
            ChatClient chatClient = _openAIClient.GetChatClient(_deploymentName);

            var requestOptions = new ChatCompletionOptions()
            {
              MaxOutputTokenCount = 1500,
              Temperature = 0.0f,
              TopP = 1.0f,
            };

            List<ChatMessage> messages =
            [
              new SystemChatMessage("You are a helpful data visualization expert that provides data visualization code."),
              new UserChatMessage(prompt)
            ];

            var response = await chatClient.CompleteChatAsync(messages, requestOptions);
            var completion = response.Value;

            return completion.Content[0].Text;
        }

        public async Task<Dictionary<string, object>> GetDataStatisticsAsync<T>(List<T> data)
        {
            var csvDataService = new CsvDataService();
            var dataString = csvDataService.ConvertRecordsToString(data, 25); // Show up to 25 records
            var dataSummary = csvDataService.GetCsvSummary(data);

            var prompt = $@"
I have a dataset with the following summary:
{dataSummary}

Here's a sample of the data:
{dataString}

Calculate and provide the following statistical metrics for the numerical columns in this dataset:
1. Min, Max, Mean, and Median values
2. Standard deviation
3. Total and sum where appropriate

Format your response as a valid JSON object where the keys are the column names and the values are objects with the calculated statistics. Round numerical values to 2 decimal places.
";            
            
            ChatClient chatClient = _openAIClient.GetChatClient(_deploymentName);

            var requestOptions = new ChatCompletionOptions()
            {
              MaxOutputTokenCount = 1000,
              Temperature = 0.0f,
              TopP = 1.0f,
            };

            List<ChatMessage> messages =
            [
              new SystemChatMessage("You are a helpful data analyst assistant that provides data statistics in JSON format."),
              new UserChatMessage(prompt)
            ];

            var response = await chatClient.CompleteChatAsync(messages, requestOptions);
            var completion = response.Value;

            var content = completion.Content[0].Text;

            try
            {
                // Extract JSON from the response (in case there's explanatory text)
                var jsonStart = content.IndexOf('{');
                var jsonEnd = content.LastIndexOf('}');

                if (jsonStart >= 0 && jsonEnd >= 0)
                {
                    var jsonContent = content.Substring(jsonStart, jsonEnd - jsonStart + 1);
                    return JsonSerializer.Deserialize<Dictionary<string, object>>(jsonContent) ?? new Dictionary<string, object>();
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error parsing JSON response: {ex.Message}");
            }

            return new Dictionary<string, object>();
        }
    }
}
```

### 8. Update Program.cs

Replace the contents of `Program.cs` with the following code:

```csharp
using AzureOpenAIDataInsights;
using Microsoft.Extensions.Configuration;
using System.Globalization;

// Set culture for consistent number formatting
CultureInfo.CurrentCulture = CultureInfo.InvariantCulture;

// Load configuration
var configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .Build();

// Get Azure OpenAI service configuration values
var endpoint = configuration["AzureOpenAI:Endpoint"] ?? throw new InvalidOperationException("Endpoint is not configured");
var key = configuration["AzureOpenAI:Key"] ?? throw new InvalidOperationException("API Key is not configured");
var deploymentName = configuration["AzureOpenAI:DeploymentName"] ?? throw new InvalidOperationException("Deployment name is not configured");

// Initialize services
var csvDataService = new CsvDataService();
var dataAnalyticsService = new DataAnalyticsService(endpoint, key, deploymentName);

// Load the CSV data
var dataFilePath = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "Data", "sales_data.csv");
Console.WriteLine($"Loading data from {dataFilePath}");

List<SalesRecord> salesData;
try
{
    salesData = csvDataService.ReadCsvFile<SalesRecord>(dataFilePath);
    Console.WriteLine($"Loaded {salesData.Count} records from the CSV file.");
    Console.WriteLine();
}
catch (Exception ex)
{
    Console.WriteLine($"Error loading CSV data: {ex.Message}");
    return;
}

// Main menu
while (true)
{
    Console.WriteLine("\n=== Sales Data Analysis ===");
    Console.WriteLine("1. View Data Summary");
    Console.WriteLine("2. Analyze Sales by Region");
    Console.WriteLine("3. Analyze Sales by Category");
    Console.WriteLine("4. Analyze Sales by Customer Type");
    Console.WriteLine("5. Generate Visualization Code");
    Console.WriteLine("6. Get Data Statistics");
    Console.WriteLine("7. Ask a Custom Question");
    Console.WriteLine("8. Exit");
    Console.Write("\nSelect an option: ");

    if (!int.TryParse(Console.ReadLine(), out int choice))
    {
        Console.WriteLine("Invalid input. Please enter a number.");
        continue;
    }

    try
    {
        switch (choice)
        {
            case 1: // Data Summary
                Console.WriteLine("\nGenerating data summary...");
                var summary = await dataAnalyticsService.GenerateDataSummaryAsync(salesData, "sales");
                Console.WriteLine("\n=== Data Summary ===");
                Console.WriteLine(summary);
                break;

            case 2: // Sales by Region
                Console.WriteLine("\nAnalyzing sales by region...");
                var regionAnalysis = await dataAnalyticsService.AnalyzeDataAsync(
                    salesData,
                    "What are the total sales and units sold by region? Which region is performing the best, and are there any trends over time?"
                );
                Console.WriteLine("\n=== Sales by Region Analysis ===");
                Console.WriteLine(regionAnalysis);
                break;

            case 3: // Sales by Category
                Console.WriteLine("\nAnalyzing sales by category...");
                var categoryAnalysis = await dataAnalyticsService.AnalyzeDataAsync(
                    salesData,
                    "What are the total sales and units sold by product category? Which category is the most profitable, and which has the highest volume of units sold?"
                );
                Console.WriteLine("\n=== Sales by Category Analysis ===");
                Console.WriteLine(categoryAnalysis);
                break;

            case 4: // Sales by Customer Type
                Console.WriteLine("\nAnalyzing sales by customer type...");
                var customerAnalysis = await dataAnalyticsService.AnalyzeDataAsync(
                    salesData,
                    "How do sales differ between Business and Consumer customer types? Which customer type generates more revenue, and which buys more units?"
                );
                Console.WriteLine("\n=== Sales by Customer Type Analysis ===");
                Console.WriteLine(customerAnalysis);
                break;

            case 5: // Generate Visualization Code
                Console.WriteLine("\nWhat type of visualization would you like?");
                Console.WriteLine("1. Sales by Region (Bar Chart)");
                Console.WriteLine("2. Sales by Category (Pie Chart)");
                Console.WriteLine("3. Sales Trend Over Time (Line Chart)");
                Console.WriteLine("4. Custom Visualization");
                Console.Write("\nSelect an option: ");

                if (!int.TryParse(Console.ReadLine(), out int vizChoice))
                {
                    Console.WriteLine("Invalid input. Please enter a number.");
                    continue;
                }

                string vizType = "";
                string vizDescription = "";

                switch (vizChoice)
                {
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
                        Console.Write("\nEnter visualization type (e.g., bar chart, pie chart, line chart): ");
                        vizType = Console.ReadLine() ?? "chart";
                        Console.Write("Enter what the visualization should show: ");
                        vizDescription = Console.ReadLine() ?? "sales data";
                        break;
                    default:
                        Console.WriteLine("Invalid choice. Using default bar chart.");
                        vizType = "bar chart";
                        vizDescription = "total sales by region";
                        break;
                }

                Console.WriteLine($"\nGenerating {vizType} code for {vizDescription}...");
                var vizCode = await dataAnalyticsService.GenerateVisualizationCodeAsync(
                    salesData,
                    vizType,
                    vizDescription
                );
                Console.WriteLine("\n=== Visualization Code ===");
                Console.WriteLine(vizCode);
                break;

            case 6: // Data Statistics
                Console.WriteLine("\nCalculating data statistics...");
                var statistics = await dataAnalyticsService.GetDataStatisticsAsync(salesData);
                Console.WriteLine("\n=== Data Statistics ===");

                foreach (var stat in statistics)
                {
                    Console.WriteLine($"{stat.Key}:");

                    if (stat.Value is JsonElement jsonElement)
                    {
                        // Handle JsonElement
                        if (jsonElement.ValueKind == JsonValueKind.Object)
                        {
                            foreach (var property in jsonElement.EnumerateObject())
                            {
                                Console.WriteLine($"  {property.Name}: {property.Value}");
                            }
                        }
                        else
                        {
                            Console.WriteLine($"  {jsonElement}");
                        }
                    }
                    else
                    {
                        // Handle other types
                        Console.WriteLine($"  {stat.Value}");
                    }

                    Console.WriteLine();
                }
                break;

            case 7: // Custom Question
                Console.Write("\nEnter your question about the sales data: ");
                var customQuestion = Console.ReadLine();

                if (string.IsNullOrWhiteSpace(customQuestion))
                {
                    Console.WriteLine("Question cannot be empty.");
                    continue;
                }

                Console.WriteLine("\nAnalyzing data to answer your question...");
                var customAnswer = await dataAnalyticsService.AnalyzeDataAsync(salesData, customQuestion);
                Console.WriteLine("\n=== Analysis Results ===");
                Console.WriteLine(customAnswer);
                break;

            case 8: // Exit
                Console.WriteLine("\nExiting program. Goodbye!");
                return;

            default:
                Console.WriteLine("\nInvalid option. Please select a number between 1 and 8.");
                break;
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"\nError: {ex.Message}");
    }

    Console.WriteLine("\nPress any key to continue...");
    Console.ReadKey();
    Console.Clear();
}
```