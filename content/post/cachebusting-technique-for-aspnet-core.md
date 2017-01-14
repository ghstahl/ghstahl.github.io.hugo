+++
date = "2017-01-14T14:09:44-08:00"
title = "Cachebusting technique for asp.net core"
draft = false

+++

## Goal

The goal of this exercise is to create a versioned route, as opposed to adding a query paramater to the end of the route.
It isn't simple browser cache busting we are after, but in later posts this technique will be used to feed a CDN.

The stock asp.net core script tag helper looks like this;
```
<script src="~/lib/jquery/dist/jquery.js"></script>
```

Our asp.net core script tag helper will look like this;
```
 <pingo-script-cbv src="~/lib/jquery/dist/jquery.js"></pingo-script-cbv>
```

The final outcome in HTML will look like this;
```
<script src="/cb-v/1.0.0.395078882/lib/jquery/dist/jquery.js"></script>
```

There are a couple things we need to do.

1. Create our TagHelper
2. Make a static FileProvider that serves the correct file.
3. Hook it up.


### The TagHelper

```
public class PingoScriptCbvTagHelper : TagHelper
{
    private static string _version;

    public static string Version
    {
        private get
        {
            if (string.IsNullOrEmpty(_version))
            {
                _version = "not-set";
            }
            return _version;

        }
        set { _version = value; }
    }

    private static string _cbcPrepend;
    private static string CBVPrepend
    {
        get
        {
            if (string.IsNullOrEmpty(_cbcPrepend))
            {
                _cbcPrepend = string.Format("/cb-v/{0}/", Version);
            }
            return _cbcPrepend;
        }
    }

    public override async Task ProcessAsync(TagHelperContext context,
                                            TagHelperOutput output)
    {
        output.TagName = "script";
        var src = output.Attributes["src"];
        var finalSrcValue = src.Value.ToString().Replace("~/", CBVPrepend);
        output.Attributes.SetAttribute("src", finalSrcValue);
    }
}
```
### The FileProvider

Our FileProvider is pretty simple.  It uses asp.net core's PhysicalFileProvider to do the heavy lifting, and only bothers to modify what gets sent as a subpath to the GetFileInfo method.
```
public class CbvPhysicalFileProvider : IFileProvider, IDisposable
{
    int NthOccurence(string s, char t, int n)
    {
        return s.TakeWhile(c => (n -= (c == t ? 1 : 0)) > 0).Count();
    }
    private PhysicalFileProvider PhysicalFileProvider { get; set; }

    public CbvPhysicalFileProvider(string root)
    {
        PhysicalFileProvider = new PhysicalFileProvider(root);
    }

    public IFileInfo GetFileInfo(string subpath)
    {
        var index = NthOccurence(subpath, '/', 2);
        var newSubpath = subpath.Substring(index);
        return PhysicalFileProvider.GetFileInfo(newSubpath);
    }

    public IDirectoryContents GetDirectoryContents(string subpath)
    {
        return PhysicalFileProvider.GetDirectoryContents(subpath);
    }

    public IChangeToken Watch(string filter)
    {
        return PhysicalFileProvider.Watch(filter);
    }

    public void Dispose()
    {
        PhysicalFileProvider.Dispose();
    }
}
```
### Hook it up
Please refer to your apps Startup.cs file.

```
public class Startup
{
    ...

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    public void Configure(  IApplicationBuilder app,
                            IHostingEnvironment env,
                            ILoggerFactory loggerFactory)
    {
        var version = typeof(Startup).GetTypeInfo()
                    .Assembly
                    .GetCustomAttribute<AssemblyInformationalVersionAttribute>()
                    .InformationalVersion;
        if (env.EnvironmentName == "Development")
        {
            version += "."+ Guid.NewGuid().ToString().GetHashCode();
        }
        PingoScriptCbvTagHelper.Version = version;
        ...
    }
    ...

}
```
