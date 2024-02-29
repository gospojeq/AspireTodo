# AspireTodo

This repo contains the sample code we'll use for the .NET + Azure (via Aspire) training day. 

> Note, I'm just iterating over the steps in here quickly and will go back and refine. If you look at the repo's commit history you'll see these steps embodied, commit-by-commit, in the actual source code for the app. 

## Day 1

These are the high-level steps you'll perform on Day 1. 

### Creating a new Aspire project

You'l create a basic project and then learn how to run it and examine how the app is running using the .NET Aspire dashboard. 

* Create a new Aspire Starter project without the Redis Output Caching enabled 
* Run the app to see the Aspire Dashboard 
* Run through the various ways of looking at console logs for each project 
* Observe how the Trace node provides distributed tracing as users hit the frontend of the site
* Stop the debugger

Next, you'll start editing the code to turn it into your very own Todo app. 

---

### Change the code to be a "Todo" app

In this phase, you'll make some basic modifications to the backend API and frontend Web project to turn the template's content into a real Todo app. 

* Change `WeatherApiClient` to `TodoApiClient`
* Reflect the type name change in the `Web` project's `Program.cs` during build-up
* Change the `WeatherForecast` record types in both the `ApiService` project and the `Web` project into `TodoItem` record types, with a `Description` string property and a `IsCompleted` bool property
* Change the `ApiService` project's `Program.cs` such that it returns a `List<ToDoItem>` at the endpoint `/todos` and build the list in the `Program.cs` file with a sample list of `TodoItem` objects
* Rename the `Web` project's `Weather.razor` in the `Pages` directory to `Todos.razor`
* Delete the `Counter.razor` and `Home.razor` files from the `Web` project's `Pages` folder
* Change the `Web` project's `Layout/NavMenu.razor` so that it only has the `Home` link, deleting `Weather` and `Counter`, but change the link text to `Todo`
* Edit the `Web` project's `Pages/Todo.razor` file to display `TodoItem` objects instead of the old `WeatherForecast` objects, using the `TodoApiClient` class instead of the `WeatherApiClient` class
* Edit the `Web` project's `Pages/Todo.razor` file such that it is what renders in the browser when the `/` (root) endpoint is called from a web browser

With these changes made, you're ready to deploy the app right up to Azure to get started learning the platform's components. 

---

### Deploy the app to Azure

In this phase, you'll publish your new AspireTodo app to Azure using the Azure Developer CLI or, Visual Studio (which uses the Azure Developer CLI as an underlying dependency). 

* Make sure AZD is defaulting to the .NET R&D subscription so you know you have everything you need to party today

    ```azd config set defaults.subscription <.NETR&DSubscriptionIdHere>```

* If you had Visual Studio open when you performed the `azd config set`, you may need to restart Visual Studio
* Right-click the `AppHost` project and select `Publish`, then go through the .NET Aspire Azure Container Apps publish flow to publish the app to Azure 
* If you aren't using Visual Studio, you can simply drop out a command prompt, `cd` into the root directory of your `.sln` file, and execute these commands:

    ```
    azd init
    azd provision
    azd deploy
    ```

* Browse to the Azure portal and see the variety of resources you've created
* Click on the `webfrontend` Azure Container App resource to view the overview of the app 
* Explore the `Ingress` area for both the `backendapi` and `webfrontend` container apps to take note of how the frontend is available via the open Internet, whereas the backend API app is private and secure

With the app published manually, you're ready to save your code and get ready for Day 2. 

---

### Push your app into a GitHub repo

