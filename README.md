# i18n
## Smart internationalization for .NET web apps
```
    PM> Install-Package I18N
```
### Introduction

The i18n library is designed to replace the use of .NET resources in favor of an easier, globally recognized standard for localizing web applications. Using this library simplifies localization by making it a first class citizen of views, controllers, and validation attributes.

### Features
- Globally recognized interface; localize like the big kids
- Localizes everything; views, controllers, validation attributes, and even routes!
- SEO-friendly; language selection varies the URL, and `Content-Language` is set appropriately
- Automatic; no routing changes required, just use an alias method where you want localization
- Smart; knows when to hold them, fold them, walk away, or run, based on i18n best practices

### Usage
To localize text in your application, use the `_("text")` alias method wherever required. That's it.
Here's an example of localizing text in a Razor view:

```html
    <div id="content">
        <h2>@_("Welcome to my web app!")</h2>
        <h3><span>@_("Amazing slogan here")</span></h3>
        <p>@_("Ad copy that would make Hiten Shah fall off his chair!")</p>
        <span class="button">
            <a href="@Url.Action("Plans", "Home", new { area = "" })">
                <strong>@_("SEE PLANS & PRICING")</strong>
                <span>@_("Free unicorn with all plans!")</span>
            </a>
        </span>
    </div>
```

And here's an example in a controller:

```csharp
    using i18n;
    
    namespace MyApplication
    {
        public class HomeController : I18NController
        {
            public ActionResult Index()
            {
                ViewBag.Message = _("Welcome to ASP.NET MVC!");

                return View();
            }
        }
    }
```

For use in URL-Helpers or other functions that require a plain string, you can use the `__("text")` alias:

```html
@Html.LabelFor(m => m.Name, __("First Name"))
```

For use in HTML attributes, you can use the `_("value", "attrname")` overload:

```html
    <img @_("Our logo", "alt") src="...">...</img>
```

And the same overload can be used for Javascript embedded into your Razor view:

```html
    <script type="text/javascript">
        $(function () {
            alert(@_("Hello world!", ""));
        });
    </script>
```

#### Installing a base WebViewPage for Razor
In the view example above, the `_("text")` alias is called on the base class of the Razor view page.
Depending on whether you're using the provided base classes or your own base class with `ILocalizing` (see below),
you'll want to change the `~/Views/web.config` file to point Razor to the base class containing the alias.
Here is how you'd set up the alias using the provided `I18NWebViewPage` class:

```xml
     <system.web.webPages.razor>
        <!-- ... -->
        <pages pageBaseType="i18n.I18NWebViewPage">
          <!-- ... -->
        </pages>
      </system.web.webPages.razor>
```

#### Using base classes vs. interfaces
The central service is `ILocalizingService`; anywhere you need localization, implement the `ILocalizing` interface.
The package comes with default base classes for convenience, including `I18NController`, `I18NWebViewPage`, and
`I18NWebViewPage<T>`. If your project needs prevent you from using a base class, implement `ILocalizing` and defer
to `ILocalizingService`; here is what implementing `ILocalizing` on a `Controller` might look like as a reference:

```csharp
    using System.Web.Mvc;
    using i18n;

    namespace MyApplication
    {
        public abstract class MyController : Controller, ILocalizing
        {
            private static ILocalizingService _service;

            protected MyController()
            {
                _service = new LocalizingService();
            }
            
            public string _(string text)
            {
                // Prefer a stored value to browser-supplied preferences
                var stored = LanguageSession.GetLanguageFromSession(ControllerContext.HttpContext);
                if (stored != null)
                {
                    return _service.GetText(text, new[] { stored });
                }

                // Find the most appropriate fit from the user's browser settings
                var languages = HttpContext.Request.UserLanguages;                
                return _service.GetText(text, languages);
            }
        }
    }
```

#### Building PO databases

To set up automatic PO database building, add the following post-build task to your project, after
adding `i18n.PostBuild.exe` as a project reference:

```
    "$(TargetDir)i18n.PostBuild.exe" "$(ProjectDir)"
```
    
After a successful build, this task will rip through your source code, finding everywhere you've used the `ILocalizing._("text")` alias, 
and uses this to build a master .PO template file located at `/locale/messages.pot` relative to your web application folder. After the
new template is constructed, any locales that exist inside the `/locale` folder are automatically merged with the template, so that
new strings can be flagged for further translation.

