---
title: 4. CQRS with OData and Wolverine
images: []
---

# 4. CQRS with OData and Wolverine

_⚠️NOTE: this post in unfinished, but I decided to push it, so I don't loose it._

As I started to think about how I want to design Excos, I came to a conclusion that a modular monolith with an old school backend rendered UX and a public web API for programmatic access, all using event driven architecture with event sourcing, is something I want to pursue.

Why? The monolith, because there no reason not to at this stage of the project. The server side rendered HTML, because I want to do frontend development differently than I've experienced at work so far (and will write in the future more about it). A web API is just convenient for custom integrations. And event driven architecture to enable further integration points.

I decided to use modern [Blazor](https://learn.microsoft.com/en-us/aspnet/core/blazor/?view=aspnetcore-8.0) static server rendering with focus on [Web Components](https://developer.mozilla.org/en-US/docs/Web/API/Web_components), [OData](https://learn.microsoft.com/en-us/odata/webapi-8/overview) for shaping the API and enabling rich queries, [Wolverine](https://wolverinefx.net/) for event mediating and [Marten](https://martendb.io/) for storage on top of Postgresql.

In this article I will dive a bit deeper into how it all fits together.

## Resources

I really like some of the ways how Microsoft Azure works.
There's a clear hierarchy of a subscription, then resource group, then resource provider, then resource, then sub-resource, etc.

Anything that's part of my system is going to be defined as a resource within a hierarchy.
And as I want to make my code modularized, each module can be thought of as a resource provider.

* At the root we have `organizations` (tenants).
    * A tenant has `identities` that is users, clients and groups, with their `settings` and `permissions`.
        * The settings collection will have instances of sections that can be provided by a module.
        * The permissions collection will list the resources the identity can access and with what roles (RBAC).
    * A tenant has `projects` which are units of configuration for the subsystems.
        * Again there's `settings` with sections defined by the modules.
        * And further resources which are either collections or singletons, as provided by the modules.

The concept of collections and singletons comes from OData and is a very intuitive way to think about a resource hierarchy.
Each of those defines a class of object types - e.g. `identities` collection can have both objects of type `user` or `client` which may have shared or distinct properties.

## Command and Query Responsibility Segregation

In short CQRS states that systems should embrace a difference between commands (updates or changes) and queries (reading system state).
This means that we can split our data models into two: a write model and a read model.

We can then take it a step further to Event Sourcing, where the state of your system is represented through events.
It allows us to describe changes precisely, keep a log of all changes to the objects, and compute a read-optimized view based on the events.

Wolverine and Marten enable a really nice event driven architecture, where the state is represented by the events that have occurred and other modules can subscribe to those events to create integrations with low coupling.
The objects exposed through the API would be aggregates of event streams.

So when using OData for the web API layer, we'll be leveraging the GET requests with built-in filtering and selecting for queries (Marten supports `IQueryable` interface operations), while for commands, instead of allowing PATCH calls to modify the entity as a whole, we will leverage the declarations of actions (POST `/entity/action()`) for command execution.

Now, it's very important to strike a balance between very specific commands and more generalized ones. As I approach modeling the operations users would invoke on the entities in my system, I realized that some of the operations would be best consolidated under a single `Edit` event which carries with it the latest representation of the entity. This makes sense for scenarios where we don't have a specific set of state transitions which are closely represented by the events.

To make it more clear, let's look at a `FeatureRollout` entity, which has a `State` property deciding if that rollout is actively provided to clients, and such properties as targeting rules and configuration for the client. The following diagram can show small discrete events manipulating the `State`, while all other properties would be bulk updated with an `Edit` event as it would not make sense to have 2 separate events, one for editing targeting rules and one for editing configuration.

{{< mermaid >}}
flowchart LR
    A((New)) -->|Start| B((Running))
    A -->|Archive| D
    B -->|Stop| C((Stopped))
    C -->|Start| B
    C -->|Archive| D((Archived))
{{< /mermaid >}}

## OData setup

The key part here is getting the EDM model right for the resource hierarchy. To do this I will leverage "Contained" entities.

I have looked into generating the EDM model and the controllers from event aggregates and commands, but decided against it for two reasons: public service surface can change differently than internal models, and the complexity of writing robust code generating isn't worth it when the code is very simple boilerplate (which also can be generated with gen AI).

I will also note that I will use [Asp.Versioning](https://github.com/dotnet/aspnet-api-versioning) to provide evolution support for the OData APIs and leverage their improved support for ApiExplorer for generating an OpenAPI document. From what I understand, the API versioning integrating with OData is implemented in a way which kind of assumes the whole API surface is versioned in order to correctly provide a full EDM metadata document. Because of this it makes sense to use URL segment versioning. 

Based on the following examples: [Ex1](https://github.com/dotnet/aspnet-api-versioning/tree/main/examples/AspNetCore/OData/ODataOpenApiExample), [Ex2](https://github.com/RicoSuter/NSwag/tree/master/src/NSwag.Generation.AspNetCore.Tests.Web)
DRAFT:

```csharp
builder.Services.AddControllers()
                .AddOData(options =>
                {
                    options.Count().Select().OrderBy().Filter().Expand();
                });
builder.Services.AddProblemDetails();
builder.Services.AddApiVersioning(options =>
{
    // version config
    options.AssumeDefaultVersionWhenUnspecified = false;
    options.ReportApiVersions = true;

    // use URL segment
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
})
.AddOData(options => 
{
    // separate data and management routes
    options.AddRouteComponents("api/mgmt");
    options.AddRouteComponents("api/data");
})
.AddODataApiExplorer(options => {
    // allow for v1, v2 etc format: 'v'major[.minor][-status]
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// NSwag
services.AddOpenApiDocument(document => 
{
    document.DocumentName = "v1";
    document.ApiGroupNames = ["v1"];
});

app.UseVersionedODataBatching();
app.UseOpenApi();

if (app.Environment.IsDevelopment())
{
    // Access ~/$odata to identify OData endpoints that failed to match a route template.
    app.UseODataRouteDebug();
    // Access ~/swagger for UI view of the OpenApi definitions
    app.UseSwaggerUi3();
}
```

With the above configuration out of the way, let's look at a snippet of the model and interaction with Wolverine.

opts.Projections.Snapshot<Project>(SnapshotLifecycle.Inline);

```csharp
public record class ProjectCreated(Guid ProjectId, string ProjectName);
public class Project(Guid Id, string ProjectName)
{
    [JsonIgnore] // navigation property for OData $expand queries
    public Configuration Configuration => CurrentSession.LoadProjectConfiguration(this.ProjectName);

    public List<ProjectSettings> Settings { get; set; }

    public static Project Create(ProjectCreated created) => new(created.ProjectId, created.ProjectName);
}

public class ProjectModelConfiguration : IModelConfiguration
{
    public void Apply(ODataModelBuilder builder, ApiVersion apiVersion, string routePrefix)
    {
        if (routePrefix != "api/mgmt") return;
        
        var projects = builder.EntitySet<ProjectModel>("Projects").EntityType;
        projects.HasMany(p => p.Settings).Contained();
    }
}

[ApiVersion(1.0)]
[Route("api/mgmt/{v:apiVersion}/projects")]
public class ProjectsController(IDocumentSession session) : ODataController
{
    [EnableQuery]
    [ProducesResponseType( typeof( ODataValue<IEnumerable<ProjectModel>> ), Status200OK )]
    public IActionResult Get() => Ok(session.Query<Project>());

    [EnableQuery]
    [ProducesResponseType( typeof( ODataValue<ProjectModel> ), Status200OK )]
    [ProducesResponseType( typeof( ProblemDetails ), Status404NotFound )]
    public IActionResult Get(string name)
    {
        var project = session.Query<Project>().FirstOrDefault(p => p.ProjectName == name);
        if (project is null)
        {
            return NotFound();
        }

        return Ok(project);
    };
}
```