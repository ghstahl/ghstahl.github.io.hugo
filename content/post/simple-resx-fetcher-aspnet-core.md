+++
draft = false
date = "2017-01-23T06:36:12-08:00"
tags = ["ASPNET Core","Localization"]
categories = ["ASPNET Core"]
slug = ""
title = "Simple Resx String Fetcher for Asp.Net Core"

+++

As I write this, it should be also filed under the **"Enough with the never ending REST apis"** Category.  This Api will eventually be implemented in my Facebook GraphQL work.  

## Background  
I wanted a single REST api that would be able to fetch all strings in a Resx for a given language.  Furthermore, I should be able to pick how those strings were to be packaged.  Some developers like object trees, others might need json arrays that look more like database records.  

For this project I would need to have a core way to read the data from Resx, and also provide developers the means to extend the treatment of the data prior to returning it.

## Data Treatment  
This is stock and trade interfaces and class implementations.  The treatments will evaluate a list of LocalizedString, and build a custom response.

**The Interface**
```c#
public interface ILocalizedStringResultTreatment
{
    object Process(IEnumerable<LocalizedString> resourceSet);
}
```  
**Some Implementations**
```c#
public class StringResourceSet
{
    public string Key { get; set; }
    public string Value { get; set; }
}

public class KeyValueArray: ILocalizedStringResultTreatment
{
    public object Process(IEnumerable<LocalizedString> resourceSet)
    {
        var result = (
                from  entry in resourceSet
                select new StringResourceSet { 
                    Key = entry.Name, 
                    Value = entry.Value })
            .ToList();
        return result;
    }
}

public class KeyValueObject: ILocalizedStringResultTreatment
{
    public object Process(IEnumerable<LocalizedString> resourceSet)
    {
        var expando = new System.Dynamic.ExpandoObject();
        var expandoMap = expando as IDictionary<string, object>;
        foreach (var rs in resourceSet)
        {
            expandoMap[rs.Name] = rs.Value;
        }
        return expando;
    }
}  
```
## Resources in Asp.Net Core
Resx in Asp.Net Core has changed a little bit, where you probably shouldn't create a open ended default resouce.
i.e. don't do Main.resx, but do do Main.en-US.resx.  in your startup configuration you call out that en-US is your default.

So what I did was create a bunch of Main.[culture].resx file, and also included the following model class in the same namespace as where my resx files live.
```cfml
public class Main
{

}
```
![ ](/images/project-resx.png)


## Startup.cs Hookup  
```c#
public void Configure(IApplicationBuilder app, 
    IHostingEnvironment env, 
    ILoggerFactory loggerFactory)
{
    ...
    var supportedCultures = new List<CultureInfo>
    {
        new CultureInfo("en-US"),
        new CultureInfo("en-AU"),
        new CultureInfo("en-GB"),
        new CultureInfo("es-ES"),
        new CultureInfo("ja-JP"),
        new CultureInfo("fr-FR"),
        new CultureInfo("zh"),
        new CultureInfo("zh-CN")
    };
    var options = new RequestLocalizationOptions
    {
        DefaultRequestCulture = new RequestCulture("en-US"),
        SupportedCultures = supportedCultures,
        SupportedUICultures = supportedCultures
    };
    app.UseRequestLocalization(options);
    ...
}
```

## Controller Implementation (REST)
As I stated earlier, I  am going to abandon REST and go with GraphQL, but for now this is what I got.
```c#
public static class ResourceApiExtensions
{
    public static int GetSequenceHashCode<T>(this IEnumerable<T> sequence)
    {
        return sequence
            .Select(item => item.GetHashCode())
            .Aggregate((total, nextCode) => total ^ nextCode);
    }
}

[Route("ResourceApi/[controller]")]
public class ResourceController : Controller
{
    private ILogger Logger { get; set; }
    private readonly IHttpContextAccessor _httpContextAccessor;
    private ISession Session => _httpContextAccessor.HttpContext.Session;
    private IStringLocalizerFactory _localizerFactory;
    private IRequestCultureProvider _requestCultureProvider;
    private readonly IStringLocalizer<ResourceController> _localizer;
    private IMemoryCache _cache;


    public ResourceController(IHttpContextAccessor httpContextAccessor,
        IMemoryCache memoryCache,
        IStringLocalizerFactory localizerFactory,
        ILogger<ResourceController> logger)
    {
        _httpContextAccessor = httpContextAccessor;
        _cache = memoryCache;
        _localizerFactory = localizerFactory;
        Logger = logger;
    }

    [Route("ByDynamic")]
    [Produces(typeof(object))]
    public async Task<IActionResult> GetResourceSet(string id, string treatment)
    {
        var rqf = Request.HttpContext.Features.Get<IRequestCultureFeature>();
        // Culture contains the information of the requested culture
        var currentCulture = rqf.RequestCulture.Culture;

        // Load Header collection into NameValueCollection object.
        var headers = _httpContextAccessor.HttpContext.Request.Headers;
        if (headers.ContainsKey("X-Culture"))
        {
            var hCulture = headers["X-Culture"];
            CultureInfo hCultureInfo = currentCulture;
            try
            {
                hCultureInfo = new CultureInfo(hCulture);
            }
            catch (Exception)
            {
                hCultureInfo = currentCulture;
            }
            currentCulture = hCultureInfo;
        }
        var key = new List<object> { currentCulture, id, treatment }
            .AsReadOnly()
            .GetSequenceHashCode();
        var newValue = new Lazy<object>(() => 
            { 
                return InternalGetResourceSet(id, treatment, currentCulture); 
            });


        var value = _cache.GetOrCreate(
            key.ToString(CultureInfo.InvariantCulture), 
            entry =>
            {
                entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromDays(100);
                return newValue;
            });

        var result = value != null ? value.Value : newValue.Value;
        return Ok(result);
    }


    private object InternalGetResourceSet(  string id, 
                                            string treatment, 
                                            CultureInfo cultureInfo)
    {
        try
        {
            var typeId = TypeHelper<Type>.GetTypeByFullName(id);
            if (typeId != null)
            {
                if (string.IsNullOrEmpty(treatment))
                {
                     treatment = "P7.Core.Localization.Treatment.KeyValueObject,P7.Core";
                }
                var typeTreatment = TypeHelper<Type>
                                        .GetTypeByFullName(treatment);
                var localizer = _localizerFactory.Create(typeId);

                var resourceSet = localizer.WithCulture(cultureInfo)
                                    .GetAllStrings(true);
                var instance = Activator.CreateInstance(typeTreatment) 
                                    as ILocalizedStringResultTreatment;
                var result = instance.Process(resourceSet);
                return result;
            }
        }
        catch (Exception e)
        {
            return "";
        }
        return "";
    }
}

```

## Usage

You basically pass 2 arguments.  

1. id = UrlEncoded(Fully qualifed class name of the resouce)  

2. treatment=UrlEncoded(Fully qualifed class name of the treatment implementation)  


You will notice that I pass in as the "id" my "Main" model.
```html
http://localhost:7791/ResourceApi/Resource/ByDynamic?id=p7.main.Resources.Main%2Cp7.main&treatment=P7.Core.Localization.Treatment.KeyValueArray%2CP7.Core
```
