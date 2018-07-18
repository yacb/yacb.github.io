# Json Api DotNet Core
If you've ever had or wanted to implement a json api in dotnet core, I would recommend using the JsonApiDotnetCore framework.

Repository: https://github.com/json-api-dotnet/JsonApiDotNetCore

Documentation: https://json-api-dotnet.github.io/

It's dead simple to implement, has easily extendable layers, and takes away lots of frustration handling json in api format, not to mention if you wanted to keep your project in the .NET sphere.

### Models

Let's say you wanted to make an endpoint for a TRex model. To begin, models must inherit from `IIdentifiable<T>`. The framework does provide a base class to extend, `Identifiable<T>`, where `T` is the key type on the model. (Typically `int` or `Guid`). 

    public class TRex : Identifiable
    {
      [Attr("name")]
      public string Name { get; set; }

      [Attr("weight")]
      public double Weight { get; set; }

      [Attr("height")]
      public double Height { get; set; }

      [Attr("length")]
      public double Length { get; set; }
    }

It's as easy as that. It's worth noting here that `Identifiable` extends `Identifiable<int>` by default. The `[Attr]` attribute is used to designate you want the property serialized by JADNC with this name.

### Controllers

Now if you've ever made controllers before in ASP.NET or dotnet core you know how tedious and repetitive they can be. Luckily the framework includes a base controller that exposes every basic controller operation auto-magically for you. This even includes standard json api operations such as includes, pagination, filtering, comparing, and ordering on properties. The most basic implementation looks like this:

    public class TRexController : JsonApiController<TRex>
    {
      public class TRexController(
        IJsonApiContext context,
        IResourceService<TRex> service,
        ILoggerFactory loggerFactory)
      : base(jsonApiContext, service, loggerFactory)
      { }
    }

It's as simple as that.

### Middleware and Startup
To finish wiring it up we have to add the json api framework middleware to our `Startup.cs`

    public IServiceProvider ConfigureServices(IServiceCollection services)
    {
      services.AddDbContext<DinosaurContext>();

      services.AddJsonApi<DinosaurContext>();
    }

    public void Configure(IApplicationBuilder app)
    {
      app.UseJsonApi();
    }

### Resource Services vs Entity Repositories
If you noticed in the previous section we use a `DinosaurContext` where we presumably have a `DbSet<TRex>`. Something as simple as:

    public class DinosaurContext : DbContext
    {
      public DbSet<TRex> TRexs { get; set; }
    }

JsonApiDotnetCore will then read the entities defined in the context and build an appropriate context graph for any of it's properties and relationship.

But what if you don't want to use Entity Framework? Or what if you wanted to make a composite endpoint that returns a custom model constructed from multiple database models? This is where an `IResourceService` comes in. (Or any of the multitude of resource layers you can implement). For an example let's first make another model, TRexExhibit to keep our TRex's in.

    public class TRexExhibit : Identifiable
    {
      public double Length { get; set; }
      public double Width { get; set; }
      public string Information { get; set; }
      public FenceType FenceType { get; set; }
      [HasOne]
      public TRex TRex { get; set; }
    }

    public class TRex : Identifiable
    {
      ...
      [HasOne]
      public TRexExhibit TRexExhibit { get; set; }
      ...
    }

    ...

    public class DinosaurContext : DbContext
    {
      public DbSet<TRex> TRexs { get; set; }
      public DbSet<TRexExhibit> TRexExhibits { get; set; }
    } 

You may have noticed the `[HasOne]` attribute. This is so that Json Api Dotnet Core understands the relationship between the models for proper json serialization. 

Now, back to the service layer. Lets for example say that our frontend or consuming application wanted a nice bundled up call for a report that shows data about the relationship between a t-rex's size and the size of exhibit that they are in. We could implement a basic model as follows:

    public class ExhibitReportRecord : Identifiable
    {
      public double TRexLength { get; set; }
      public double TRexHeight { get; set; }
      public double ExhibitLength { get; set; }
      public double ExhibitWidth { get; set; }
    }

Since this is a composite type and we don't want to store it in the database, we won't be adding it to our `DbContext`. Instead, we'll define a resource service to the work for us. Depending on the operations that need to take place on your composite type endpoint, you may want to look at the service hierarchy located in the docs linked above. Since our model is only going to be used for a report, we know it will only be a 'get all' operation. As it turns out, JADNC has a service interface for just this instance, called the `IGetAllService`.