Title: Durable Functions error - 'An EventStarted event is required'
Date: 2023.11.02
Summary: The solution to InvalidOperationException with 'An EventStarted event is required' when running or migrating Durable Functions to Isolated process
Image: http://briandunnington.github.io/images/eventstarted_event_required.png

<div class="hero-unit">
<h1>Durable Functions error - 'An EventStarted event is required'</h1>
<p>The solution to InvalidOperationException with 'An EventStarted event is required' when running or migrating Durable Functions to Isolated process</p>
</div>

We have been migrating our Azure Functions to the new [isolated process model](isolated) and for the most part, it has been fairly straight-forward. However, the most recent function app I was migrating used Durable Functions and I kept getting a strange error:

<br><img src="/images/eventstarted_event_required.png" width="100%"><br>

For those seaching, the full text is:

    An exception was thrown by the invocation.  
    Exception: System.InvalidOperationException: An ExecutionStarted event is required.  
        at DurableTask.Core.OrchestrationRuntimeState.GetExecutionStartedEventOrThrow()  
        in /_/src/DurableTask.Core/OrchestrationRuntimeState.cs:line 201


Of course I turned to Google:

<br><img src="/images/eventstarted_no_google.png" width="100%"><br>

You know it is going to be a fun debugging session when there are *zero* results on Google.

This is a big complicated function app, so I created a `File > New` Durable Function app from Visual Studio to see if I could reproduce the issue. Of course it ran fine out of the box. Now that I knew those functions themselves worked, I copied them into my actual function app, and when I ran it, I got the `An EventStarted event is required` error.

Like I said, this is a complex app with our own custom middleware and authentication and lots of stuff going on, so I assumed there was some strange incompatibility somehwere. I spent a ton of time trying to hunt down the issue thinking it had to do with how we were modifying the function results and possibly messing up the durable functions runtime somehow.

I started to slowly move some of our custom logic over to the working sample app, piece by piece, trying to see what finally broke it. Eventually, I copied over my custom `host.json` and suddently the sample app also started breaking!

Taking a look at the `host.json`, I saw this:

    //...snip...
    "extensions": {
        "http": {
            "routePrefix": "api"
        },
        "durableTask": {
            "extendedSessionsEnabled": true,
            "extendedSessionIdleTimeoutInSeconds": 30
        }
    }
    //...snip...

The sample app didnt include the `extendedSessions` stuff, but I recognized it from somewhere. In fact, the Azure Functions runtime emits this message when I ran the app:

<br><img src="/images/eventstarted_extendedSessions.png" width="100%"><br>

    Durable Functions does not work with extendedSessions = true for non-.NET languages. 
    This value is being set to false instead. 
    See https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-perf-and-scale#extended-sessions for more details.

Note that 'non-.NET languages' *includes* the Isolated model even if you are using .NET (it really means 'non-in-process model'). If you check out [the help link from the message](extendedSession), you can see this:

<br><img src="/images/eventstarted_notSupported.png" width="100%"><br>

So `extendedSessions` are not supported, and the output from the runtime says it is being set to `false`. But apparently, even when the runtime sets it to `false`, it still causes issues. I removed the `extendedSessions` config from my `host.json` file and my big complicated function app started working as expected 🎉

This took me a long time to hunt down so I wanted to share this with the world. Hopefully the next time someone searches, Google will have at least one result.

You can also [follow me on Twitter][briandunnington] if you want more esoteric Azure Functions tidbits.


[isolated]: https://learn.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-guide
[extendedSessions]: https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-perf-and-scale#extended-sessions
[briandunnington]: https://twitter.com/briandunnington
