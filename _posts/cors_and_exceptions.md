Title: CORS and exceptions on ASP.NET Core
Date: 2018.10.09
Summary: How to ensure your CORS headers are properly returned with HTTP error responses on ASP.NET Core
Image: http://briandunnington.github.io/images/aspnetcore.png

<div class="hero-unit">
<h1>CORS and exceptions on ASP.NET Core</h1>
<p>How to ensure your CORS headers are properly returned with HTTP error responses on ASP.NET Core</p>
</div>

<style>
.maincontent h4 { margin: 16px 0 8px 0; }
.img img { display: block; margin-bottom: 16px; border: solid 1px #666666; }
.img span { text-align: center; }
blockquote p { font-size: 90%; }
blockquote pre { margin: 10px 0; }
</style>

Setting up CORS in ASP.NET Core is [really easy and straight-forward][ConfigureCORS] with just a couple of lines of code. Once configured, the appropriate CORS headers will be returned with each response.

<div class="img">
<img src="images/cors_200yes.png"/>
</div>

Another nice thing about ASP.NET Core is the [easy exception handling][ConfigureExceptionHandling]. But, when used together, there can be an unexpected side effect:

<div class="img">
<img src="images/cors_500no.png"/>
</div>

Notice that the CORS headers are not present in the response in this case. Some exhaustive Googling turned up [this GitHub issue][RootCause] where the ASP.NET Core team essentially said that the exception handler was working as expected. [Their reasoning][Reason] was that, in the event of an unhanlded exception, the runtime should not leak any unintended information in the response, so the response body and headers are cleared out when the exception handler is invoked.

For many RESTful APIs though, the expectation may be that the browser displays a friendly error message in the event of unexpected exceptions. Due to the [same-origin policy][SameOrigin] though, the browser is prevented from access the error response if the CORS headers are not present.

### The Solution

Luckily, the world is filled with smart folks and somebody else had already [come up with a solution][Solution] to this problem. The CORS headers are normally added to the response early on in the pipeline, so the workaround is simply to make a copy of the CORS headers as early as possilbe, and then re-add them **after** the exception handler has ran and cleared them out.

I whipped up a simple middleware component to make it easy to reuse this behavior across all of our APIs. The CORS-related headers (they all start with `Access-Control-`) are saved into a temporary variable, and then the `Response.OnStarting()` callback is used to make sure they are added back to the response if necessary.

    using Microsoft.AspNetCore.Builder;
    using Microsoft.AspNetCore.Http;
    using System;
    using System.Threading.Tasks;

    namespace Examples.AspNetCore
    {
        public static class MaintainCorsExtension
        {
            public static IApplicationBuilder MaintainCorsHeadersOnError(this IApplicationBuilder builder)
            {
                return builder.Use(async (httpContext, next) =>
                {
                    var corsHeaders = new HeaderDictionary();
                    foreach (var pair in httpContext.Response.Headers)
                    {
                        if (!pair.Key.StartsWith("access-control-", StringComparison.InvariantCultureIgnoreCase)) { continue; }
                        corsHeaders[pair.Key] = pair.Value;
                    }

                    httpContext.Response.OnStarting(o => {
                        var ctx = (HttpContext)o;
                        var headers = ctx.Response.Headers;
                        foreach (var pair in corsHeaders)
                        {
                            if (headers.ContainsKey(pair.Key)) { continue; }
                            headers.Add(pair.Key, pair.Value);
                        }
                        return Task.CompletedTask;
                    }, httpContext);

                    await next();
                });
        }
        }
    }

To use the middleware, simply call it from your `Configure()` method in `Startup.cs`. Be sure to call this middleware early in the pipeline, right after the `.UseCors()` call, but before other middleware.

    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        app.UseCors();
        app.MaintainCorsHeadersOnError();
        //...configure the rest of your pipeline...
        app.UseMvc();
    }

Now when an exception is thrown and handled by the ASP.NET runtime, the CORS headers will be restored before the response is sent back.

<div class="img">
<img src="images/cors_500yes.png"/>
</div>

Now the broswer can access the response as expected and take the appropriate action. This is [supposed to be fixed in the next version of ASP.NET Core][FutureFix], but this does the trick in the meantime.



[ConfigureCORS]: https://docs.microsoft.com/en-us/aspnet/core/security/cors?view=aspnetcore-2.1
[ConfigureExceptionHandling]: https://docs.microsoft.com/en-us/aspnet/core/fundamentals/error-handling?view=aspnetcore-2.1
[RootCause]: https://github.com/aspnet/CORS/issues/46
[Reason]: https://github.com/aspnet/CORS/issues/46#issuecomment-157553623
[SameOrigin]: https://en.wikipedia.org/wiki/Same-origin_policy
[Solution]: https://github.com/aspnet/CORS/issues/90#issuecomment-348323102
[FutureFix]: https://github.com/aspnet/CORS/commit/554855cab34961a27a6cf248fbb847b9dd8bd8d4
