+++
date = "2017-01-20T08:35:55-08:00"
tags = ["ASPNET Core"]
categories = ["ASPNET Core"]
slug = ""
title = "Simple Redirector for Asp.Net Core"
draft = false

+++

## Introduction

This is going to be a really simplistic implementation of a redirect, but it has served me for 99% of my needs.
This redirector will be leveraged later in a more complicated CDN implementation.  So stay tuned.

## Goal  
I would like to be able to have redirection happen by a simple key in a route.  
i.e.   
```
    https://{redirector endpoint}?  
        key={my-key}&
        remaining={url encoded my-final-route}
```
would redirect to  
```
    https://{my-key registered endpoint}/{my-final-route}  
```

## Implementation  

Registration for the redirector looks like the following;

**Our Model**

```
public class SimpleRedirectRecord
{
    public string Key { get; set; }
    public string BaseUrl { get; set; }
    public string Scheme { get; set; }
}
```  

**Our Configuration Store Interface**

```
public interface ISimpleRedirectorStore
{
    /// <summary>
    /// Finds the redirector record by key
    /// </summary>
    /// <param name="key"></param>
    /// <returns>null or RedirectRecord</returns>
    Task<SimpleRedirectRecord> FetchRedirectRecord(string key);
}    


class CInMemorySimpleRedirectStore : ISimpleRedirectorStore
{
    private List<SimpleRedirectRecord> _simpleRedirectRecords;

    private List<SimpleRedirectRecord> Records
    {
        get
        {
            if (_simpleRedirectRecords == null)
            {
                _simpleRedirectRecords = new List<SimpleRedirectRecord>()
                {
                    new SimpleRedirectRecord() 
                        {   BaseUrl = "www.google.com", 
                            Key = "google", 
                            Scheme = "https"},
                    new SimpleRedirectRecord() 
                        {   BaseUrl = "www.facebook.com", 
                            Key = "facebook", 
                            Scheme = "https"},
                    new SimpleRedirectRecord() 
                        {   BaseUrl = "www.microsoft.com", 
                            Key = "microsoft", 
                            Scheme = "https"}
                };
            }
            return _simpleRedirectRecords;
        }
    }

    public async Task<SimpleRedirectRecord> FetchRedirectRecord(string key)
    {
        var query = from item in Records
            where item.Key == key
            select item;
        return query.FirstOrDefault();
    }
}
```

**Our Controller**

```
[Area("SimpleRedirector")]
public class HomeController : Controller
{
    private ILogger Logger { get; set; }
    private readonly IHttpContextAccessor _httpContextAccessor;
    private ISession Session => _httpContextAccessor.HttpContext.Session;
    private ISimpleRedirectorStore _simpleRedirectorStore;

    public HomeController(IHttpContextAccessor httpContextAccessor,
        ISimpleRedirectorStore simpleRedirectorStore,
        ILogger<HomeController> logger)
    {
        _httpContextAccessor = httpContextAccessor;
        _simpleRedirectorStore = simpleRedirectorStore;
        Logger = logger;
    }

    public async Task<ActionResult> Index(string key, string remaining)
    {
        var record = await _simpleRedirectorStore.FetchRedirectRecord(key);
        if (record != null)
        {

            var requestScheme = _httpContextAccessor.HttpContext.Request.Scheme;
            string scheme = (record.Scheme) ?? requestScheme;

            var realUrl = string.Format("{0}://{1}/{2}", 
                scheme, record.BaseUrl, remaining);
            return new RedirectResult(realUrl);
        }
        return new NotFoundResult();
    }
}
```

**My Dependency Injection**  
I use autofac, so...
```
public class AutofacModule : Module
{
    protected override void Load(ContainerBuilder builder)
    {
        builder.Register(c => new CInMemorySimpleRedirectStore())
            .As<ISimpleRedirectorStore>()
            .SingleInstance();
    }
}
```

**Finally**

```
http://localhost:7791/SimpleRedirector/Home/Index?key=google&remaining=images/branding/product/ico/googleg_lodp.ico   

```
redirects to..    
```
https://www.google.com/images/branding/product/ico/googleg_lodp.ico  
```

