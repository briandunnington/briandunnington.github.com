Title: The many uses of Azure Functions Proxies
Date: 2018.03.25
Summary: Azure Functions Proxies are awesome - here are just a few ways to leverage them
Image: http://briandunnington.github.io/images/azure_functions.png

<div class="hero-unit">
<h1>The many uses of Azure Functions Proxies</h1>
<p>Azure Functions Proxies are awesome - here are just a few ways to leverage them</p>
</div>

<style>
.maincontent h4 { margin: 16px 0 8px 0; }
.img img { display: block; margin-bottom: 16px; border: solid 1px #666666; }
.img span { text-align: center; }
blockquote p { font-size: 90%; }
blockquote pre { margin: 10px 0; }
</style>

It is [no secret][AlexaFunctions] that I am loving [Azure Functions][AzureFunctions] lately. They are super easy to get started with, and they have great tooling and [local debugging][LocalDebugging] experience. Awhile back, the Azure team [announced Azure Functions Proxies][AnnounceProxies] which make Azure Functions even more useful. Let's look at just some of the cool uses:

### Creating stubs during development

This is a very common scenario: you start development of your new app and the front-end folks immediately need to set up the back-end API calls. Since the API is not built yet, you usually have to provide some kind of work-around with local data or mocked calls, but with Azure Functions Proxies, you can set up the real live production endpoint that accepts all of the actual parameters, but temporarily return hard-coded mock data.

<div class="img">
<img src="images/azure_functions_proxies_stub1.png"/>
</div>

<div class="img">
<img src="images/azure_functions_proxies_stub2.png"/>
</div>

This lets you seamlessly swap out the implementation later when the actual API logic is complete with *zero* changes to the front-end code (not even updating the url). And since Azure Functions Proxies come with [OpenAPI (Swagger)][OpenAPI] support out of the box, you can generate your API documenation up-front for an API-first approach.

<div class="img">
<img src="images/azure_functions_proxies_stub3.png"/>
</div>

### Testing error cases

Normally when testing an API consumer, testing special/edge/error cases can be challenging since there is no easy way to force these exceptional cases to occur. But following on with the proxies ability to return mock responses, you can easily set them up to return error responses as well.

<div class="img">
<img src="images/azure_functions_proxies_stub4.png"/>
</div>

<div class="img">
<img src="images/azure_functions_proxies_stub5.png"/>
</div>

### Present a unified API surface

Azure Functions Proxies can not only abstract third-party APIs, but also other Azure services. Many Azure services ([Logic Apps][LogicApps], [Azure Storage][AzureStorage], etc) already provide APIs for accessing them directly, but your code can become littered with a bunch of different urls to track. By putting up a proxy in front of those other services, they can appear as one unified API/endpoint.

Here is a function proxy that calls into an Azure Storage account to fetch an image to represent the current weather:

<div class="img">
<img src="images/azure_functions_proxies_unified1.png"/>
</div>

Note that the SAS querystring is handled by the proxy, so instead of having to call this from my app:

`https://weatherornotb4b8.blob.core.windows.net/images/sunny.png?sv=2015-04-05&st=2015-04-29T22%3A18%3A26Z&se=2015-04-30T02%3A23%3A26Z&sr=b&sp=rw&sip=168.1.5.60-168.1.5.70&spr=https&sig=Z%2FRHIX5Xcg0Mq2rqI3OlWTjEg2tYkboXr1P9ZUXDtkk%3D`

I can simply call this:

`https://weatherornot.azurewebsites.net/images/sunny`

And get this:

<div class="img">
<img src="images/azure_functions_proxies_unified2.png"/>
</div>

This also works for other Azure Functions, so if you have broken your logic into a microservice pattern, all of those functions can also appear as a single API.

### BONUS - Abstract & decouple third-party APIs (not yet available)

There is an [open feature request][Bonus] that would allow access to the backend response body JSON as first-class variables. Once that happens, it would open up a lot of interesting scenarios. One potential awesome use for Azure Functions Proxies would be to abstract integrations with third-party APIs. Let's pretend we were building a weather app and want to leverage an existing public API like [OpenWeatherMap][]. We could set up our proxy like this:

<div class="img">
<img src="images/azure_functions_proxies_bonus1.png"/>
</div>

But as time goes on, we decide we want to change our data source to the [Here Weather API][HereWeather]. The new API has a completely different response format, even though it represents roughly the same data. Instead of having to update every app, we can simply change the backing implementation since the proxy is acting as an abstraction.

<div class="img">
<img src="images/azure_functions_proxies_bonus2.png"/>
</div>

We can update the backend url and the response body transformation to map the new response data to our abstracted response, requiring aboslutely no changes to the app. By using the proxy, it decouples the app (API consumer) from the API implementation details. Win!


[AlexaFunctions]: use_azure_functions_with_alexa.html
[AzureFunctions]: https://azure.microsoft.com/en-us/services/functions/
[LocalDebugging]: https://medium.freecodecamp.org/serverless-doesnt-have-to-be-an-infuriating-black-box-b23cca2b2ba2
[AnnounceProxies]: https://blogs.msdn.microsoft.com/appserviceteam/2017/11/15/azure-functions-proxies-is-now-generally-available/
[OpenAPI]: https://github.com/OAI/OpenAPI-Specification
[OpenWeatherMap]: https://openweathermap.org/api
[HereWeather]: https://developer.here.com/api-explorer/rest/auto_weather/weather-observation-zipcode
[LogicApps]: https://azure.microsoft.com/en-us/services/logic-apps/
[AzureStorage]: https://azure.microsoft.com/en-us/services/storage/
[Bonus]: https://github.com/Azure/azure-functions-host/issues/1968
