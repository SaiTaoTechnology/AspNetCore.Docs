---
title: Call a web API from an ASP.NET Core Blazor app
author: guardrex
description: Learn how to call a web API from Blazor apps.
monikerRange: '>= aspnetcore-3.1'
ms.author: riande
ms.custom: mvc
ms.date: 02/09/2024
uid: blazor/call-web-api
---
# Call a web API from ASP.NET Core Blazor

[!INCLUDE[](~/includes/not-latest-version.md)]

This article describes how to call a web API from a Blazor app.

## Package

The [`System.Net.Http.Json`](https://www.nuget.org/packages/System.Net.Http.Json) package provides extension methods for <xref:System.Net.Http.HttpClient?displayProperty=fullName> and <xref:System.Net.Http.HttpContent?displayProperty=fullName> that perform automatic serialization and deserialization using [`System.Text.Json`](https://www.nuget.org/packages/System.Text.Json). The `System.Net.Http.Json` package is provided by the .NET shared framework and doesn't require adding a package reference to the app.

:::moniker range=">= aspnetcore-8.0"

## Sample apps

See the sample apps in the [`dotnet/blazor-samples`](https://github.com/dotnet/blazor-samples/) GitHub repository.

### `BlazorWebAppCallWebApi`

Call an "external" todo list web API from a Blazor Web App:

* `Backend`: A web API app for maintaining a todo list, based on [Minimal APIs](xref:fundamentals/minimal-apis). The web API app is a separate app from the Blazor Web App, possibly hosted on a different server.
* `BlazorApp`/`BlazorApp.Client`: A Blazor Web App that calls the web API app with an <xref:System.Net.Http.HttpClient> for todo list operations, such as creating, reading, updating, and deleting (CRUD) items from the todo list.

For client-side rendering (CSR), which includes Interactive WebAssembly components and Auto components that have adopted CSR, calls are made with a preconfigured <xref:System.Net.Http.HttpClient> registered in the `Program` file of the client project (`BlazorApp.Client`):

```csharp
builder.Services.AddScoped(sp => new HttpClient { ... });
```

For server-side rendering (SSR), which includes prerendered and interactive Server components, prerendered WebAssembly components, and Auto components that are prerendered or have adopted SSR, calls are made with an <xref:System.Net.Http.HttpClient> registered in the `Program` file of the server project (`BlazorApp`):

```csharp
builder.Services.AddHttpClient();
```

Call an "internal" movie list API, where the API resides in the server project of the Blazor Web App:

* `BlazorApp`: A Blazor Web App that maintains a movie list:
  * When operations are performed on the movie list within the app on the server, ordinary API calls are used.
  * When API calls are made by a web-based client, a web API is used for movie list operations, based on [Minimal APIs](xref:fundamentals/minimal-apis).
* `BlazorApp.Client`: The client project of the Blazor Web App, which contains Interactive WebAssembly and Auto components for user management of the movie list.

For CSR, which includes Interactive WebAssembly components and Auto components that have adopted CSR, calls to the API are made via a client-based service (`ClientMovieService`) that uses a preconfigured <xref:System.Net.Http.HttpClient> registered in the `Program` file of the client project (`BlazorApp.Client`). Because these calls are made over a public or private web, the movie list API is a *web API*.

The following example obtains a list of movies:

```csharp
public class ClientMovieService(IConfiguration config, HttpClient httpClient) 
    : IMovieService
{
    private readonly string serviceEndpoint = 
        $"{config.GetValue<string>("FrontendUrl")}/movies";

    private HttpClient Http { get; set; } = httpClient;

    public async Task<Movie[]> GetMoviesAsync(bool watchedMovies)
    {
        var requestUri = watchedMovies ? $"{serviceEndpoint}/watched" : 
            serviceEndpoint;

        return await Http.GetFromJsonAsync<Movie[]>(requestUri) ?? [];
    }
}
```

For SSR, which includes prerendered and interactive Server components, prerendered WebAssembly components, and Auto components that are prerendered or have adopted SSR, calls are made directly via a server-based service (`ServerMovieService`). The API doesn't rely on a network, so it's a standard API for movie list CRUD operations.

The following example obtains a list of movies:

```csharp
public class ServerMovieService(MovieContext db) : IMovieService
{
    public async Task<Movie[]> GetMoviesAsync(bool watchedMovies)
    {
        return watchedMovies ? 
            await db.Movies.Where(t => t.IsWatched).ToArrayAsync() : 
            await db.Movies.ToArrayAsync();
    }
}
```

### `BlazorWebAppCallWebApi_Weather`

A weather data sample app that uses streaming rendering for weather data.

### `BlazorWebAssemblyCallWebApi`

Calls a todo list web API from a Blazor WebAssembly app:

* `Backend`: A web API app for maintaining a todo list, based on [Minimal APIs](xref:fundamentals/minimal-apis).
* `BlazorTodo`: A Blazor WebAssembly app that calls the web API with a preconfigured <xref:System.Net.Http.HttpClient> for todo list CRUD operations.

:::moniker-end

## Server-side scenarios for calling external web APIs

Server-based components call external web APIs using <xref:System.Net.Http.HttpClient> instances, typically created using <xref:System.Net.Http.IHttpClientFactory>. For guidance that applies to server-side apps, see <xref:fundamentals/http-requests>.

A server-side app doesn't include an <xref:System.Net.Http.HttpClient> service by default. Provide an <xref:System.Net.Http.HttpClient> to the app using the [`HttpClient` factory infrastructure](xref:fundamentals/http-requests).

In the `Program` file:

```csharp
builder.Services.AddHttpClient();
```

The following Razor component makes a request to a web API for GitHub branches similar to the *Basic Usage* example in the <xref:fundamentals/http-requests> article.

`CallWebAPI.razor`:

```razor
@page "/call-web-api"
@using System.Text.Json
@using System.Text.Json.Serialization
@inject IHttpClientFactory ClientFactory

<h1>Call web API from a Blazor Server Razor component</h1>

@if (getBranchesError || branches is null)
{
    <p>Unable to get branches from GitHub. Please try again later.</p>
}
else
{
    <ul>
        @foreach (var branch in branches)
        {
            <li>@branch.Name</li>
        }
    </ul>
}

@code {
    private IEnumerable<GitHubBranch>? branches = Array.Empty<GitHubBranch>();
    private bool getBranchesError;
    private bool shouldRender;

    protected override bool ShouldRender() => shouldRender;

    protected override async Task OnInitializedAsync()
    {
        var request = new HttpRequestMessage(HttpMethod.Get,
            "https://api.github.com/repos/dotnet/AspNetCore.Docs/branches");
        request.Headers.Add("Accept", "application/vnd.github.v3+json");
        request.Headers.Add("User-Agent", "HttpClientFactory-Sample");

        var client = ClientFactory.CreateClient();

        var response = await client.SendAsync(request);

        if (response.IsSuccessStatusCode)
        {
            using var responseStream = await response.Content.ReadAsStreamAsync();
            branches = await JsonSerializer.DeserializeAsync
                <IEnumerable<GitHubBranch>>(responseStream);
        }
        else
        {
            getBranchesError = true;
        }

        shouldRender = true;
    }

    public class GitHubBranch
    {
        [JsonPropertyName("name")]
        public string? Name { get; set; }
    }
}
```

For an additional working example, see the server-side file upload example that uploads files to a web API controller in the <xref:blazor/file-uploads#upload-files-to-a-server-with-server-side-rendering> article.

:::moniker range=">= aspnetcore-8.0"

## Blazor Web App server project-based (internal) web APIs

*This section applies to Blazor Web Apps that maintain a web API in the server project of the app (internal).*

When using the interactive WebAssembly and Auto render modes, components are prerendered by default. Auto components are also initially rendered interactively from the server before the Blazor bundle downloads to the client and the client-side runtime activates. This means that components using these render modes should be designed so that they run successfully from both the client and the server. If the component must call a server project-based API when running on the client, the recommended approach is to abstract that API call behind a service interface and implement client and server versions of the service:

* The client version calls the web API with a preconfigured <xref:System.Net.Http.HttpClient>.
* The server version can typically access the server-side resources directly. Injecting an <xref:System.Net.Http.HttpClient> on the server that makes calls back to the server is ***not*** recommended, as the network request is typically unnecessary.

When using the WebAssembly render mode, you also have the option of disabling prerendering, so the components only render from the client. For more information, see <xref:blazor/components/render-modes#prerendering>.

Examples ([sample apps](#sample-apps)):

* Movie list web API in the `BlazorWebAppCallWebApi` sample app.
* Streaming rendering weather data web API in the `BlazorWebAppCallWebApi_Weather` sample app.

## Blazor Web App external web APIs

*This section applies to Blazor Web Apps that call a web API maintained by a separate (external) project, possibly hosted on a different server.*

Blazor Web Apps normally prerender client-side WebAssembly components, and Auto components render on the server during static or interactive server-side rendering (SSR). <xref:System.Net.Http.HttpClient> services aren't registered by default in a Blazor Web App's main project. If the app is run with only the <xref:System.Net.Http.HttpClient> services registered in the `.Client` project, as described in the [Add the `HttpClient` service](#add-the-httpclient-service) section, executing the app results in a runtime error:

> :::no-loc text="InvalidOperationException: Cannot provide a value for property 'Http' on type '...{COMPONENT}'. There is no registered service of type 'System.Net.Http.HttpClient'.":::

Use ***either*** of the following approaches:

* Add the <xref:System.Net.Http.HttpClient> services to the server project to make the <xref:System.Net.Http.HttpClient> available during SSR. Use the following service registration in the server project's `Program` file:

  ```csharp
  builder.Services.AddHttpClient();
  ```

  No explicit package reference is required because <xref:System.Net.Http.HttpClient> services are provided by the shared framework.

  Example: Todo list web API in the `BlazorWebAppCallWebApi` [sample app](#sample-apps)

* If prerendering isn't required for a WebAssembly component that calls the web API, disable prerendering by following the guidance in <xref:blazor/components/render-modes#prerendering>. If you adopt this approach, you don't need to add <xref:System.Net.Http.HttpClient> services to the main project of the Blazor Web App because the component won't be prerendered on the server.

For more information, see [Client-side services fail to resolve during prerendering](xref:blazor/components/render-modes#client-side-services-fail-to-resolve-during-prerendering).

## Prerendered data

When prerendering, components render twice: first statically, then interactively. State doesn't automatically flow from the prerendered component to the interactive one. If a component performs asynchronous initialization operations and renders different content for different states during initialization, such as a "Loading..." progress indicator, you may see a flicker when the component renders twice.

<!-- UPDATE 9.0 Keep an eye on the status of the enhanced nav fix
                currently scheduled for Pre4. Remove the cross-link
                to the PU issue when 9.0 releases. 
                Note that the README of the "weather" call web API
                sample has a cross-link and remark on this, and the
                sample app disabled enhanced nav on the weather
                component link. -->

You can address this by flowing prerendered state using the Persistent Component State API, which the `BlazorWebAppCallWebApi` and `BlazorWebAppCallWebApi_Weather` [sample apps](#sample-apps) demonstrate. When the component renders interactively, it can render the same way using the same state. However, the API doesn't currently work with enhanced navigation, which you can work around by disabling enhanced navigation on links to the page (`data-enhanced-nav=false`). For more information, see the following resources:

* <xref:blazor/components/prerender#persist-prerendered-state>
* <xref:blazor/fundamentals/routing#enhanced-navigation-and-form-handling>
* [Support persistent component state across enhanced page navigations (`dotnet/aspnetcore` #51584)](https://github.com/dotnet/aspnetcore/issues/51584)

:::moniker-end

## Add the `HttpClient` service

*The guidance in this section applies to client-side scenarios.*

Client-side components call web APIs using a preconfigured <xref:System.Net.Http.HttpClient> service, which is focused on making requests back to the server of origin. Additional <xref:System.Net.Http.HttpClient> service configurations for other web APIs can be created in developer code. Requests are composed using Blazor JSON helpers or with <xref:System.Net.Http.HttpRequestMessage>. Requests can include [Fetch API](https://developer.mozilla.org/docs/Web/API/Fetch_API) option configuration.

The configuration examples in this section are only useful when a single web API is called for a single <xref:System.Net.Http.HttpClient> instance in the app. When the app must call multiple web APIs, each with its own base address and configuration, you can adopt the following approaches, which are covered later in this article:

* [Named `HttpClient` with `IHttpClientFactory`](#named-httpclient-with-ihttpclientfactory): Each web API is provided a unique name. When app code or a Razor component calls a web API, it uses a named <xref:System.Net.Http.HttpClient> instance to make the call.
* [Typed `HttpClient`](#typed-httpclient): Each web API is typed. When app code or a Razor component calls a web API, it uses a typed <xref:System.Net.Http.HttpClient> instance to make the call.

In the `Program` file, add an <xref:System.Net.Http.HttpClient> service if it isn't already present from a Blazor project template used to create the app:

```csharp
builder.Services.AddScoped(sp => 
    new HttpClient
    {
        BaseAddress = new Uri(builder.HostEnvironment.BaseAddress)
    });
```

The preceding example sets the base address with `builder.HostEnvironment.BaseAddress` (<xref:Microsoft.AspNetCore.Components.WebAssembly.Hosting.IWebAssemblyHostEnvironment.BaseAddress%2A?displayProperty=nameWithType>), which gets the base address for the app and is typically derived from the `<base>` tag's `href` value in the host page.

The most common use cases for using the client's own base address are:

* The client project (`.Client`) of a Blazor Web App (.NET 8 or later) makes web API calls from WebAssembly components or code that runs on the client in WebAssembly to APIs in the server app.
* The client project (**:::no-loc text="Client":::**) of a hosted Blazor WebAssembly app makes web API calls to the server project (**:::no-loc text="Server":::**). Note that the hosted Blazor WebAssembly project template is no longer available in .NET 8 or later. However, hosted Blazor WebAssembly apps remain supported for .NET 8.

If you're calling an external web API (not in the same URL space as the client app), set the URI to the web API's base address. The following example sets the base address of the web API to `https://localhost:5001`, where a separate web API app is running and ready to respond to requests from the client app:

```csharp
builder.Services.AddScoped(sp => 
    new HttpClient
    {
        BaseAddress = new Uri("https://localhost:5001")
    });
```

## JSON helpers

<xref:System.Net.Http.HttpClient> is available as a preconfigured service for making requests back to the origin server.

<xref:System.Net.Http.HttpClient> and JSON helpers (<xref:System.Net.Http.Json.HttpClientJsonExtensions?displayProperty=nameWithType>) are also used to call third-party web API endpoints. <xref:System.Net.Http.HttpClient> is implemented using the browser's [Fetch API](https://developer.mozilla.org/docs/Web/API/Fetch_API) and is subject to its limitations, including enforcement of the same-origin policy, which is discussed later in this article in the *Cross-Origin Resource Sharing (CORS)* section.

The client's base address is set to the originating server's address. Inject an <xref:System.Net.Http.HttpClient> instance into a component using the [`@inject`](xref:mvc/views/razor#inject) directive:

```razor
@using System.Net.Http
@inject HttpClient Http
```

Use the <xref:System.Net.Http.Json?displayProperty=fullName> namespace for access to <xref:System.Net.Http.Json.HttpClientJsonExtensions>, including <xref:System.Net.Http.Json.HttpClientJsonExtensions.GetFromJsonAsync%2A>, <xref:System.Net.Http.Json.HttpClientJsonExtensions.PutAsJsonAsync%2A>, and <xref:System.Net.Http.Json.HttpClientJsonExtensions.PostAsJsonAsync%2A>:

```razor
@using System.Net.Http.Json
```

### GET from JSON (`GetFromJsonAsync`)

<xref:System.Net.Http.Json.HttpClientJsonExtensions.GetFromJsonAsync%2A> sends an HTTP GET request and parses the JSON response body to create an object.

In the following component code, the `todoItems` are displayed by the component. <xref:System.Net.Http.Json.HttpClientJsonExtensions.GetFromJsonAsync%2A> is called when the component is finished initializing ([`OnInitializedAsync`](xref:blazor/components/lifecycle#component-initialization-oninitializedasync)).

> [!NOTE]
> When targeting ASP.NET Core 5.0 or earlier, add `@using` directives to the following component for <xref:System.Net.Http?displayProperty=fullName>, <xref:System.Net.Http.Json?displayProperty=fullName>, and <xref:System.Threading.Tasks?displayProperty=fullName>.

```csharp
todoItems = await Http.GetFromJsonAsync<TodoItem[]>("todoitems");
```

### POST as JSON (`PostAsJsonAsync`)

<xref:System.Net.Http.Json.HttpClientJsonExtensions.PostAsJsonAsync%2A> sends a POST request to the specified URI containing the value serialized as JSON in the request body.

In the following component code, `newItemName` is provided by a bound element of the component. The `AddItem` method is triggered by selecting a `<button>` element.

> [!NOTE]
> When targeting ASP.NET Core 5.0 or earlier, add `@using` directives to the following component for <xref:System.Net.Http?displayProperty=fullName>, <xref:System.Net.Http.Json?displayProperty=fullName>, and <xref:System.Threading.Tasks?displayProperty=fullName>.

```csharp
await Http.PostAsJsonAsync("todoitems", addItem);
```

<xref:System.Net.Http.Json.HttpClientJsonExtensions.PostAsJsonAsync%2A> returns an <xref:System.Net.Http.HttpResponseMessage>. To deserialize the JSON content from the response message, use the <xref:System.Net.Http.Json.HttpContentJsonExtensions.ReadFromJsonAsync%2A> extension method. The following example reads JSON weather data as an array:

:::moniker range=">= aspnetcore-6.0"

```csharp
var content = await response.Content.ReadFromJsonAsync<WeatherForecast[]>() ??
    Array.Empty<WeatherForecast>();
```

In the preceding example, an empty array is created if no weather data is returned by the method, so `content` isn't null after the statement executes.

:::moniker-end

:::moniker range="< aspnetcore-6.0"

```csharp
var content = await response.Content.ReadFromJsonAsync<WeatherForecast[]>();
```

:::moniker-end

### PUT as JSON (`PutAsJsonAsync`)

<xref:System.Net.Http.Json.HttpClientJsonExtensions.PutAsJsonAsync%2A> sends an HTTP PUT request with JSON-encoded content.

In the following component code, `editItem` values for `Name` and `IsCompleted` are provided by bound elements of the component. The item's `Id` is set when the item is selected in another part of the UI (not shown) and `EditItem` is called. The `SaveItem` method is triggered by selecting the `<button>` element. The following example doesn't show loading `todoItems` for brevity. See the [GET from JSON (`GetFromJsonAsync`)](#get-from-json-getfromjsonasync) section for an example of loading items.

> [!NOTE]
> When targeting ASP.NET Core 5.0 or earlier, add `@using` directives to the following component for <xref:System.Net.Http?displayProperty=fullName>, <xref:System.Net.Http.Json?displayProperty=fullName>, and <xref:System.Threading.Tasks?displayProperty=fullName>.

```csharp
await Http.PutAsJsonAsync($"todoitems/{editItem.Id}", editItem);
```

<xref:System.Net.Http.Json.HttpClientJsonExtensions.PutAsJsonAsync%2A> returns an <xref:System.Net.Http.HttpResponseMessage>. To deserialize the JSON content from the response message, use the <xref:System.Net.Http.Json.HttpContentJsonExtensions.ReadFromJsonAsync%2A> extension method. The following example reads JSON weather data as an array:

:::moniker range=">= aspnetcore-6.0"

```csharp
var content = await response.Content.ReadFromJsonAsync<WeatherForecast[]>() ??
    Array.Empty<WeatherForecast>();
```

In the preceding example, an empty array is created if no weather data is returned by the method, so `content` isn't null after the statement executes.

:::moniker-end

:::moniker range="< aspnetcore-6.0"

```csharp
var content = await response.Content.ReadFromJsonAsync<WeatherForecast[]>();
```

:::moniker-end

:::moniker range=">= aspnetcore-7.0"

### PATCH as JSON (`PatchAsJsonAsync`)

> [!NOTE]
> This section's example isn't currently demonstrated in the sample apps (`BlazorWebAppCallWebApi`/`BlazorWebAssemblyCallWebApi`, .NET 8 or later).

<xref:System.Net.Http.Json.HttpClientJsonExtensions.PatchAsJsonAsync%2A> sends an HTTP PATCH request with JSON-encoded content.

This section's examples are based on a `TodoItem` class that stores the following todo item data:

* ID (`Id`, `long`): Unique ID of the item.
* Name (`Name`, `string`): Name of the item.
* Status (`IsComplete`, `bool`): Indication if the todo item is finished.

`TodoItem` class:

```csharp
public class TodoItem
{
    public long Id { get; set; }
    public string? Name { get; set; }
    public bool IsComplete { get; set; }
}
```

In the following component code:

* `incompleteTodoItems` is an array of incomplete `TodoItem`. The following example doesn't show loading `incompleteTodoItems` for brevity. Loading items is covered in the [GET from JSON (`GetFromJsonAsync`)](#get-from-json-getfromjsonasync) section.
* The `UpdateItem` method is triggered by selecting the `<button>` element.
* The PATCH document is provided as a plain text string. The web API described in the <xref:tutorials/first-web-api> article doesn't handle PATCH requests by default. To make the PATCH example in this section work with the tutorial's web API, implement a PATCH controller action in the web API following the guidance in <xref:web-api/jsonpatch>. Later, this section demonstrates an example controller action and shows how to compose PATCH documents for ASP.NET Core web API apps that use .NET JSON PATCH support.

> [!NOTE]
> When targeting ASP.NET Core 5.0 or earlier, add `@using` directives to the following component for <xref:System.Net.Http?displayProperty=fullName>, <xref:System.Net.Http.Json?displayProperty=fullName>, and <xref:System.Threading.Tasks?displayProperty=fullName>.

```razor
@using System.Text.Json
@using System.Text.Json.Serialization
@inject HttpClient Http

<ul>
    @foreach (var item in incompleteTodoItems)
    {
        <li>
            @item.Name 
            <button @onclick="_ => UpdateItem(item.Id)">
                Mark 'Complete'
            </button>
        </li>
    }
</ul>

@code {
    private async Task UpdateItem(long id) =>
        await Http.PatchAsJsonAsync(
            $"todoitems/{id}", 
            "[{\"operationType\":2,\"path\":\"/IsComplete\",\"op\":\"replace\",\"value\":true}]");
}
```

<xref:System.Net.Http.Json.HttpClientJsonExtensions.PatchAsJsonAsync%2A> returns an <xref:System.Net.Http.HttpResponseMessage>. To deserialize the JSON content from the response message, use the <xref:System.Net.Http.Json.HttpContentJsonExtensions.ReadFromJsonAsync%2A> extension method. The following example reads JSON todo item data as an array. An empty array is created if no item data is returned by the method, so `content` isn't null after the statement executes:

```csharp
var content = await response.Content.ReadFromJsonAsync<TodoItem[]>() ??
    Array.Empty<TodoItem>();
```

<xref:System.Net.Http.Json.HttpClientJsonExtensions.PatchAsJsonAsync%2A> receives a JSON PATCH document for the PATCH request. The preceding `UpdateItem` method called <xref:System.Net.Http.Json.HttpClientJsonExtensions.PatchAsJsonAsync%2A> with a PATCH document as a string with escaped quotes. Laid out with indentation, spacing, and non-escaped quotes, the unencoded PATCH document appears as the following JSON:

```json
[
  {
    "operationType": 2,
    "path": "/IsComplete",
    "op": "replace",
    "value": true
  }
]
```

To simplify the creation of PATCH documents in the app issuing PATCH requests, an app can use .NET JSON PATCH support, as the following guidance demonstrates.

Install the [`Microsoft.AspNetCore.JsonPatch`](https://www.nuget.org/packages/Microsoft.AspNetCore.JsonPatch) NuGet package and use the API features of the package to compose a <xref:Microsoft.AspNetCore.JsonPatch.JsonPatchDocument> for a PATCH request.

[!INCLUDE[](~/includes/package-reference.md)]

Add an `@using` directive for the <xref:Microsoft.AspNetCore.JsonPatch?displayProperty=fullName> namespace to the top of the Razor component:

```razor
@using Microsoft.AspNetCore.JsonPatch
```

Compose the <xref:Microsoft.AspNetCore.JsonPatch.JsonPatchDocument> for a `TodoItem` with `IsComplete` set to `true` using the <xref:Microsoft.AspNetCore.JsonPatch.JsonPatchDocument.Replace%2A> method:

```csharp
var patchDocument = new JsonPatchDocument<TodoItem>()
    .Replace(p => p.IsComplete, true);
```

Pass the document's operations (`patchDocument.Operations`) to the <xref:System.Net.Http.Json.HttpClientJsonExtensions.PatchAsJsonAsync%2A> call. The following example shows how to make the call:

```csharp
private async Task UpdateItem(long id)
{
    await Http.PatchAsJsonAsync(
        $"todoitems/{id}", 
        patchDocument.Operations, 
        new JsonSerializerOptions()
        {
            DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingDefault,
            WriteIndented = true
        });
}
```

<xref:System.Text.Json.JsonSerializerOptions.DefaultIgnoreCondition?displayProperty=nameWithType> is set to <xref:System.Text.Json.Serialization.JsonIgnoreCondition.WhenWritingDefault?displayProperty=nameWithType> to ignore a property only if it equals the default value for its type.

<xref:System.Text.Json.JsonSerializerOptions.WriteIndented?displayProperty=nameWithType> is used merely to present the JSON payload in a pleasant format for this article. Writing indented JSON has no bearing on processing PATCH requests and isn't typically performed in production apps for web API requests.

Next, follow the guidance in the <xref:web-api/jsonpatch> article to add a PATCH controller action to the web API, which is reproduced here in the following steps.

Add a package reference for the [`Microsoft.AspNetCore.Mvc.NewtonsoftJson`](https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.NewtonsoftJson) NuGet package to the web API app.

> [!NOTE]
> There's no need to add a package reference for the [`Microsoft.AspNetCore.JsonPatch`](https://www.nuget.org/packages/Microsoft.AspNetCore.JsonPatch) package to the app because the reference to the `Microsoft.AspNetCore.Mvc.NewtonsoftJson` package automatically transitively adds a package reference for `Microsoft.AspNetCore.JsonPatch`.

Add a custom JSON PATCH input formatter to the web API app.

`JSONPatchInputFormatter.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Formatters;
using Microsoft.Extensions.Options;

public static class JSONPatchInputFormatter
{
    public static NewtonsoftJsonPatchInputFormatter Get()
    {
        var builder = new ServiceCollection()
            .AddLogging()
            .AddMvc()
            .AddNewtonsoftJson()
            .Services.BuildServiceProvider();

        return builder
            .GetRequiredService<IOptions<MvcOptions>>()
            .Value
            .InputFormatters
            .OfType<NewtonsoftJsonPatchInputFormatter>()
            .First();
    }
}
```

Configure the web API's controllers to use the `Microsoft.AspNetCore.Mvc.NewtonsoftJson` package and process PATCH requests with the JSON PATCH input formatter. Insert the `JSONPatchInputFormatter` in the first position of MVC's input formatter collection so that it processes requests prior to any other input formatter.

In the `Program` file modify the call to <xref:Microsoft.Extensions.DependencyInjection.MvcServiceCollectionExtensions.AddControllers%2A>:

```csharp
builder.Services.AddControllers(options =>
{
    options.InputFormatters.Insert(0, JSONPatchInputFormatter.Get());
}).AddNewtonsoftJson();
```

In `Controllers/TodoItemsController.cs`, add a `using` statement for the <xref:Microsoft.AspNetCore.JsonPatch?displayProperty=fullName> namespace:

```csharp
using Microsoft.AspNetCore.JsonPatch;
```

In `Controllers/TodoItemsController.cs`, add the following `PatchTodoItem` action method:

```csharp
[HttpPatch("{id}")]
public async Task<IActionResult> PatchTodoItem(long id, 
    JsonPatchDocument<TodoItem> patchDoc)
{
    if (patchDoc == null)
    {
        return BadRequest();
    }
    
    var todoItem = await _context.TodoItems.FindAsync(id);

    if (todoItem == null)
    {
        return NotFound();
    }

    patchDoc.ApplyTo(todoItem);

    _context.Entry(todoItem).State = EntityState.Modified;

    try
    {
        await _context.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException) when (!TodoItemExists(id))
    {
        return NotFound();
    }

    return NoContent();
}
```

> [!WARNING]
> As with the other examples in the <xref:web-api/jsonpatch> article, the preceding PATCH controller action doesn't protect the web API from over-posting attacks. For more information, see <xref:tutorials/first-web-api#prevent-over-posting>.

:::moniker-end

### Additional extension methods

<xref:System.Net.Http> includes additional extension methods for sending HTTP requests and receiving HTTP responses. <xref:System.Net.Http.HttpClient.DeleteAsync%2A?displayProperty=nameWithType> is used to send an HTTP DELETE request to a web API.

In the following component code, the `<button>` element calls the `DeleteItem` method. The bound `<input>` element supplies the `id` of the item to delete.

> [!NOTE]
> When targeting ASP.NET Core 5.0 or earlier, add `@using` directives to the following component for <xref:System.Net.Http?displayProperty=fullName> and <xref:System.Threading.Tasks?displayProperty=fullName>.

```csharp
await Http.DeleteAsync($"todoitems/{id}");
```

## Named `HttpClient` with `IHttpClientFactory`

<xref:System.Net.Http.IHttpClientFactory> services and the configuration of a named <xref:System.Net.Http.HttpClient> are supported.

> [!NOTE]
> An alternative to using a named <xref:System.Net.Http.HttpClient> from an <xref:System.Net.Http.IHttpClientFactory> is to use a typed <xref:System.Net.Http.HttpClient>. For more information, see the [Typed `HttpClient`](#typed-httpclient) section.

Add the [`Microsoft.Extensions.Http`](https://www.nuget.org/packages/Microsoft.Extensions.Http) NuGet package to the app.

[!INCLUDE[](~/includes/package-reference.md)]

In the `Program` file:

```csharp
builder.Services.AddHttpClient("WebAPI", client => 
    client.BaseAddress = new Uri(builder.HostEnvironment.BaseAddress));
```

The preceding example sets the base address with `builder.HostEnvironment.BaseAddress` (<xref:Microsoft.AspNetCore.Components.WebAssembly.Hosting.IWebAssemblyHostEnvironment.BaseAddress%2A?displayProperty=nameWithType>), which gets the base address for the app and is typically derived from the `<base>` tag's `href` value in the host page.

The most common use cases for using the client's own base address are:

* The client project (`.Client`) of a Blazor Web App (.NET 8 or later) makes web API calls from WebAssembly components or code that runs on the client in WebAssembly to APIs in the server app.
* The client project (**:::no-loc text="Client":::**) of a hosted Blazor WebAssembly app makes web API calls to the server project (**:::no-loc text="Server":::**).

If you're calling an external web API (not in the same URL space as the client app), set the URI to the web API's base address. The following example sets the base address of the web API to `https://localhost:5001`, where a separate web API app is running and ready to respond to requests from the client app:

```csharp
builder.Services.AddHttpClient("WebAPI", client => 
    client.BaseAddress = new Uri(https://localhost:5001));
```

In the following component code:

* An instance of <xref:System.Net.Http.IHttpClientFactory> creates a named <xref:System.Net.Http.HttpClient>.
* The named <xref:System.Net.Http.HttpClient> is used to issue a GET request for JSON weather forecast data from the web API.

> [!NOTE]
> When targeting ASP.NET Core 5.0 or earlier, add `@using` directives to the following component for <xref:System.Net.Http?displayProperty=fullName>, <xref:System.Net.Http.Json?displayProperty=fullName>, and <xref:System.Threading.Tasks?displayProperty=fullName>.

`FetchDataViaFactory.razor`:

```razor
@page "/fetch-data-via-factory"
@inject IHttpClientFactory ClientFactory

<h1>Fetch data via <code>IHttpClientFactory</code></h1>

@if (forecasts == null)
{
    <p><em>Loading...</em></p>
}
else
{
    <h2>Temperatures by Date</h2>

    <ul>
        @foreach (var forecast in forecasts)
        {
            <li>
                @forecast.Date.ToShortDateString():
                @forecast.TemperatureC &#8451;
                @forecast.TemperatureF &#8457;
            </li>
        }
    </ul>
}

@code {
    private WeatherForecast[]? forecasts;

    protected override async Task OnInitializedAsync()
    {
        var client = ClientFactory.CreateClient("WebAPI");

        forecasts = await client.GetFromJsonAsync<WeatherForecast[]>(
            "WeatherForecast");
    }
}
```

For an additional example based on calling Microsoft Graph with a named <xref:System.Net.Http.HttpClient>, see <xref:blazor/security/webassembly/graph-api?pivots=named-client-graph-api>.

## Typed `HttpClient`

Typed <xref:System.Net.Http.HttpClient> uses one or more of the app's <xref:System.Net.Http.HttpClient> instances, default or named, to return data from one or more web API endpoints.

> [!NOTE]
> An alternative to using a typed <xref:System.Net.Http.HttpClient> is to use a named <xref:System.Net.Http.HttpClient> from an <xref:System.Net.Http.IHttpClientFactory>. For more information, see the [Named `HttpClient` with `IHttpClientFactory`](#named-httpclient-with-ihttpclientfactory) section.

Add the [`Microsoft.Extensions.Http`](https://www.nuget.org/packages/Microsoft.Extensions.Http) NuGet package to the app.

[!INCLUDE[](~/includes/package-reference.md)]

`WeatherForecastHttpClient.cs`:

:::moniker range=">= aspnetcore-8.0"

```csharp
using System.Net.Http.Json;

namespace BlazorSample.Client;

public class WeatherForecastHttpClient(HttpClient http)
{
    private readonly HttpClient http = http;
    private WeatherForecast[]? forecasts;

    public async Task<WeatherForecast[]> GetForecastAsync()
    {
        forecasts = await http.GetFromJsonAsync<WeatherForecast[]>(
            "WeatherForecast");

        return forecasts ?? [];
    }
}
```

:::moniker-end

:::moniker range="< aspnetcore-8.0"

```csharp
using System.Net.Http.Json;

public class WeatherForecastHttpClient
{
    private readonly HttpClient http;
    private WeatherForecast[]? forecasts;

    public WeatherForecastHttpClient(HttpClient http)
    {
        this.http = http;
    }

    public async Task<WeatherForecast[]> GetForecastAsync()
    {
        forecasts = await http.GetFromJsonAsync<WeatherForecast[]>(
            "WeatherForecast");

        return forecasts ?? Array.Empty<WeatherForecast>();
    }
}
```

:::moniker-end

In the `Program` file:

```csharp
builder.Services.AddHttpClient<WeatherForecastHttpClient>(client => 
    client.BaseAddress = new Uri(builder.HostEnvironment.BaseAddress));
```

The preceding example sets the base address with `builder.HostEnvironment.BaseAddress` (<xref:Microsoft.AspNetCore.Components.WebAssembly.Hosting.IWebAssemblyHostEnvironment.BaseAddress%2A?displayProperty=nameWithType>), which gets the base address for the app and is typically derived from the `<base>` tag's `href` value in the host page.

The most common use cases for using the client's own base address are:

* The client project (`.Client`) of a Blazor Web App (.NET 8 or later) makes web API calls from WebAssembly/Auto components or code that runs on the client in WebAssembly to APIs in the server app.
* The client project (**:::no-loc text="Client":::**) of a hosted Blazor WebAssembly app makes web API calls to the server project (**:::no-loc text="Server":::**).

If you're calling an external web API (not in the same URL space as the client app), set the URI to the web API's base address. The following example sets the base address of the web API to `https://localhost:5001`, where a separate web API app is running and ready to respond to requests from the client app:

```csharp
builder.Services.AddHttpClient<WeatherForecastHttpClient>(client => 
    client.BaseAddress = new Uri(https://localhost:5001));
```

Components inject the typed <xref:System.Net.Http.HttpClient> to call the web API.

In the following component code:

* An instance of the preceding `WeatherForecastHttpClient` is injected, which creates a typed <xref:System.Net.Http.HttpClient>.
* The typed <xref:System.Net.Http.HttpClient> is used to issue a GET request for JSON weather forecast data from the web API.

> [!NOTE]
> When targeting ASP.NET Core 5.0 or earlier, add an `@using` directive to the following component for <xref:System.Threading.Tasks?displayProperty=fullName>.

`FetchDataViaTypedHttpClient.razor`:

```razor
@page "/fetch-data-via-typed-httpclient"
@inject WeatherForecastHttpClient Http

<h1>Fetch data via typed <code>HttpClient</code></h1>

@if (forecasts == null)
{
    <p><em>Loading...</em></p>
}
else
{
    <h2>Temperatures by Date</h2>

    <ul>
        @foreach (var forecast in forecasts)
        {
            <li>
                @forecast.Date.ToShortDateString():
                @forecast.TemperatureC &#8451;
                @forecast.TemperatureF &#8457;
            </li>
        }
    </ul>
}

@code {
    private WeatherForecast[]? forecasts;

    protected override async Task OnInitializedAsync()
    {
        forecasts = await Http.GetForecastAsync();
    }
}
```

## `HttpClient` and `HttpRequestMessage` with Fetch API request options

*The guidance in this section applies to client-side scenarios that rely upon bearer token authentication.*

[`HttpClient`](xref:fundamentals/http-requests) ([API documentation](xref:System.Net.Http.HttpClient)) and <xref:System.Net.Http.HttpRequestMessage> can be used to customize requests. For example, you can specify the HTTP method and request headers. The following component makes a `POST` request to a web API endpoint and shows the response body.

> [!NOTE]
> When targeting ASP.NET Core 5.0 or earlier, add `@using` directives to the following component for <xref:System.Net.Http?displayProperty=fullName> and <xref:System.Net.Http.Json?displayProperty=fullName>.

`TodoRequest.razor`:

```razor
@page "/todo-request"
@using System.Net.Http.Headers
@using Microsoft.AspNetCore.Components.WebAssembly.Authentication
@inject HttpClient Http
@inject IAccessTokenProvider TokenProvider

<h1>ToDo Request</h1>

<h1>ToDo Request Example</h1>

<button @onclick="PostRequest">Submit POST request</button>

<p>Response body returned by the server:</p>

<p>@responseBody</p>

@code {
    private string? responseBody;

    private async Task PostRequest()
    {
        var requestMessage = new HttpRequestMessage()
        {
            Method = new HttpMethod("POST"),
            RequestUri = new Uri("https://localhost:10000/todoitems"),
            Content =
                JsonContent.Create(new TodoItem
                {
                    Name = "My New Todo Item",
                    IsComplete = false
                })
        };

        var tokenResult = await TokenProvider.RequestAccessToken();

        if (tokenResult.TryGetToken(out var token))
        {
            requestMessage.Headers.Authorization =
                new AuthenticationHeaderValue("Bearer", token.Value);

            requestMessage.Content.Headers.TryAddWithoutValidation(
                "x-custom-header", "value");

            var response = await Http.SendAsync(requestMessage);
            var responseStatusCode = response.StatusCode;

            responseBody = await response.Content.ReadAsStringAsync();
        }
    }

    public class TodoItem
    {
        public long Id { get; set; }
        public string? Name { get; set; }
        public bool IsComplete { get; set; }
    }
}
```

Blazor's client-side implementation of <xref:System.Net.Http.HttpClient> uses [Fetch API](https://developer.mozilla.org/docs/Web/API/fetch) and configures the underlying [request-specific Fetch API options](https://developer.mozilla.org/docs/Web/API/fetch#Parameters) via <xref:System.Net.Http.HttpRequestMessage> extension methods and <xref:Microsoft.AspNetCore.Components.WebAssembly.Http.WebAssemblyHttpRequestMessageExtensions>. Set additional options using the generic <xref:Microsoft.AspNetCore.Components.WebAssembly.Http.WebAssemblyHttpRequestMessageExtensions.SetBrowserRequestOption%2A> extension method. Blazor and the underlying Fetch API don't directly add or modify request headers. For more information on how user agents, such as browsers, interact with headers, consult external user agent documentation sets and other web resources.

The HTTP response is typically buffered to enable support for synchronous reads on the response content. To enable support for response streaming, use the <xref:Microsoft.AspNetCore.Components.WebAssembly.Http.WebAssemblyHttpRequestMessageExtensions.SetBrowserResponseStreamingEnabled%2A> extension method on the request.

To include credentials in a cross-origin request, use the <xref:Microsoft.AspNetCore.Components.WebAssembly.Http.WebAssemblyHttpRequestMessageExtensions.SetBrowserRequestCredentials%2A> extension method:

```csharp
requestMessage.SetBrowserRequestCredentials(BrowserRequestCredentials.Include);
```

For more information on Fetch API options, see [MDN web docs: WindowOrWorkerGlobalScope.fetch(): Parameters](https://developer.mozilla.org/docs/Web/API/fetch#Parameters).

## Handle errors

Handle web API response errors in developer code when they occur. For example, <xref:System.Net.Http.Json.HttpClientJsonExtensions.GetFromJsonAsync%2A> expects a JSON response from the web API with a `Content-Type` of `application/json`. If the response isn't in JSON format, content validation throws a <xref:System.NotSupportedException>.

In the following example, the URI endpoint for the weather forecast data request is misspelled. The URI should be to `WeatherForecast` but appears in the call as `WeatherForcast`, which is missing the letter `e` in `Forecast`.

The <xref:System.Net.Http.Json.HttpClientJsonExtensions.GetFromJsonAsync%2A> call expects JSON to be returned, but the web API returns HTML for an unhandled exception with a `Content-Type` of `text/html`. The unhandled exception occurs because the path to `/WeatherForcast` isn't found and middleware can't serve a page or view for the request.

In <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> on the client, <xref:System.NotSupportedException> is thrown when the response content is validated as non-JSON. The exception is caught in the `catch` block, where custom logic could log the error or present a friendly error message to the user.

> [!NOTE]
> When targeting ASP.NET Core 5.0 or earlier, add `@using` directives to the following component for <xref:System.Net.Http?displayProperty=fullName>, <xref:System.Net.Http.Json?displayProperty=fullName>, and <xref:System.Threading.Tasks?displayProperty=fullName>.

`ReturnHTMLOnException.razor`:

```razor
@page "/return-html-on-exception"
@using {PROJECT NAME}.Shared
@inject HttpClient Http

<h1>Fetch data but receive HTML on unhandled exception</h1>

@if (forecasts == null)
{
    <p><em>Loading...</em></p>
}
else
{
    <h2>Temperatures by Date</h2>

    <ul>
        @foreach (var forecast in forecasts)
        {
            <li>
                @forecast.Date.ToShortDateString():
                @forecast.TemperatureC &#8451;
                @forecast.TemperatureF &#8457;
            </li>
        }
    </ul>
}

<p>
    @exceptionMessage
</p>

@code {
    private WeatherForecast[]? forecasts;
    private string? exceptionMessage;

    protected override async Task OnInitializedAsync()
    {
        try
        {
            // The URI endpoint "WeatherForecast" is misspelled on purpose on the 
            // next line. See the preceding text for more information.
            forecasts = await Http.GetFromJsonAsync<WeatherForecast[]>("WeatherForcast");
        }
        catch (NotSupportedException exception)
        {
            exceptionMessage = exception.Message;
        }
    }
}
```

> [!NOTE]
> The preceding example is for demonstration purposes. A web API can be configured to return JSON even when an endpoint doesn't exist or an unhandled exception occurs on the server.

For more information, see <xref:blazor/fundamentals/handle-errors>.

## Cross-Origin Resource Sharing (CORS)

Browser security restricts a webpage from making requests to a different domain than the one that served the webpage. This restriction is called the *same-origin policy*. The same-origin policy restricts (but doesn't prevent) a malicious site from reading sensitive data from another site. To make requests from the browser to an endpoint with a different origin, the *endpoint* must enable [Cross-Origin Resource Sharing (CORS)](https://www.w3.org/TR/cors/).

For more information on server-side CORS, see <xref:security/cors>. The article's examples don't pertain directly to Razor component scenarios, but the article is useful for learning general CORS concepts.

For information on client-side CORS requests, see <xref:blazor/security/webassembly/additional-scenarios#cross-origin-resource-sharing-cors>.

:::moniker range=">= aspnetcore-8.0"

## Antiforgery support

To add antiforgery support to an HTTP request, inject the `AntiforgeryStateProvider` and add a `RequestToken` to the headers collection as a `RequestVerificationToken`:

```razor
@inject AntiforgeryStateProvider Antiforgery
```

```csharp
private async Task OnSubmit()
{
    var antiforgery = Antiforgery.GetAntiforgeryToken();
    var request = new HttpRequestMessage(HttpMethod.Post, "action");
    request.Headers.Add("RequestVerificationToken", antiforgery.RequestToken);
    var response = await client.SendAsync(request);
    ...
}
```

For more information, see <xref:blazor/security/index#antiforgery-support>.

:::moniker-end

## Blazor framework component examples for testing web API access

Various network tools are publicly available for testing web API backend apps directly, such as [Firefox Browser Developer](https://www.mozilla.org/firefox/developer/). Blazor framework's reference source includes <xref:System.Net.Http.HttpClient> test assets that are useful for testing:

[`HttpClientTest` assets in the `dotnet/aspnetcore` GitHub repository](https://github.com/dotnet/aspnetcore/tree/main/src/Components/test/testassets/BasicTestApp/HttpClientTest)

[!INCLUDE[](~/includes/aspnetcore-repo-ref-source-links.md)]

## Additional resources

### General

* [Cross-Origin Resource Sharing (CORS) at W3C](https://www.w3.org/TR/cors/)
* <xref:security/cors>: Although the content applies to ASP.NET Core apps, not Razor components, the article covers general CORS concepts.

### Server-side

* <xref:blazor/security/server/additional-scenarios>: Includes coverage on using <xref:System.Net.Http.HttpClient> to make secure web API requests.
* <xref:fundamentals/http-requests>
* <xref:security/enforcing-ssl>
* [Kestrel HTTPS endpoint configuration](xref:fundamentals/servers/kestrel/endpoints)

### Client-side

* <xref:blazor/security/webassembly/additional-scenarios>: Includes coverage on using <xref:System.Net.Http.HttpClient> to make secure web API requests.
* <xref:blazor/security/webassembly/graph-api>
* [Fetch API](https://developer.mozilla.org/docs/Web/API/fetch)
