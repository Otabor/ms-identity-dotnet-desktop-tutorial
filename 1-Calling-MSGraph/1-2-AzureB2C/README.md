---
services: active-directory
platforms: dotnet
author: TiagoBrenck
level: 100
client: .NET Desktop (Console)
service: Microsoft Graph
endpoint: Microsoft identity platform
page_type: sample
languages:
  - csharp  
products:
  - azure
  - azure-active-directory  
  - dotnet
  - office-ms-graph
description: "This sample demonstrates a .NET Desktop (Console) application calling The Microsoft Graph"
---

# Using the Microsoft identity platform to call Microsoft Graph API from a multi-target console application.

![Build badge](https://identitydivision.visualstudio.com/_apis/public/build/definitions/a7934fdd-dcde-4492-a406-7fad6ac00e17/<BuildNumber>/badge)

## About this sample

### Overview

This sample demonstrates a .NET Desktop (Console) application calling The Microsoft Graph.

1. The .NET Desktop (Console) application uses the Microsoft Authentication Library (MSAL) to obtain a JWT access token from **Azure AD B2C**:
2. The access token is used as a bearer token to authenticate the user when calling the Microsoft Graph.

### Scenario

The console application:

- gets an access token from Azure AD B2C interactively
- and then calls the Microsoft Graph `/me` endpoint to get the user information, which it then displays in the console.

![Overview](./ReadmeFiles/topology.png)

## How to run this sample

To run this sample, you'll need:

- [Visual Studio 2017](https://aka.ms/vsdownload)
- An Internet connection
- (Optional) [create an Azure Active Directory B2C tenant](https://azure.microsoft.com/documentation/articles/active-directory-b2c-get-started)

### Step 1:  Clone or download this repository

From your shell or command line:

```Shell
git clone https://github.com/Azure-Samples/ms-identity-dotnet-desktop-tutorial.git
```

or download and extract the repository .zip file.

> Given that the name of the sample is quiet long, and so are the names of the referenced NuGet packages, you might want to clone it in a folder close to the root of your hard drive, to avoid file size limitations on Windows.

## Option 1 - Run the pre-configured sample

1. Build the solution and run it as is.

## Option 2 - Configure the sample with your own B2C app

1. If you don't have an Azure AD B2C tenant yet, you'll need to create an Azure AD B2C tenant by following the [Tutorial: Create an Azure Active Directory B2C tenant](https://azure.microsoft.com/documentation/articles/active-directory-b2c-get-started).
1. Go through the steps below to register your own app and configure the sample accordingly.

### Step 2: Create your own user flow (policy)

This sample uses a unified sign-up/sign-in user flow (policy). Create this policy by following [these instructions on creating an AAD B2C tenant](https://azure.microsoft.com/documentation/articles/active-directory-b2c-reference-policies). You may choose to include as many or as few identity providers as you wish, but make sure **DisplayName** is checked in `User attributes` and `Application claims`.

If you already have an existing unified sign-up/sign-in user flow (policy) in your Azure AD B2C tenant, feel free to re-use it. There is no need to create a new one just for this sample.

Copy the sign-up/sign-in user flow name, so you can use it in step 5.

### Step 2:  Register the sample application with your Azure AD B2C tenant

Now you need to [register your desktop app in your B2C tenant](https://docs.microsoft.com/en-us/azure/active-directory-b2c/add-native-application?tabs=applications), so that it has its own Application ID.

1. Navigate to the Microsoft identity platform for developers [App registrations](https://go.microsoft.com/fwlink/?linkid=2083908) page.
1. Click **New registration** on top.
1. In the **Register an application page** that appears, enter your application's registration information:
   - In the **Name** section, enter a meaningful application name that will be displayed to users of the app, for example `Console-Interactive-MultiTarget-v2`.
   - Change **Supported account types** to **Accounts in any organizational directory and personal Microsoft accounts (e.g. Skype, Xbox, Outlook.com)**.
1. Click on the **Register** button in bottom to create the application.
1. In the app's registration screen, find the **Application (client) ID** value and record it for use later. You'll need it to configure the configuration file(s) later in your code.
1. In the app's registration screen, click on the **Authentication** blade in the left.
   - If you don't have a platform added yet, click on **Add a platform** and select the **Public client (mobile & desktop)** option.
   - In the **Redirect URIs** section, enter the following redirect URIs.
      - `http://localhost`
   - In the **Redirect URIs** | **Suggested Redirect URIs for public clients (mobile, desktop)** section, select **https://login.microsoftonline.com/common/oauth2/nativeclient**

1. Click the **Save** button on top to save the changes.
1. In the app's registration screen, click on the **API permissions** blade in the left to open the page where we add access to the Apis that your application needs.
   - Click the **Add a permission** button and then,
   - Ensure that the **Microsoft APIs** tab is selected.
   - In the *Commonly used Microsoft APIs* section, click on **Microsoft Graph**
   - In the **Delegated permissions** section, select the **User.Read** in the list. Use the search box if necessary.
   - Click on the **Add permissions** button at the bottom.

##### Configure the  client app (Console-Interactive-MultiTarget-v2) to use your app registration

Open the project in your IDE (like Visual Studio) to configure the code.

1. Open the `Console-Interactive-MultiTarget\appsettings.json` file
1. Find the app key `ClientId` and replace the existing value with the application ID (clientId) of the `Console-Interactive-MultiTarget-v2` application copied from the Azure portal.
1. Find the app key `Domain` and replace the existing value with your Azure AD B2C tenant domain.
1. Find the app key `DefaultUserFlow` and replace the existing value with your sign-up/sign-in user flow name.

### Step 4: Run the sample

Clean the solution, rebuild the solution, and run it.

Start the application, sign-in and check the result in the console.

## About the code

The relevant code for this sample is in the `Program.cs` file, in the Main() method. The steps are:

1- Use `appsettings.json` as a configuration file.

```csharp
private static IConfiguration configuration;

var builder = new ConfigurationBuilder()
                .SetBasePath(System.IO.Directory.GetCurrentDirectory())
                .AddJsonFile("appsettings.json");

configuration = builder.Build();
```

2- Create the MSAL public client application.

```csharp
var app = PublicClientApplicationBuilder.Create(appConfiguration.ClientId)
                                                    .WithB2CAuthority(_authority)
                                                    .WithRedirectUri(appConfiguration.RedirectUri)
                                                    .Build();
```

3- Try to acquire an access token for Microsoft Graph silently, but if it fails, do it interactively.

```csharp
string[] scopes = new[] { "user.read" };
AuthenticationResult result;

try
{
    var accounts = await app.GetAccountsAsync();
    result = await app.AcquireTokenSilent(scopes, accounts.FirstOrDefault())
                .ExecuteAsync();
}
catch (MsalUiRequiredException)
{
    result = await app.AcquireTokenInteractive(scopes)
                .ExecuteAsync();
}
```

4- Instantiate `GraphServiceClient` (from [Microsoft.Graph NuGet package](https://docs.microsoft.com/graph/sdks/sdk-installation)) using the Microsoft Graph access token acquired in the previous step.

```csharp
private static GraphServiceClient GetGraphServiceClient(string accessToken, string graphApiUrl)
{
    GraphServiceClient graphServiceClient = new GraphServiceClient(graphApiUrl,
                                                            new DelegateAuthenticationProvider(
                                                                async (requestMessage) =>
                                                                {
                                                                    await Task.Run(() =>
                                                                    {
                                                                        requestMessage.Headers.Authorization = new AuthenticationHeaderValue("bearer", accessToken);
                                                                    });
                                                                }));

    return graphServiceClient;
}
```

5- Call Microsoft Graph `/me` endpoint, using [Microsoft Graph SDK](https://docs.microsoft.com/graph/sdks/create-requests?tabs=CS).

```csharp
string graphApiUrl = configuration.GetValue<string>("GraphApiUrl");

var graphClient = GetGraphServiceClient(result.AccessToken, graphApiUrl);

var me = await graphClient.Me.Request().GetAsync();
```

## Community Help and Support

Use [Stack Overflow](http://stackoverflow.com/questions/tagged/msal) to get support from the community.
Ask your questions on Stack Overflow first and browse existing issues to see if someone has asked your question before.
Make sure that your questions or comments are tagged with [`azure-active-directory` `msal` `dotnet`].

If you find a bug in the sample, please raise the issue on [GitHub Issues](../../issues).

To provide a recommendation, visit the following [User Voice page](https://feedback.azure.com/forums/169401-azure-active-directory).

## Contributing

If you'd like to contribute to this sample, see [CONTRIBUTING.MD](/CONTRIBUTING.md).

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information, see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## More information

For more information, see MSAL.NET's conceptual documentation:

- [MSAL.NET's conceptual documentation](https://aka.ms/msal-net)
- [Microsoft identity platform (Azure Active Directory for developers)](https://docs.microsoft.com/azure/active-directory/develop/)
- [Quickstart: Register an application with the Microsoft identity platform](https://docs.microsoft.com/azure/active-directory/develop/quickstart-register-app)
- [Quickstart: Configure a client application to access web APIs](https://docs.microsoft.com/azure/active-directory/develop/quickstart-configure-app-access-web-apis)

- [Understanding Azure AD application consent experiences](https://docs.microsoft.com/azure/active-directory/develop/application-consent-experience)
- [Understand user and admin consent](https://docs.microsoft.com/azure/active-directory/develop/howto-convert-app-to-be-multi-tenant#understand-user-and-admin-consent)
- [Application and service principal objects in Azure Active Directory](https://docs.microsoft.com/azure/active-directory/develop/app-objects-and-service-principals)

For more information about how OAuth 2.0 protocols work in this scenario and other scenarios, see [Authentication Scenarios for Azure AD](http://go.microsoft.com/fwlink/?LinkId=394414).