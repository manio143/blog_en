---
title: 1. Writing system tests
---

# 1. Writing system tests

So here I am - I made a decision to make a rewrite of the service.
Where do I start? Well, the number one thing I want to do is to try to follow good practices.
I don't want to just write this thing from scratch and swap them out in one go, possibly breaking tens of existing users with something I haven't tested.
So I need to first document how the existing service works.
I will use system tests for that.

## What are system tests?

If you heard about the [test pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
you may be familiar with service or integration tests.
The way I define system tests is an external testing process making calls to a locally deployed application and validating the general behavior.

What this means is that the system test mostly doesn't care about implementation details and focuses on externalities.
For a web API that means most tests should be performable by making HTTP requests against the service.
In my case because I need to document a bit deeper how the service behaves I will also be connecting to the database to inspect it, clean up state created with the tests, etc.

### Setup
I've created a new .NET 6 project with the XUnit template and added [Xunit.DependencyInjection](https://github.com/pengweiqhca/Xunit.DependencyInjection)
which allows me to use dependency injection, configuration files and advanced logging controls.

I've also created a project for models and generated classes using Entity Framework Core [dotnet tool](https://learn.microsoft.com/en-us/ef/core/cli/dotnet).
The app uses postgres so I installed `Npgsql.EntityFrameworkCore.PostgreSQL`.
Then I ran `dotnet ef dbcontext scaffold` to set up the models.
I like to keep models mostly pure, so all my EF configuration ends up in the DbContext class.

Finally I needed some tools for making the HTTP requests and I opted for writing my own, specifically tailored to this app.

## HttpClient
I have read a lot in the past on how to use the `HttpClient`.
The most recent advice says to use `IHttpClientFactory` from the `Microsoft.Extensions.Http` package.
It allows us to use dependency injection to configure the client.
And you can name the client to have multiple configurations.

But the main benefit is that the factory intelligently caches the `HttpClientHandler` instances which hold system resources.
A big issue of manual handler management is port exhaustion when you create to many instances.
It's a general issue and I've seen systems die from this.

In .NET Core a new system of layers has been designed on top of the `HttpClientHandler`.
There's many small and reusable instances of `DelegatingHandler` which act very similarly to ASP.NET middleware.
It can be used for injecting headers or otherwise modifying the request or the response.

### Cookies
For accessing the management hub website you login with an email and a password and get a session cookie.
Standard stuff. Took me 2 days to make it work.

So the first thing is cookies.
As mentioned above, the `HttpClientHandler` is reused by multiple `HttpClient` instances to save on system resources.
This means that the built in cookie management mechanism are out the window, because they'd be shared across multiple clients.
We need the cookie store to be specific to a given test.
Also we need to allow tests to run in parallel for fast CI times later on.

So I looked around and came up with this middleware
(shout out to [damianh's gist](https://gist.github.com/damianh/038195c1ab0c5013ad3883d7e3c59d99) which saved me from parsing cookies by hand)

```csharp
/// <summary>
/// Message handler which substitutes the handling of cookies of the <see cref="HttpClientHandler"/>.
/// It allows to reuse a single instance of a underlying handler across multiple cookie sessions.
/// </summary>
public class CookieSessionMessageHandler : DelegatingHandler
{
	public static readonly HttpRequestOptionsKey<CookieContainer> CookieContainerOption
        = new("CookieContainer");

	protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
	{
		if (!request.Options.TryGetValue(CookieContainerOption, out var cookieContainer))
		    return await base.SendAsync(request, cancellationToken);

        if (cookieContainer.Count > 0)
            request.Headers.Add("Cookie", cookieContainer.GetCookieHeader(request.RequestUri!));

        var response = await base.SendAsync(request, cancellationToken);

        if (response.Headers.TryGetValues("Set-Cookie", out var cookieValues))
            foreach (var cookieValue in cookieValues)
            {
                cookieContainer.SetCookies(request.RequestUri!, cookieValue);
            }

        return response;
	}
}
```

I've cleaned up the code a bit for this post to keep it shorter, but I also have some debug logs in there.

We can see I'm using the `HttpRequestOptions` collection to store the `CookieContainer`.
This means I needed to write custom extensions over `HttpClient` for things like `GetStringAsync()`,
because I wanted to build my own `HttpRequestMessage` and add the cookie container.

The middleware also works if we don't care about cookies - this is generally a good practice -
make your middleware opt-in rather than opt-out.

For sending the cookies we're using a method of the `CookieContainer` called `GetCookieHeader(url)`.
For saving new cookies we're using the `SetCookies(url, cookieHeader)` method.
If you happen to have issues with it, checkout the helper class `SetCookieHeaderValue`, from `Microsoft.Net.Http.Headers` package, to parse the headers.

So with this cookie middleware I was able to log in, but afterwards something wasn't working and I had to debug it.

### Redirects
The session cookie system of Rails works in a way that sends a new cookie with each request.
I believe it's meant to prevent an attacker from stealing a session in some way (see [Rails guide on security](https://guides.rubyonrails.org/security.html#sessions)).

With this new cookie middleware I was sending redirect requests with the same cookie as the original request.

{{< mermaid >}}
sequenceDiagram
    CookieMiddleware->>HttpClientHandler: Headers["Cookie"]
    HttpClientHandler->>Website: HTTP request
    Website-->>HttpClientHandler: 302 Headers["Location"]
    HttpClientHandler->>Website: HTTP GET request to new location
    Website-->>HttpClientHandler: 200 OK
    HttpClientHandler-->>CookieMiddleware: Headers["Set-Cookie"]
{{< /mermaid >}}

And this wasn't working, so I had to write my own redirect layer on top of it.

```csharp
/// <summary>
/// Message handler which substitutes the handling of redirects of the <see cref="HttpClientHandler"/>.
/// We needed redirection layer on top of the custom cookie middleware.
/// </summary> 
public class FollowRedirectsMessageHandler : DelegatingHandler
{
	protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
	{
		var response = await base.SendAsync(request, cancellationToken);
		if (response.Headers.TryGetValues("Location", out var locations))
		{
			var redirectRequest = new HttpRequestMessage(HttpMethod.Get, locations.First());
			if (request.Options is IDictionary<string, object?> requestOptions &&
			    redirectRequest.Options is IDictionary<string, object?> newOptions)
                foreach (var kvp in requestOptions)
                {
                    newOptions.Add(kvp.Key, kvp.Value);
                }

			return await this.SendAsync(redirectRequest, cancellationToken);
		}

		return response;
	}
}
```

Basically whenever there's a Location header present we make a new request there and copy all the request options.
Note how I used `base.SendAsync` and `this.SendAsync` to go directly down the request pipeline or call this method recursively to allow for following multiple redirects.

{{< mermaid >}}
sequenceDiagram
    RequestMiddleware->>CookieMiddleware: .
    CookieMiddleware->>HttpClientHandler: Headers["Cookie"]
    HttpClientHandler->>Website: HTTP request
    Website-->>HttpClientHandler: 302 Headers["Location"]
    HttpClientHandler-->>CookieMiddleware: Headers["Set-Cookie"]
    CookieMiddleware-->>RequestMiddleware: .
    RequestMiddleware->>CookieMiddleware: GET redirected
    CookieMiddleware->>HttpClientHandler: Headers["Cookie"]
    HttpClientHandler->>Website: HTTP request
{{< /mermaid >}}

### Configuring the HttpClient
Finally we go to the `Configure` method to set up dependency injection.

```csharp
services.AddTransient<FollowRedirectsMessageHandler>();
services.AddTransient<CookieSessionMessageHandler>();

services.AddHttpClient(RequestBuilder.HttpClientName)
.ConfigurePrimaryHttpMessageHandler(() => new HttpClientHandler
{
    AllowAutoRedirect = false,
    UseCookies = false,
})
// Ordering of the handlers is important, from top (outermost) to bottom
.AddHttpMessageHandler<FollowRedirectsMessageHandler>()
.AddHttpMessageHandler<CookieSessionMessageHandler>();
```

We disable redirects and cookies on the `HttpClientHandler` because we add middleware to handle it instead.
