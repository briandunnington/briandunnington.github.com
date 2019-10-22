Title: Azure Functions: Route Priority
Date: 2019.08.01
Summary: Making Azure Functions route matching make more sense
Image: http://briandunnington.github.io/images/azure_functions.png

<div class="hero-unit">
<h1>Azure Functions: Route Priority</h1>
<p>Making Azure Functions route matching make more sense</p>
</div>

**[Updated][Update]**: Updated to handle recent changes in the Functions webhost code.

Although I continue to [profess][DynamicConnectionString] [my][FunctionProxies] [love][Alexa] for [Azure Functions][AzureFunctions], today we are going to discuss something that it doesn't do so good: route priority. Consider these two endpoints:

    /api/{userId}
    /api/ping

(I won't get into API design and RESTful practices or whether or not you should even have routes like this - but it comes up often enough that I am gonna talk about it anyway)

If you are used to most flavors of ASP.NET, you would rightly assume that a request for `/api/ping` would route to the second endpoint and `/api/sam123` would route to the first endpoint.

But if you set up these routes in Azure Functions, can you guess which function will get ran for each request? Trick question - the answer is *'it depends'*. When your code is loaded up, the Functions runtime locates all functions and then registers routes for each of them. Specifically, the [`InitializeHttpFunctions` in `HttpInitializationService`][InitializeHttpFunctions] does this:

    private void InitializeHttpFunctions(IEnumerable<FunctionDescriptor> functions, HttpOptions httpOptions)
    {
        //...
        foreach (var function in functions)
        {
            var httpTrigger = function.GetTriggerAttributeOrNull<HttpTriggerAttribute>();
            if (httpTrigger != null)
            {
                //...
                string route = httpTrigger.Route;

                if (string.IsNullOrEmpty(route) && !function.Metadata.IsProxy)
                {
                    route = function.Name;
                }

                WebJobsRouteBuilder builder = function.Metadata.IsProxy ? proxiesRoutesBuilder : routesBuilder;
                builder.MapFunctionRoute(function.Metadata.Name, route, constraints, function.Metadata.Name);
            }
        }
        //...
    }

The [`MapFunctionRoute` method in `WebJobsRouteBuilder`][MapFunctionRoute] creates the route and adds it to a `RouteCollection` of routes:

    var routes = new RouteCollection();
    _routes.Add(new Route(
        new RouteHandler(c => _handler.InvokeAsync(c, functionName)),
        name,
        template,
        new RouteValueDictionary(defaults),
        new RouteValueDictionary(constraints),
        tokens,
        _constraintResolver));

 The `RouteCollection` that is built just tries to match in the order that the routes are added. And since we don't know how `IEnumerable<FunctionDescriptor>` orders the functions, we can't be sure of the route order (empirical evidence suggests that the functions are loaded in alphabetical order, but I wouldn't rely on that fact)

 The problem is now obvious: a request for `/api/ping` might match the `/api/ping` route if it happens to get registered first, but it might instead match the `/api/{userId}` route (with a `{userId}` value of `ping`) if that happened to get registered first.

 Sadly, Azure Functions doesn't have a built-in good solution for this. One option suggested by the Functions team is to [use route constraints][RouteConstraintRecommendation] to make your routes unique. So if your `{userId}` values follow a certain pattern, you could potentially do something like this:

    /api/{userId:minlength(10)}

The full list of route constraints [can be found here][RouteConstraints]. That can work for simple cases, but it quickly gets unwieldy if you have non-trivial constraints. What would be better is if the Functions runtime just did what ASP.NET Core did and smartly figured out which route you really intended. And since ASP.NET Core is open source, we can go see just how they are doing that.

[In the `RouteTableFactory.cs` file][RouteComparison], we can see the `RouteComparison` method that compares two `RouteEntry` objects and smartly orders them based on segment length, which segments are literals vs. parameters, and which parameters have constraints on them. So could we use this in Azure Functions some how?

It turns out that it is not very easy to do this. Azure Functions does not expose any functionality to add middleware or hook into the request processing pipeline. However, the `HttpInitializationService` class above does give a clue - it takes a dependency on `IWebJobsRouter` which certainly sounds like something that might be useful. If we could get access to that, we could get a list of the registered routes and then perhaps re-order them.

## Attempt #1 - Custom IWebJobsRouter implementation ❌

My first thought was to simply create a new `CustomWebJobsRouter` class that inherited from `WebJobsRouter`, register this class with the DI system, and have my way with it. So I added the following to my `Startup.cs` file:

    builder.Services.AddSingleton<IWebJobsRouter, CustomWebJobsRouter>

Setting a break point in my `CustomWebJobsRouter` class showed that it was indeed being called. But suddenly all of my routing stopped working entirely (every request returned a 404 no matter what). Since the Functions runtime uses the Web Jobs SDK behind the scenes, I [tracked down this code][ApplicationServices] that showed that the `ApplicationServices` also registered `IWebJobsRouter`, but since they are not the same instance, it appeared to cause some kind of internal issue. In fact, if I changed my `Startup.cs` to register the built-in `WebJobsRouter` implementation, it still broke all routing. So trying to replace the `IWebJobsRouter` registration wasn't going to work.

## Attempt #2 - DI dependency ❌

Instead of trying to replace the `IWebJobsRouter` registration, I decided to try to get a reference to the native instance instead. I added a dependency on `IWebJobsRouter` to another class that I was using, and lo-and-behold, I was able to get access to the instance.

    public class MyClass(IWebJobsRouter router)
    {
        this.router = router; // yay! we have it
    }

At this point, I was able to grab the registered routes and re-order them to make more sense. Break out the champagne - I had done it! But upon further testing, I realized there was a small issue. My custom class was not instantiated until a request came in, at which time it re-ordered the routes. But that first request was still routed to the original old route. My class was not running until too late in the pipeline to affect that first request. I tried moving the dependency to a bunch of other classes, hoping to get it earlier in the pipeline. Nothing worked though - the first request would always get processed before the routes were updated.

## Attempt #3 & #4 - Getting desperate ❌

Arg! I was so close, but that first request was gnawing at me. I started trying more and more questionable ideas to try to get it to work. I wanted to try something like ASP.NET MVC's `RewritePath` but that doesn't exist in ASP.NET Core. I tried just modifying the `HttpContext.Request.Path` property directly, but it was still too late in the pipeline to have an effect.

I even thought of using an `HttpClient` to have the API call itself on startup to try to work around the issue. The idea was that the 'first request' would be the API itself, which could rewrite the routes and allow them to be correct for future requests. But this was entering pretty hacky territory, so I scrapped that idea and went and had dinner instead.

## Attempt #5 - Custom extension <strike>✔</strike> ❌

**NOTE**: This is no longer the best approach. See **[Update][Update]** below.

After a nice meal, I finally remembered that you can register an implementation of `IExtensionConfigProvider` to allow custom code to be run. Normally this is intended to be used to create custom bindings, but I decided to try to (ab)use it for my own purposes. The extension is constructed by DI, so it was easy to take a dependency on the `IWebJobsRouter` instance again. But now I had the opposite problem - the code was called *too* early and the routes were not yet set up at all.

But now I had access to the instance, and I knew the routes were *going* to be set up, so I set up a `Task.Delay` loop to keep checking until the routes were set (I hear you yelling 'HACK!' but I prefer to hear 'ingenious solution!'). Once the routes were set up, I was able to reorder them and confirm that they were correctly ordered before the first request ever comes in. All of the pieces were now available.

<a name="update"></a>

## ***Update***

After running this in production for awhile, I would sometimes notice a strange issue where the routes appeared to be reset back to their unordered state. Some deeper digging finally revealed what was going on.

## Attempt #6 - Update HttpInitializationService ❌

It turns out that `HttpInitializationService` that is actually responsible for initializing the `IWebJobsRouter` instance implements the `IHostedService` interface. That interface has `StartAsync` and `StopAsync` methods which appear to get called when the function goes idle and then restarts. Each time it did that, the `IWebJobsRouter.AddFunctionRoutes` got called again, which reset the routes. I updated the extension to grab the `HttpInitializationService` from the service provider and then replaced the internal `IWebJobsRouter` instance with a wrapper object that did the route reordering. This allowed me to remove the `Task.Delay` loop entirely and also solved the problem of routes being reset because I was able to intercept the `AddFunctionRoutes` call and reorder the routes no matter when it was called.

I ran this in production for about a month and it was rock solid until a few days ago...

## Attempt #7 - Listen to ApplicationStarted event ✔

A few days ago, I noticed that some of my functions appeared to no longer have their routes correctly ordered. Strangely, some machines appeared to still be correct but others did not. I went digging and found out that there had been a [recent change to how route initialization was done][WebScriptHostHttpRoutesManager]. The old `HttpInitializationService` was replaced with a new `WebScriptHostHttpRoutesManager` class. This class no longer implemented `IHostedService`, which was good news because that meant that the route ordering is no longer done every time the function goes idle and then restarts. But the bad news was that there was no longer a way to get a reference to the instance or override the dependency registration.

I finally stumbled upon [the `IApplicationLifetime.ApplicationStarted` property][ApplicationStarted]. This property gets called once, right after the service is full configured and initialized so it is the perfect hook for our logic. I wired up the route reordering to be triggered by this event and confirmed that it works with both the previous runtime and the new updated runtime.

## Putting it all together

Even though I now had the data I needed at the time that I needed it, it was still a bit of a challenge. The `IWebJobsRouter` does not expose its collection of routes, so I had to use a little reflection to access the private member that holds them. Once I had them, I used a modified[*][Comparison] version of the `RouteComparison` comparer to order them properly and then reset the collection value. Here is what the meat of my extension ended up looking like:

    public class RoutePriorityExtensionConfigProvider : IExtensionConfigProvider
    {
        IApplicationLifetime applicationLifetime;
        IWebJobsRouter router;

        public RoutePriorityExtensionConfigProvider(IApplicationLifetime applicationLifetime, IWebJobsRouter router)
        {
            this.applicationLifetime = applicationLifetime;
            this.router = router;

            this.applicationLifetime.ApplicationStarted.Register(() =>
            {
                ReorderRoutes();
            });
        }

        public void Initialize(ExtensionConfigContext context)
        {
        }

        public void ReorderRoutes()
        {
            var unorderedRoutes = router.GetRoutes();
            var routePrecedence = Comparer<Route>.Create(RouteComparison);
            var orderedRoutes = unorderedRoutes.OrderBy(id => id, routePrecedence);
            var orderedCollection = new RouteCollection();
            foreach (var route in orderedRoutes)
            {
                orderedCollection.Add(route);
            }
            router.ClearRoutes();
            router.AddFunctionRoutes(orderedCollection, null);
        }

        static int RouteComparison(Route x, Route y)
        {
            var xTemplate = x.ParsedTemplate;
            var yTemplate = y.ParsedTemplate;

            for (var i = 0; i < xTemplate.Segments.Count; i++)
            {
                if (yTemplate.Segments.Count <= i)
                {
                    return -1;
                }

                var xSegment = xTemplate.Segments[i].Parts[0];
                var ySegment = yTemplate.Segments[i].Parts[0];
                if (!xSegment.IsParameter && ySegment.IsParameter)
                {
                    return -1;
                }
                if (xSegment.IsParameter && !ySegment.IsParameter)
                {
                    return 1;
                }

                if (xSegment.IsParameter)
                {
                    if (xSegment.InlineConstraints.Count() > ySegment.InlineConstraints.Count())
                    {
                        return -1;
                    }
                    else if (xSegment.InlineConstraints.Count() < ySegment.InlineConstraints.Count())
                    {
                        return 1;
                    }
                }
                else
                {
                    var comparison = string.Compare(xSegment.Text, ySegment.Text, StringComparison.OrdinalIgnoreCase);
                    if (comparison != 0)
                    {
                        return comparison;
                    }
                }
            }
            if (yTemplate.Segments.Count > xTemplate.Segments.Count)
            {
                return 1;
            }
            return 0;
        }
    }

    public static class IWebJobsRouterExtensions
    {
        public static List<Route> GetRoutes(this IWebJobsRouter router)
        {
            var type = typeof(WebJobsRouter);
            var fields = type.GetRuntimeFields();
            var field = fields.FirstOrDefault(f => f.Name == "_functionRoutes");
            var functionRoutes = field.GetValue(router);
            var routeCollection = (RouteCollection)functionRoutes;
            var routes = GetRoutes(routeCollection);
            return routes;
        }

        static List<Route> GetRoutes(RouteCollection collection)
        {
            var routes = new List<Route>();
            for (var i = 0; i < collection.Count; i++)
            {
                var nestedCollection = collection[i] as RouteCollection;
                if (nestedCollection != null)
                {
                    routes.AddRange(GetRoutes(nestedCollection));
                    continue;
                }
                routes.Add((Route)collection[i]);
            }
            return routes;
        }
    }

To hide the complexity and reflection bits, I created a small extension method so you can just use this one line instead:

    builder.AddRoutePriority();

With that one line, now your functions will have the same sane and expected routing as ASP.NET Core.[*][Comparison]

<a name="comparison"></a>

## A note about route comparison

*To be completely honest, this routing is slightly modified from the ASP.NET Core version. If you look at the ASP.NET Core version, there is a descriptive comment that says:

    /// To calculate the precedence we sort the list of routes as follows:
    /// * Shorter routes go first.
    /// * A literal wins over a parameter in precedence.
    /// * For literals with different values (case insensitive) we choose the lexical order
    /// * For parameters with different numbers of constraints, the one with more wins

 So the first thing it does is compare routes based on length (number of segments) with the assumption that shorter routes are higher priority than longer routes. In practice, I found that length was not really the most important factor - my routing instead compares segment-by-segment and prefers literal segments over parameter-based segments. So where the ASP.NET Core routing would prioritize two routes like this:

    /api/{userId}
    /api/tools/ping

My routing would order them like this:

    /api/tools/ping
    /api/{userId}

Routes with literal segments are matched first because they are explicit routes that are definitive matches and the parameter-based routes are only tried after the literal routes.

## Show me the code

Here is the complete thing that you can copy and paste to check it out. Just declare your functions as normal, add `builder.AddRoutePriority()` to your `Startup.cs` and enjoy 🍻

<script src="https://gist.github.com/briandunnington/10e83919272bd6564ed95edbced77610.js"></script>


[DynamicConnectionString]: /azure_functions_dynamic_connection_string
[FunctionProxies]: /azure_functions_proxies
[Alexa]: /use_azure_functions_with_alexa
[AzureFunctions]: https://azure.microsoft.com/en-us/services/functions/
[InitializeHttpFunctions]: https://github.com/Azure/azure-functions-host/blob/2986ab411cc1addd8cea77dd9e2c6ddc438bc3ae/src/WebJobs.Script.WebHost/HttpInitializationService.cs#L49
[MapFunctionRoute]: https://github.com/Azure/azure-webjobs-sdk-extensions/blob/12f060f59d210a4d4892fa85f1a63a645edeba18/src/WebJobs.Extensions.Http/Routing/WebJobsRouteBuilder.cs#L49
[RouteConstraintRecommendation]: https://github.com/MicrosoftDocs/azure-docs/issues/11755#issuecomment-405651650
[RouteConstraints]: https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-2.2#route-constraint-reference
[RouteComparison]: https://github.com/aspnet/AspNetCore/blob/2c3a44371a651e31d10ed1969b1f0a86335214ee/src/Components/Components/src/Routing/RouteTableFactory.cs#L113
[ApplicationServices]: https://github.com/Azure/azure-webjobs-sdk-extensions/blob/cdeb65faacf04c53845fb76c4b513c9998fc97de/src/WebJobs.Extensions.Http/HttpBindingApplicationBuilderExtension.cs#L34
[WebScriptHostHttpRoutesManager]: https://github.com/Azure/azure-functions-host/pull/5102/files
[ApplicationStarted]: https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.hosting.iapplicationlifetime.applicationstarted
[Comparison]: #comparison
[Update]: #update
