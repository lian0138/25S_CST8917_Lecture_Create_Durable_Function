# CST8917 Lecture Exercise: Create and Deploy a Python Durable Functions App in Azure

## Introduction

In this exercise, you'll learn how to build and deploy a serverless **Durable Functions** app using **Azure Functions** and **Python**. Durable Functions extend Azure Functions by enabling **stateful workflows** in a **serverless** environment. This makes it possible to coordinate complex sequences of operations, such as calling multiple functions in order, handling retries, and tracking progress over time.

Using **Visual Studio Code** and the **Azure Functions extension**, you'll create a simple "Hello World" orchestrator that calls activity functions to greet multiple cities. You'll test it locally using the **Azurite** storage emulator, then deploy your project to Azure and verify it's working in the cloud.

This hands-on activity will help you:

- Understand the structure of a Durable Functions app (client, orchestrator, activity)
- Use Visual Studio Code and Azure Functions Core Tools for development
- Deploy serverless workflows to the cloud with minimal setup
- Gain experience using REST APIs to interact with long-running cloud processes

By the end of the lab, you’ll have a complete, working Durable Functions app and the knowledge to build more complex orchestrations for real-world use cases.

> **Note:**  
> This exercise is based on the official Microsoft quickstart:  
> [Quickstart: Create a Python Durable Functions app using Visual Studio Code](https://learn.microsoft.com/en-us/azure/azure-functions/durable/quickstart-python-vscode?tabs=linux)
---
##  Prerequisites
Ensure the following are installed:

- [Visual Studio Code](https://code.visualstudio.com/)
- Azure Functions extension for VS Code
- Azure Functions Core Tools (v4)
- [Azurite](https://marketplace.visualstudio.com/items?itemName=Azurite.azurite)
- Python 3.7–3.10
- An [Azure account](https://azure.microsoft.com/en-us/free/students/)
- REST Client extension for VS Code ( for `.http` testing)
---

## Instructions
### Step 1: Create the Local Project

1. Open **VS Code** and press `F1` → `Azure Functions: Create New Project`
2. Follow the prompts:
   - Select a folder
   - **Language**: Python
   - **Python version**: 3.10
   - **Template**: Skip for now

### Step 2: Install Required Packages

1. Replace `requirements.txt` content with:
    ```txt
    azure-functions
    azure-functions-durable
    ```
2. Then in terminal:
    ```# Activate virtual environment
    source .venv/bin/activate   # (Linux/macOS)
    or
    .venv\Scripts\activate      # (Windows)
    ```
3. Install dependencies
    ```
    pip install -r requirements.txt
    ```

### Step 3: Add Durable Functions Code
In `function_app.py`, paste:
```python
import azure.functions as func
import azure.durable_functions as df

myApp = df.DFApp(http_auth_level=func.AuthLevel.ANONYMOUS)

# An HTTP-triggered function with a Durable Functions client binding
@myApp.route(route="orchestrators/{functionName}", methods=["POST"]) 
@myApp.durable_client_input(client_name="client")
async def http_start(req: func.HttpRequest, client):
    function_name = req.route_params.get('functionName')
    instance_id = await client.start_new(function_name)
    response = client.create_check_status_response(req, instance_id)
    return response

# Orchestrator
@myApp.orchestration_trigger(context_name="context")
def hello_orchestrator(context):
    result1 = yield context.call_activity("hello", "Seattle")
    result2 = yield context.call_activity("hello", "Tokyo")
    result3 = yield context.call_activity("hello", "London")

    return [result1, result2, result3]

# Activity
@myApp.activity_trigger(input_name="city")
def hello(city: str):
    return f"Hello {city}"
```
#### Explanation
- `http_start` is the client function which is the entry point for your Durable Function app. When someone sends a `POST` request to `/api/orchestrators/hello_orchestrator`, this function:
    - Starts a new orchestration (hello_orchestrator)
    - Returns `URLs` to check status, send events, or terminate the workflow
- `hello_orchestrator` is the brain of the workflow. The orchestrator function:
    - Runs in a deterministic way (can be replayed exactly)
    - Calls the same activity function (hello) multiple times, passing in different city names
    - Waits for each activity to finish before moving to the next
    - Returns the list of results
- `hello` is activity function. Activity functions are like building blocks of logic:
    - They do a small piece of work (e.g., calling an API, processing data, or returning a greeting)
    - Here, it just returns "Hello <city>" for whatever city it receives

### Step 4: Configure Azurite for Local Testing
1. Edit local.settings.json:
    ```
    {
    "IsEncrypted": false,
    "Values": {
        "AzureWebJobsStorage": "UseDevelopmentStorage=true",
        "FUNCTIONS_WORKER_RUNTIME": "python"
        }
    }
    ```
2. Then start Azurite from Command Palette:
    ```
    Azurite: Start
    ```

### Step 5: Run and Test Locally
- Option 1: Start via VS Code Debugger
    - Press F5 or go to Run > Start Debugging
    
        This will build and start your function app, and output will appear in the Terminal and Debug Console.

- Option 2: Start via Terminal
    - Open a terminal in your project folder
    - Activate your virtual environment: `source .venv/bin/activate` (for linux).
    - Start the function app using Azure Functions Core Tools:
        ```
        func start
        ```

You should see output indicating that the app is running at:
    ```
    http://localhost:7071
    ```
### Testing
- Once your function app is running (either via F5 or func start), you'll test the orchestration by sending HTTP requests.

- In terminal, copy the `http://localhost:7071/api/orchestrators/hello_orchestrator` URL.

- Send a POST request using `.http` file:
    ```
    ### Start orchestration
    POST http://localhost:7071/api/orchestrators/hello_orchestrator
    Content-Type: application/json

    ### Query status (replace with actual status URL from response)
    GET http://localhost:7071/runtime/webhooks/durabletask/instances/<instanceId>?taskHub=TestHubName&connection=Storage

    ```
- What's Happening?
    - The HTTP POST request triggers the client function in your Durable Functions app. It starts a new instance of the orchestrator function called hello_orchestrator. The empty JSON {} represents the input payload (not used in this example).
    - The response will include a JSON object that contains several useful URLs, including:
        - statusQueryGetUri – used to check the status of the orchestration
        - sendEventPostUri, terminatePostUri, etc. – used for advanced orchestration control (not used in this exercise)
    - The HTTP GET request checks the current status of the orchestration instance you started.
    - You’ll replace <instanceId> with the actual value you got in the response to the POST request.
    - The response will show whether the orchestration is still running, completed, or failed — and include any output it returned (e.g., greetings from all the cities).


### Step 6: Deploy to Azure
1. In VS Code, sign in via Azure panel (if not already).
2. Run:
    ```
    Azure Functions: Create Function App in Azure
    ```
3. Follow prompts.
4. Deploy code: 
    ```
    Azure Functions: Deploy to Function App
    ```
5. Test in Azure
- Test again using the following URL format in your HTTP client: `https://<your-function-app-name>.azurewebsites.net/api/orchestrators/hello_orchestrator`



### Step 7: Push Your Code to GitHub
After testing locally and deploying to Azure, back up your project and share your code by pushing it to a GitHub repository, all from within VS Code.

1. Open the Source Control Panel:
    - Click the Source Control icon on the left sidebar (or press Ctrl+Shift+G).
2. Click the Publish to GitHub button in the Source Control panel.
    - Enter a repository name (e.g., durable-functions-lab)
    - Decide whether to make it public or private
    - Wait a few seconds — your code is now online on GitHub!
