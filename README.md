# Angular SPA and .NET Core web API using App Roles to implement Role-Based Access Control

 1. [Overview](#overview)
 1. [Scenario](#scenario)
 1. [Contents](#contents)
 1. [Prerequisites](#prerequisites)
 1. [Setup](#setup)
 1. [Registration](#registration)
 1. [Running the sample](#running-the-sample)
 1. [Explore the sample](#explore-the-sample)
 1. [About the code](#about-the-code)
 1. [More information](#more-information)
 1. [Community Help and Support](#community-help-and-support)
 1. [Contributing](#contributing)

## Overview

This sample demonstrates a cross-platform application suite involving an Angular single-page application (*TodoListSPA*) calling an ASP.NET Core web API (*TodoListAPI*) secured with the Microsoft identity platform. In doing so, it implements **Role-based Access Control** (RBAC) by using Azure AD **App Roles**.

Access control in Azure AD can be done with **Security Groups** as well, as we will cover in the [next tutorial](../2-call-api-groups/README.md). **Security Groups** and **App Roles** in Azure AD are by no means mutually exclusive - they can be used in tandem to provide even finer grained access control.

## Scenario

In the sample, a **dashboard** component allows signed-in users to see the tasks assigned to users and is only accessible by users under an **app role** named **TaskAdmin**.

- The **TodoListSPA** uses [MSAL Angular](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-angular) to authenticate a user with the Microsoft identity platform.
- The app then obtains an [access token](https://docs.microsoft.com/azure/active-directory/develop/access-tokens) from Azure Active Directory (Azure AD) on behalf of the authenticated user for the **TodoListAPI**.
- **TodoListAPI** uses [Microsoft.Identity.Web](https://github.com/AzureAD/microsoft-identity-web) to protect its endpoint and accept only authorized calls.

![Topology](./ReadmeFiles/topology.png)

## Contents

| File/folder                         | Description                                                |
|-------------------------------------|------------------------------------------------------------|
| `SPA/src/app/auth-config.ts`        | Authentication parameters for SPA project reside here.     |
| `SPA/src/app/app.module.ts`         | MSAL Angular is initialized here.                          |
| `SPA/src/app/role-guard.service.ts` | This service protects other components that require user to be in a role. |
| `API/appsettings.json`              | Authentication parameters for API project reside here.     |
| `API/Startup.cs`                    | Microsoft.Identity.Web is initialized here.                |

## Prerequisites

- An **Azure AD** tenant. For more information see: [How to get an Azure AD tenant](https://docs.microsoft.com/azure/active-directory/develop/quickstart-create-new-tenant)
- At least **two** user accounts in your Azure AD tenant.

## Setup

Using a command line interface such as VS Code integrated terminal, follow the steps below:

### Step 1. Install .NET Core API dependencies

```console
    git clone https://github.com/Azure-Samples/ms-identity-javascript-angular-tutorial.git
```

or download and extract the repository .zip file.

> :warning: To avoid path length limitations on Windows, we recommend cloning into a directory near the root of your drive.

### Step 2. Install .NET Core API dependencies

```console
    cd ms-identity-javascript-angular-tutorial
    cd 5-AccessControl/1-call-api-roles/API
    dotnet restore
```

### Step 3. Trust development certificates

```console
    dotnet dev-certs https --clean
    dotnet dev-certs https --trust
```

For more information and potential issues, see: [HTTPS in .NET Core](https://docs.microsoft.com/aspnet/core/security/enforcing-ssl).

### Step 4. Install Angular SPA dependencies

```console
    cd ../
    cd SPA
    npm install
```

## Registration

### Register the sample application(s) with your Azure Active Directory tenant

There are two projects in this sample. Each needs to be separately registered in your Azure AD tenant. To register these projects, you can:

- follow the steps below for manually register your apps
- or use PowerShell scripts that:
  - **automatically** creates the Azure AD applications and related objects (passwords, permissions, dependencies) for you.
  - modify the projects' configuration files.

<details>
  <summary>Expand this section if you want to use this automation:</summary>

> :warning: If you have never used **Azure AD Powershell** before, we recommend you go through the [App Creation Scripts](./AppCreationScripts/AppCreationScripts.md) once to ensure that your environment is prepared correctly for this step.

1. On Windows, run PowerShell as **Administrator** and navigate to the root of the cloned directory
1. If you have never used Azure AD Powershell before, we recommend you go through the [App Creation Scripts](./AppCreationScripts/AppCreationScripts.md) once to ensure that your environment is prepared correctly for this step.
1. In PowerShell run:

   ```PowerShell
   Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process -Force
   ```

1. Run the script to create your Azure AD application and configure the code of the sample application accordingly.
1. In PowerShell run:

   ```PowerShell
   cd .\AppCreationScripts\
   .\Configure.ps1
   ```

   > Other ways of running the scripts are described in [App Creation Scripts](./AppCreationScripts/AppCreationScripts.md)
   > The scripts also provide a guide to automated application registration, configuration and removal which can help in your CI/CD scenarios.

</details>

### Choose the Azure AD tenant where you want to create your applications

As a first step you'll need to:

1. Sign in to the [Azure portal](https://portal.azure.com).
1. If your account is present in more than one Azure AD tenant, select your profile at the top right corner in the menu on top of the page, and then **switch directory** to change your portal session to the desired Azure AD tenant.

### Register the service app (msal-dotnet-api)

1. Navigate to the [Azure portal](https://portal.azure.com) and select the **Azure AD** service.
1. Select the **App Registrations** blade on the left, then select **New registration**.
1. In the **Register an application page** that appears, enter your application's registration information:
   - In the **Name** section, enter a meaningful application name that will be displayed to users of the app, for example `msal-dotnet-api`.
   - Under **Supported account types**, select **Accounts in this organizational directory only**.
1. Select **Register** to create the application.
1. In the app's registration screen, find and note the **Application (client) ID**. You use this value in your app's configuration file(s) later in your code.
1. Select **Save** to save your changes.
1. In the app's registration screen, select the **Expose an API** blade to the left to open the page where you can declare the parameters to expose this app as an API for which client applications can obtain [access tokens](https://docs.microsoft.com/azure/active-directory/develop/access-tokens) for.
The first thing that we need to do is to declare the unique [resource](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-auth-code-flow) URI that the clients will be using to obtain access tokens for this Api. To declare an resource URI, follow the following steps:
   - Select `Set` next to the **Application ID URI** to generate a URI that is unique for this app.
   - For this sample, accept the proposed Application ID URI (`api://{clientId}`) by selecting **Save**.
1. All APIs have to publish a minimum of one [scope](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-auth-code-flow#request-an-authorization-code) for the client's to obtain an access token successfully. To publish a scope, follow the following steps:
   - Select **Add a scope** button open the **Add a scope** screen and Enter the values as indicated below:
        - For **Scope name**, use `access_as_user`.
        - Select **Admins and users** options for **Who can consent?**.
        - For **Admin consent display name** type `Access msal-dotnet-api`.
        - For **Admin consent description** type `Allows the app to access msal-dotnet-api as the signed-in user.`
        - For **User consent display name** type `Access msal-dotnet-api`.
        - For **User consent description** type `Allow the application to access msal-dotnet-api on your behalf.`
        - Keep **State** as **Enabled**.
        - Select the **Add scope** button on the bottom to save this scope.
1. On the right side menu, select the `Manifest` blade.
   - Set `accessTokenAcceptedVersion` property to **2**.
   - Click on **Save**.

#### Define Application Roles

1. Still on the same app registration, select the **App roles** blade to the left.
1. Select **Create app role**:
    - For **Display name**, enter a suitable name, for instance **TaskAdmin**.
    - For **Allowed member types**, choose **User**.
    - For **Value**, enter **TaskAdmin**.
    - For **Description**, enter **Admins can read any user's todo list**.
1. Select **Create app role**:
    - For **Display name**, enter a suitable name, for instance **TaskUser**.
    - For **Allowed member types**, choose **User**.
    - For **Value**, enter **TaskUser**.
    - For **Description**, enter **Users can read and modify their todo lists**.
1. Select **Apply** to save your changes.

To add users to this app role, follow the guidelines here: [Assign users and groups to roles](https://docs.microsoft.com/azure/active-directory/develop/howto-add-app-roles-in-azure-ad-apps#assign-users-and-groups-to-roles).

> :bulb: **Important security tip**
>
> When you set **User assignment required?** to **Yes**, Azure AD will check that only users assigned to your application in the **Users and groups** blade are able to sign-in to your app. You can assign users directly or by assigning security groups they belong to.

For more information, see: [How to: Add app roles in your application and receive them in the token](https://docs.microsoft.com/azure/active-directory/develop/howto-add-app-roles-in-azure-ad-apps)

#### Configure the service app (msal-dotnet-api) to use your app registration

Open the project in your IDE (like Visual Studio or Visual Studio Code) to configure the code.

> In the steps below, "ClientID" is the same as "Application ID" or "AppId".

1. Open the `TodoListAPI\appsettings.json` file.
1. Find the key `Domain` and replace the existing value with your Azure AD tenant name.
1. Find the key `TenantId` and replace the existing value with your Azure AD tenant ID.
1. Find the key `ClientId` and replace the existing value with the application ID (clientId) of **msal-dotnet-api** app copied from the Azure portal.

1. Open the `API\Controllers\TodoListController.cs` file.
1. Find the variable `scopeRequiredByApi` and replace its value with the name of the API scope that you have just exposed (by default `access_as_user`).

### Register the client app (msal-angular-spa)

1. Navigate to the [Azure portal](https://portal.azure.com) and select the **Azure AD** service.
1. Select the **App Registrations** blade on the left, then select **New registration**.
1. In the **Register an application page** that appears, enter your application's registration information:
   - In the **Name** section, enter a meaningful application name that will be displayed to users of the app, for example `msal-angular-spa`.
   - Under **Supported account types**, select **Accounts in this organizational directory only**.
   - In the **Redirect URI (optional)** section, select **Single-page application** in the combo-box and enter the following redirect URI: `http://localhost:4200/`.
1. Select **Register** to create the application.
1. In the app's registration screen, find and note the **Application (client) ID**. You use this value in your app's configuration file(s) later in your code.
1. Select **Save** to save your changes.
1. In the app's registration screen, select the **API permissions** blade in the left to open the page where we add access to the APIs that your application needs.
   - Select the **Add a permission** button and then,
   - Ensure that the **My APIs** tab is selected.
   - In the list of APIs, select the API `msal-dotnet-api`.
   - In the **Delegated permissions** section, select the **Access 'msal-dotnet-api'** in the list. Use the search box if necessary.
   - Select the **Add permissions** button at the bottom.

#### Define Application Roles

1. Still on the same app registration, select the **App roles** blade to the left.
1. Select **Create app role**:
    - For **Display name**, enter a suitable name, for instance **TaskAdmin**.
    - For **Allowed member types**, choose **User**.
    - For **Value**, enter **TaskAdmin**.
    - For **Description**, enter **Admins can read any user's todo list**.
1. Select **Create app role**:
    - For **Display name**, enter a suitable name, for instance **TaskUser**.
    - For **Allowed member types**, choose **User**.
    - For **Value**, enter **TaskUser**.
    - For **Description**, enter **Users can read and modify their todo lists**.
1. Select **Apply** to save your changes.

To add users to this app role, follow the guidelines here: [Assign users and groups to roles](https://docs.microsoft.com/azure/active-directory/develop/howto-add-app-roles-in-azure-ad-apps#assign-users-and-groups-to-roles).

#### Configure the client app (msal-angular-spa) to use your app registration

Open the project in your IDE (like Visual Studio or Visual Studio Code) to configure the code.

> In the steps below, "ClientID" is the same as "Application ID" or "AppId".

1. Open the `TodoListSPA\src\app\auth-config.json` file.
1. Find the key `clientId` and replace the existing value with the application ID (clientId) of **msal-angular-spa** app copied from the Azure portal.
1. Find the key `tenantId` and replace the existing value with your Azure AD tenant ID copied from the Azure portal.
1. Find the key `protectedResources.todoListApi.scopes` and replace the existing value with scope you created during the app registration of `TodoListAPI` e.g. `api://{clientId_of_service_app}/access_as_user`.

To receive the `roles` claim in **Id** and **Access** tokens with the name of the app roles this user is assigned to, make sure that the user accounts you plan to sign-in to this app is assigned to the app roles of this app. The guide, [Assign a user or group to an enterprise app in Azure Active Directory](https://docs.microsoft.com/azure/active-directory/manage-apps/assign-user-or-group-access-portal#assign-a-user-to-an-app---portal) provides step by step instructions.

   | Role          | Rights                                                                   |
   |---------------|--------------------------------------------------------------------------|
   | `TaskAdmin`   | Admins can read others' TodoLists but cannot add/remove todos.           |
   | `TaskUser`    | Users can read and modify their TodoList but cannot see others' lists.   |

> :information_source: The number of **App Roles** that can be created for an app are limited by the [App Manifest limits](https://docs.microsoft.com/azure/active-directory/develop/reference-app-manifest#manifest-limits).