At this point your code should be ready to save, so get it into a GitHub repo (public or private, doesn't matter).

* Use Visual Studio's GitHub features, the `gh` CLI tool, or your favorite method of creating Git repositories to create a new local Git repository and remote GitHub repository
* Commit and push your code to the remote repository

This should be the end of the stuff we get through in Day 1. We'll kick off Day 2 by automating the deployment of your app when you commit code to the repo in which you've saved your code. 

## Day 2

During Day 2, you'll add Continuous Integration / Continuous Deploy (CI/CD) capabilities to your GitHub repository, add database support, and use asynchrous messaging rather than direct HTTP calls between your frontend and backend apps. 

### Automate dotnet build

In this phase you'll automate the process of building the `AspireTodo` source code whenever you want, or, whenever you commit. 

* Open your GitHub AspireTodo repository and go to the `Actions` tab
* Search for the `Continuous integration` area
* Find the `.NET` item in this section, labelled `Build and test a .NET or ASP.NET Core project` and click the `Configure` button
* Change the `checkout` and `setup-dotnet` tasks to be `v4`
* Change the `dotnet-version` from `6.0.x` to `8.0.102`
* Add a step between the change you just made and the `Restore dependencies` step containing this YAML code

    ```yaml
    - name: Install workload
      run: dotnet workload install aspire
    ```

* Commit the YAML file back to your `main` branch once you've made these changes
* Browse to the `Actions` tab in GitHub and watch your continuous integration build your app

At this point, make sure you clone your changes back to your DevBox or Virtual Machine, so you have the changes you just made in the browser back down on your workstation. 

---

### Setting up Continuous Deployment

Since you already have an AZD environment provisioned in Azure *and* the local configuration specifying that AZD environment as the destination to which code should be deployed when changes happen, and since AZD works the same in CI/CD as it does locally, the process is simple. 

* At the command line, type the command

    ```
    azd pipeline config --auth-type client-credentials --provider github --principal-name augmentrprincipal --environment <yourEnvironmentNameHere>
    ```

    > Note: The parameter `augmentrprincipal` is a managed identity created for another sample, but it'll work here, too. Since not everyone in the class may have (or need) the permissions required to create Managed Identities in the R&D sub, we'll just use this one for now.

* Go to your GitHub repositorie's `Settings` area, and you'll notice that both secrets and variables have been injected into your repository by the `azd pipeline config` step
* Create a new file in the `.github\workflows` folder, named `continousdeploy.yml` and place this YAML code into it

```yaml
name: Provision and Deploy

on:
  workflow_dispatch:

jobs:

  build:
    runs-on: ubuntu-latest

    env:
      
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}

    steps:

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 8.0.102

      - name: Checkout
        uses: actions/checkout@v4

      - name: Install workload
        run: dotnet workload install aspire

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore

      - name: Install azd
        uses: Azure/setup-azd@v0.1.0
      
      - name: Log in with Azure (Client Credentials)
        if: ${{ env.AZURE_CREDENTIALS != '' }}
        run: |
          $info = $Env:AZURE_CREDENTIALS | ConvertFrom-Json -AsHashtable;
          Write-Host "::add-mask::$($info.clientSecret)"

          azd auth login `
            --client-id "$($info.clientId)" `
            --client-secret "$($info.clientSecret)" `
            --tenant-id "$($info.tenantId)"
        shell: pwsh
        env:
          AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Log in with Azure (Federated Credentials)
        if: ${{ env.AZURE_CLIENT_ID != '' }}
        run: |
          azd auth login `
            --client-id "$Env:AZURE_CLIENT_ID" `
            --federated-credential-provider "github" `
            --tenant-id "$Env:AZURE_TENANT_ID" --debug
        shell: pwsh

      - name: Provision Infrastructure
        run: azd provision --no-prompt
        env:
          AZURE_ENV_NAME: ${{ vars.AZURE_ENV_NAME }}
          AZURE_LOCATION: ${{ vars.AZURE_LOCATION }}
          AZURE_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}
          AZD_INITIAL_ENVIRONMENT_CONFIG: ${{ secrets.AZD_INITIAL_ENVIRONMENT_CONFIG }}

      - name: Deploy App
        run: azd deploy --no-prompt
        env:
          AZURE_ENV_NAME: ${{ vars.AZURE_ENV_NAME }}
          AZURE_LOCATION: ${{ vars.AZURE_LOCATION }}
          AZURE_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}
          AZD_INITIAL_ENVIRONMENT_CONFIG: ${{ secrets.AZD_INITIAL_ENVIRONMENT_CONFIG }}

```

* Commmit and push your code changes back into your remote GitHub repository
* Go to the `Actions` tab again and run the `Provision and Deploy` action

You'll observe how the GitHub Action logs into Azure, then uses `azd provision` and `azd deploy` to build your app's infrastructure and then deploy your app into it.

---

### Adding messaging

In this phase, you'll add messaging to the app so the frontend can be used to add new items to your todo list. The frontend will be used to collect a new todo item. It will drop a message on an Azure Storage Queue. You'll add a new `BackgroundWorker` class to the backend project that will receive the incoming messages and add them to the server-side todo list. 

* Add the `Aspire.Azure.Storage.Queues` package (version `8.0.0-preview.3.24105.21`) to both the `AppHost`, `ApiService`, and `Web` projects
* Add the `Aspire.Hosting.Azure` package (version `8.0.0-preview.3.24105.21`) to the `AppHost` project
* Update the `AppHost` project's `Program.cs` so that it contains a new service - the Azure Storage reference, along with a second reference to the Queue service Azure Storage offers for asynchronous messaging. 

  ```csharp
  var builder = DistributedApplication.CreateBuilder(args);

  var storage = builder.AddAzureStorage("azureStorage").UseEmulator();

  var queues = storage.AddQueues("azureQueues");

  var apiService = builder.AddProject<Projects.AspireTodo_ApiService>("apiservice")
      .WithReference(queues);

  var frontend = builder.AddProject<Projects.AspireTodo_Web>("webfrontend")
      .WithReference(queues)
      .WithReference(apiService);

  builder.Build().Run();
  ```

With Queueing activated app-wide, you can now add support for sending messages into a queue when they're received from the frontend. 

> Note: At this point, when you try to run the `AspireTodo` project, if you lack Docker Desktop, you'll be prompted to install it. That could take about a half-hour, depending on how powerful your machine is (it took about 10 minutes in a new DevBox).

---

### Sending Messages

In this phase, you'll add code to the frontend project that will accept user input and drop it into a queue. 

* In the `Web` project's `Program.cs`, use the `AddAzureQueueService` method to add queueing support to the frontend project

  ```csharp
  // Add service defaults & Aspire components.
  builder.AddServiceDefaults(); // this will be there already

  // Add Storage Queue Support
  builder.AddAzureQueueService("azureQueues");
  ```

* In the `Web` project's `Componentns\Pages\Todo.razor` file, replace the code you have with this update: 

  ```html
  @page "/"
  @rendermode InteractiveServer
  @using Azure.Storage.Queues
  @inject QueueServiceClient queueServiceClient

  @inject TodoApiClient TodoClient

  <PageTitle>AspireTodo</PageTitle>

  <h1>Todo Items</h1>

  <p>These are the things we AspireTodo.</p>

  @if (todos == null)
  {
      <p><em>Loading...</em></p>
  }
  else
  {
      <table class="table">
          <thead>
              <tr>
                  <th>Description</th>
              </tr>
          </thead>
          <tbody>
              @foreach (var todo in todos.Where(x => !x.IsCompleted))
              {
                  <tr>
                      <td>@todo.Description</td>
                  </tr>
              }
          </tbody>
          <tfoot>
              <tr>
                  <td>
                      <input type="text" @bind="@newTodoItemDescription" />
                      <input type="button" class="btn btn-primary" @onclick="SaveTodo" value="Save" />
                  </td>
              </tr>
          </tfoot>
      </table>
  }

  @code {
      private TodoItem[]? todos;
      private string newTodoItemDescription = "";

      protected override async Task OnInitializedAsync()
      {
          await queueServiceClient.GetQueueClient("incoming").CreateIfNotExistsAsync();
          todos = await TodoClient.GetAllTodoItems();
      }

      private async Task SaveTodo()
      {
          if (!string.IsNullOrEmpty(newTodoItemDescription))
              await queueServiceClient.GetQueueClient("incoming").SendMessageAsync(newTodoItemDescription);
          newTodoItemDescription = "";
      }
  }
  ```

* Run the app and post a few new todo items - you won't see the list update yet, but if you review the logs in the Aspire dashboard you'll see the messages are being sent 

At this point, you're finished with the code required to send messages using an Azure Queue. Commit your code back to the GitHub repository and get ready for the next phase. 

---

### Receiving Messages

In this phase, you'll add code to the backend project to start receiving the queued messages, so they can be added to the list of todo items asynchronously. 

