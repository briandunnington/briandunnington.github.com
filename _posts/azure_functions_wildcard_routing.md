Title: Azure Functions: Wildcard Routing
Date: 2020.03.19
Summary: A quick tip to support wildcard routes in Azure Functions
Image: http://briandunnington.github.io/images/azure_functions.png

<div class="hero-unit">
<h1>Azure Functions: Wildcard Routing</h1>
<p>A quick tip to support wildcard routes in Azure Functions</p>
</div>


I have been using and writing about [Azure Functions][AzureFunctions] quite a bit lately, including:

- [Making Azure Functions route matching make more sense][RoutePriority]
- [How to set the Cosmos DB trigger connection string at runtime][DynamicConnectionString]
- [The many uses of Azure Functions Proxies][FunctionProxies]

Today's tip is a quick and easy one. Some times you want to handle a bunch of routes with a single function. Maybe you are building a pass-thru to call another API. You might be able to use Azure Functions Proxies, but if you need to execute custom code, that option is out. What you need is a way to set up a wildcard route in your HTTP trigger.

The [docs][RouteParameters] say "You can use any [Web API Route Constraint][WebApiRouteConstraint] with your parameters". [*Side note*: There are lots of cool things you can do with route parameters like using regular expressions, optional parameters, and default values - check them all out]. But one thing you *won't* see documented there is anything about wildcard parameters.

But it turns out that they are supported if you know the syntax. And the syntax is [the same one that is used by Proxies][ProxySyntax]: `/api/{*restOfPath}`. The docs say: "If the route template terminates in a wildcard, such as `/api/{*restOfPath}`, the value `{restOfPath}` is a string representation of the remaining path segments from the incoming request."

So you can use this same technique in your HTTP trigger binding set up:

    [FunctionName("WildcardExample")]
    public async Task WildcardExample([HttpTrigger(AuthorizationLevel.Anonymous, "get", 
                                        Route = "{*restOfPath}")]) string restOfPath)
    {
        // use restOfPath in your function here
    }

I saw a few questions about this on the internet so I thought I would share this simple solution.


[RoutePriority]: /azure_functions_route_priority
[DynamicConnectionString]: /azure_functions_dynamic_connection_string
[FunctionProxies]: /azure_functions_proxies
[AzureFunctions]: https://azure.microsoft.com/en-us/services/functions/
[RouteParameters]: https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=csharp#using-route-parameters
[WebApiRouteConstraint]: https://docs.microsoft.com/en-us/aspnet/web-api/overview/web-api-routing-and-actions/attribute-routing-in-web-api-2#constraints
[ProxySyntax]: https://docs.microsoft.com/en-us/azure/azure-functions/functions-proxies
