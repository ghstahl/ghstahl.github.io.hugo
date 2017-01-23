+++
title = "CDN Origin Server Redirector Technique"
draft = false
date = "2017-01-21T08:51:52-08:00"
tags = ["ASPNET Core","CDN"]
categories = ["ASPNET Core"]
slug = ""

+++
## Pre Reading
This is a continuation on the previous [post](/post/simple-redirector-for-aspnet-core/) on our simple redirector technique.


## Problem
I had the joy of setting up a CDN configuration in Akamai.  Though it wasn't that bad, the rarity of that task made me never want to do it twice.
I only really needed to CDN cache static assets, and chose to harden the rules for anyone that wanted to use my configuration.

## The CDN Configuration Rules.
1. I am going to set the CDN time to live for 1 year
2. I will NOT be clearing your CDN cache just because you made a mistake
3. Our CDN will be configured to chase redirects.  4 deep in this case.

## Story Requirement



**I would like to be able to edge cache in the following way.**  

```
https://{mycdn.akamai.com}/cdn/{key}/{my-everything-else}
    would eventually find its way via redirects to   
https://{my origin server endpoint}/{my-everything-else} 
```  

**I would like to be able to edge cache in the following way, but with a throw-away version scheme.**

```
https://{mycdn.akamai.com}/cdn-v/{version}/{key}/{my-everything-else}
    would eventually find its way via redirects to   
https://{my origin server endpoint}/{my-everything-else} 
```

## Implementation
This is built upon the previous simple redirector work.

**ASP.NET Core URL Rewriter Rules**  
```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="CDN Cache Buster V">
          <match url="^cdn-v/([_0-9a-zA-Z-]+)/([_0-9a-zA-Z-]+)/(.*)$" />
          <action type="Rewrite" url="SimpleRedirector/Home/Index?key={R:2}&amp;remaining={R:3}" />
        </rule>
        <rule name="CDN">
          <match url="^cdn/([_0-9a-zA-Z-]+)/(.*)$" />
          <action type="Rewrite" url="SimpleRedirector/Home/Index?key={R:1}&amp;remaining={R:2}" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
``` 
**Hooking up URL Rewrite**  
 
```c#
public void Configure(  IApplicationBuilder app, 
                        IHostingEnvironment env, 
                        ILoggerFactory loggerFactory)
{
    ...  

    var root = env.ContentRootFileProvider;
    var rewriteOptions = new RewriteOptions()
        .AddIISUrlRewrite(root, "IISUrlRewrite.config");
    app.UseRewriter(rewriteOptions);

    ...
}
```

## Finally
In summary, any CDN provider you choose needs to be able to chase redirects.  
Also, this technique states that versioning is the problem of the origin servers and shall not be fixed or accounted for by the CDN providers.
This will make sure that your CDN configuration doesn't get out of control, especially if you need to swap out providers.
 Keep it simple!
 
 