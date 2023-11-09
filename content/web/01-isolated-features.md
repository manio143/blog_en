---
title: 1. Isolated web features within a single ASP.NET Core service
images: []
---

# 1. Isolated web features within a single ASP.NET Core service

We've been discussion recently at work how we can increase our velocity for the team. I went to think and saw that a lot of new projects have a high upfront cost when it comes to provisioning the resources, setting up the build and deployment pipelines, etc. This is fine for longer running projects but would be delaying quick experiments much more than it should.

I came up with the question - is it possible to host multiple separate projects within a single ASP.NET Core service? This would make us go once through the initial setup, allow the team to create multiple experiments and when an experiment is deemed good for running long term it would get extracted into a new fully fledged service so that it can be scaled up.

The key thing I had to worry about here is making the features as independent from the host and each other as possible, while keeping the cost of adding a new experiment low. Interdependence could cause a lot of trouble during extraction.

I tried to see if there was anything already done for this topic under such terms as "multi-tenant", but the articles are generally focused on multiple tenants using the same app, vs hosting multiple tenant apps in the same service.

## Request pipeline and branching

ASP.NET Core allows you to configure branches in your request pipeline ([docs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-7.0#branch-the-middleware-pipeline)). Each branch can be running different middlewares, possibly configured differently. This is great because it means we can isolate the experiments - instead of them relying on the shared middleware setup, they need to explicitly write up their own request pipeline which can later be easily transferred during extraction.

The branching can either preserve tha path in the request or move the prefix by which we branched to the [path base](https://andrewlock.net/understanding-pathbase-in-aspnetcore/). Routing is applied on top of a path base. With manually mapped endpoints (e.g. using `MapGet()`) this can be used easily to isolate the path prefix from the paths in routes. However, my use case is about making it work with controllers. When you say `MapControllers()` it applies to all controllers known to the application. And unfortunately controllers are resolved at the host level, not at the request pipeline level. This means that if two features use the `/` route we can't really differentiate that across branches. Hence for me I had to set `preserveMatchedPathSegment: true` and add a prefix to each route in a given feature.

```csharp
// feature initializer
return appBuilder.Map(routePrefix, preserveMatchedPathSegment: true, app =>
{
  // configure feature specific pipeline
});
```

## Single startup

Something I had to discover - when using ASP.NET Core we don't get the same treatment of building request pipelines as we do with configuring services. Likely because order of adding distinct services doesn't matter, but order of adding middleware does. Therefore all middleware must be configured within a single Startup class.

In my case I wanted each feature to be independent and have it's own method for configuring the pipeline. I did a quick trick of creating a wrapper class that I can put in DI and then resolve in my Startup to apply feature specific services.

```csharp
internal class IsolatedFeatureInitializer
{
  public Action<IApplicationBuilder> Initializer { get; }

  public IsolatedFeatureInitializer(Action<IApplicationBuilder> initializer)
  {
    Initializer = initializer;
  }
}

// Startup
public void Configure(IApplicationBuilder app)
{
  // any middleware before will be executed for all requests
  IEnumerable<IsolatedFeatureInitializer> initializers = app.ApplicationServices.GetServices<IsolatedFeatureInitializer>();
  foreach (IsolatedFeatureInitializer initializer in initializers)
  {
    initializer.Initializer(app);
  }
  // any middleware after will be executed only for non-isolated requests
}
```

## Discovering controllers from other assemblies

My extension methods for feature management were in a project that didn't reference the features. As such the controllers were not auto-discovered and I had to add:

```csharp
services.AddMvcCore().AddApplicationPart(typeof(Feature).Assembly);
```

Reference: [When ASP.NET Core can't find your controller: debugging application parts](https://andrewlock.net/when-asp-net-core-cant-find-your-controller-debugging-application-parts/).

## Automatic route prefix for controllers and Swagger groups

I was able to use an MVC convention to modify all controllers in the feature assembly.

```csharp
services.AddMvcCore(c =>
{
  c.Conventions.Add(new IsolatedFeatureConvention(routePrefix, typeof(Feature).Assembly));
})

internal class IsolatedFeatureConvention : IActionModelConvention
{
  private readonly string m_featureName;
  private readonly Assembly m_sourceAssembly;

  public IsolatedFeatureConvention(string featureName, Assembly sourceAssembly)
  {
    m_featureName = featureName;
    m_sourceAssembly = sourceAssembly;
  }

  public void Apply(ActionModel action)
  {
    if (action.Controller.ControllerType.Assembly == m_sourceAssembly)
    {
      // apply group name to separate the actions for Swagger display
      // cannot start with / as it is removed from the swagger document url
      action.ApiExplorer.GroupName = m_featureName.TrimStart('/');

      // enforce a route prefix for the feature actions
      AttributeRouteModel routePrefix = new(new RouteAttribute(m_featureName));
      foreach (SelectorModel selector in action.Selectors)
      {
        if (selector.AttributeRouteModel != null)
        {
          // in order for this to work the controller cation cannot have a route prefix starting with '/' which is considered an override
          selector.AttributeRouteModel = AttributeRouteModel.CombineAttributeRouteModel(routePrefix, selector.AttributeRouteModel);
        }
        else
        {
          selector.AttributeRouteModel = routePrefix;
        }
      }
    }
  }
}
```

Further reading: [GroupName -> Swagger doc](https://github.com/domaindrivendev/Swashbuckle.AspNetCore/issues/562), [Swagger UI](https://github.com/domaindrivendev/Swashbuckle.AspNetCore#list-multiple-swagger-documents), if you want to include version: [this](https://github.com/dotnet/aspnet-api-versioning/wiki/API-Explorer-Options#format-group-name) and [this](https://github.com/dotnet/aspnet-api-versioning/issues/516).

## Isolated DI

Ideally the features would be completely separated with regards to dependencies. There are some things that need to be defined in the host (e.g. hosted services, healthchecks, controller configuration), but most dependencies will be needed in the context of requests. Luckily for us, the dependencies are resolved in layers. A middleware is created using dependencies resolved as the pipeline progresses to that middleware. Same for controllers. It means we can override the value of `HttpContext.RequestServices` with another service provider and any middleware further down in the pipeline will resolve its dependencies using the override.

```csharp
internal class IsolatedFeatureMiddleware
{
  private readonly RequestDelegate m_next;
  private readonly IServiceProvider m_featureServices;

  public IsolatedFeatureMiddleware(RequestDelegate next, IServiceProvider featureServices)
  {
    m_next = next;
    m_featureServices = featureServices;
  }

  public async Task InvokeAsync(HttpContext context)
  {
    IServiceProvider originalProvider = context.RequestServices;
    try
    {
      using IServiceScope scope = m_featureServices.CreateScope();
      context.RequestServices = scope.ServiceProvider;

      await m_next(context);
    }
    finally
    {
      context.RequestServices = originalProvider;
    }
  }
}
```

You may want to create links back to the host for singleton instances. The way I went about it is that I clone the `IServiceCollection` used to configure the host, while redirecting singleton references to the hosts `IServiceProvider` (except for open generic types).