From here, you use any of the widely available PO editing tools (like [POEdit](http://www.poedit.net))
to provide locale-specific text and place them in your `/locale` folder relative to the provided language, i.e. `locale/fr`. 
If you change a PO file on the fly, i18n will update accordingly; you do _not_ need to redeploy your application.

#### Automatic routing

To participate in the automatic routing features of this library, call `I18N.Register()` in your startup code;
this will register a global filter and route decorator to provide the feature.

I18N comes with the ability to build on top of your existing routes to automatically redirect language choice to
an appropriate URL suffix. In Basic mode (i18n now supports an [Enhanced mode](#EM)), when a regular route is accessed, I18N will inspect the browser's `Accept-Language` header
to find the most appropriate choice of language from those you've prepared from PO files; for example, if the user's
most preferred language is `fr-CA` followed by `en-US`, and your application has PO files for `en-US` and `fr`, I18N 
will select `fr` as the best language choice. At this point, the regular route is then redirected to the language
specific URL, and the `Content-Language` header is set accordingly.

If your user is surfing in the `fr` locale, but then explicitly asks for a route with `en` appended, their preference
is persisted, and all subsequent requests will redirect to the `en` locale, making it simple to change language choice
for the user. You may also optionally use `/?language=fr` style query string parameters to elicit the same redirection
behavior. Route requests for languages you do not have resources for will _not_ redirect to a default resource, they
will 404 as expected.

To specifically exclude a route from this automatic behaviour, add a datatoken to it called 'nolocalize', for example:

```csharp
    routes.MapRoute(
        name: "SiteMapRoute",
        url: "sitemap.xml",
        defaults: new { controller = "SiteMap", action = "SiteMapXml" }
    ).DataTokens["nolocalize"] = 1;

```

#### <a id="EM"></a>Enhanced mode

i18n now supports an Enhanced mode or operation. This introduces an alternative to the original language selection,
routing, and matching algorithm.

First of all, the set of languages is established for which one or more translated messages exist.
Then, for each request, one of these languages is selected as the Principal Application Language (PAL) for the request.

The PAL is determined by the first of the following conditions that is met:

1. The path component of the URL is prefixed with a language tag that matches *exactly* one of the application languages. E.g. "example.com/fr/account/signup".

2. The path component of the URL is prefixed with a language tag that matches *loosely* one of the application languages.

3. The request contains a cookie called "i18n.langtag" with a language tag that matches (exactly or loosely) one of the application languages.

4. The request contains an Accept-Language header with a language that matches (exactly or loosely) one of the application languages.

5. The default application language is selected.

Where a *loose* match is made above, the URL is updated with the matched application language tag
and a redirect is issued. E.g. "example.com/fr-CA/account/signup" -> "example.com/fr/account/signup".

Language matching is performed when a list of one or more user-preferred languages is matched against
a list of one or more application laguages, the goal being to choose one of the application languages
which the user is most likely to understand.

The algorithm for this in i18n Enhanced mode is multi-facted and multi-pass and takes the Language, 
Script and Region subtags into account. Matching is performed once per-request to determine 
the principal language, and also once per GetText call. The multi-pass approach ensures a thorough attempt is made at matching a user's list of preferred 
languages (from their Accept-Language HTTP header). E.g. in the context of the following request:

```
User Languages: fr-CH, fr-CA  
Application Languages: fr-CA, fr, en
```

*fr-CA* will be matched first, and if no resource exists for that language, *fr* is tried, and failing
that, the default language *en* is fallen back on.

In recognition of the potential bottleneck of the GetText call (which typically is called many times per-request),
the matching algorithm is efficient for managed code (lock-free and essentially heap-allocation free).

To enable Enhanced mode in your project (the default mode being the original, Basic mode),
include the following in your Application_Start() method:

```csharp
    i18n.DefaultSettings.TheMode = i18n.DefaultSettings.Mode.Enhanced;
    i18n.I18N.Register();
```

Note that the following Chinese languages tags are normalized: zh-CN to zh-Hans, and zh-TW to zh-Hant.
It is still safe to use zh-CN and zh-TW, but internally they will be treated as equivalent to their new forms.

#### Validation attributes

Generally speaking, the stock validation attributes in ASP.NET MVC are closed to change, which makes it difficult to
use them with a framework like this. Still, I18N contains replacements for several `ValidationAttributes`, including
a base class for extending to build your own. These replacements function similarly to the originals, but use the
`ILocalizing` interface; just use normally and any derived text will pass through the localization process. Any custom
interactions that occur using these attributes elsewhere in the framework, however, will not work as expected. For the
most part, you should be able to swap `System.ComponentModel.DataAnnotations` and `i18n.DataAnnotations` namespaces
cleanly.

#### A reminder about folders in a web application

Your `locale` folder is exposed to HTTP requests as-is, just like a typical log directory, so remember to block all requests
to this folder by adding a `Web.config` file. 

```xml
    <?xml version="1.0"?>
    <configuration>    
        <system.web>
            <httpHandlers>
                <add path="*" verb="*" type="System.Web.HttpNotFoundHandler"/>
            </httpHandlers>
        </system.web>
        <system.webServer>
            <handlers>
                <remove name="BlockViewHandler"/>
                <add name="BlockViewHandler" path="*" verb="*" preCondition="integratedMode" type="System.Web.HttpNotFoundHandler"/>
            </handlers>
        </system.webServer>
    </configuration>
```

### Contributing
There's lot of room for further enhancements and features to this library, and you are encouraged to fork it and
contribute back anything new. Specifically, these would be great places to add more functionality:

* Help me fix the bugs! Chances are I don't ship in your language. Fix what hurts. Please?
* Better parsing and handling of PO files for more general purposes / outside editors
* Additional validation attributes (though these days I think this feature should be tossed)
* Generic handling that will work in ASP.NET Web API
* Support for additional storage mechanisms beyond ASP.NET Session (i.e. cookies, closures, etc.)