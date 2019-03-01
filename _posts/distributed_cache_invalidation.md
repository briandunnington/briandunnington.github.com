Title: Distributed Cache Invalidation using Azure Service Bus
Date: 2019.02.28
Summary: It has been said that there are only two hard problems in computer science: cache invalidation and naming things. Time to cross one of those off of the list
Image: http://briandunnington.github.io/images/azure_servicebus.png

<div class="hero-unit">
<h1>Distributed Cache Invalidation using Azure Service Bus</h1>
<p>It has been said that there are only two hard problems in computer science: cache invalidation and naming things. Time to cross one of those off of the list</p>
</div>

I am sure you have [heard this before][TwoHardThings] since it is one of the most repeated sayings in software development:

> There are only two hard things in Computer Science: cache invalidation and naming things.
> 
> -- Phil Karlton

I am no expert in naming things, so today we are going to focus on cache invaliation instead.

### Why cache invalidation is hard

If you are using an external cache like [Redis][], then cache invalidation is actually not hard at all. Since there is only one cache in a centralized spot and all clients use it as the source of truth, there is nothing to synchronize.

But sometimes an external cache is not the right answer. Sometimes the objects being cached are very large and still take a long time to transmit across the wire from the cache. Or the data might not be easily serializable. Or your system design doesn't easily allow the addition of a caching service. Whatever the reason, sometimes a local in-memory cache is the solution. The data can be fetched instantly and contain any kind of complex object.

But in any kind of distributed system (like a website running across multiple servers in the cloud), each machine will have their own copy of the cached data, and thus it can get out of sync. If the data is changed on one server, how can the other servers be aware of the change so that they don't serve stale data?

### Getting the word out

What we need is a way for one server to let all of the others know about the change. If you have a static and finite amount of other machines in your farm, you might be able to simply send a message to each one instructing them to invalidate the cached item. But in today's cloud-first world, you likely don't even know which other machines are running your code. And maybe you even have auto-scaling turned on so you don't even know *how many* other machines are running your code.

So we need a way to send a message out and let any number of other unknown listeners receive the message. This is a [classic pub/sub scenario][PubSub] which allows decoupling of the message sending (publishing) from the message receiving (subscribing).

### Azure Service Bus topics and subscriptions

There are lots of ways to implement a pub/sub system, but one of the easiest (IMHO) is to leverage the [Topics and Subscriptions feature of Azure Service Bus][Topics]. A topic is like a normal message bus queue that can receive messages. But unlike a classic queue where each message is consumed only once, a topic can broadcast a copy of the message to any number of subscriptions.

For our implementation, each machine in the farm will register itself as a subscriber. This happens automatically, so even if your site is scaled up and a bunch of new machines come online, you don't have to worry about tracking who needs to get the messages. 

When a change is made on one machine, it publishes a message to the topic that says 'item X in cache Y has changed'. Any machines that have registered as subscribers will get a copy of that message and will know that they need to invalidate that item in their copy of the cache as well.

### SynchronizedCache implementation

So let's build out an implementation of these ideas. The `SynchronizedCache` class will provide the usual caching features but also support this distributed invalidation by using Azure Service Bus.

The actual cache is just a simple `ConcurrentDictionary` that stores objects by key. When they are requested, they are served from the cache if available, or they are loaded from the source automatically. Each implementation of `SynchronizedCache` must implement the `Load()` method with the app-specific logic for fetching the data from the source when it is not found in the cache. So far, this is just like a typical in-memory cache.

    public Task<TValue> GetAsync(TKey key)
    {
        return cache.GetOrAdd(GetHashKey(key), (k) => Load(key));
    }

    protected abstract Task<TValue> Load(TKey key);

But when the cache is first used, it checks the Azure Service bus to see if a subscription is already registered for this machine. If not, it automatically registers a subscription. The subscription also sets a filter rule so that it only gets messages for this specific type of cache (which allows you to have several differnt types of caches in your app and only require one single topic). 

    if (!await serviceBusManagementClient.SubscriptionExistsAsync(topicClient.Path, subscriptionName))
    {
        var cacheTypeRule = new RuleDescription()
        {
            Name = "CacheTypeRule",
            Filter = new CorrelationFilter() { ContentType = cacheType }
        };
        var subscriptionDescription = new SubscriptionDescription(topicClient.Path, subscriptionName) { AutoDeleteOnIdle = TimeSpan.FromDays(1), DefaultMessageTimeToLive = TimeSpan.FromMinutes(15) };
        subscriptionDescription = await serviceBusManagementClient.CreateSubscriptionAsync(subscriptionDescription, cacheTypeRule);
    }
    this.subscriptionClient = new SubscriptionClient(topicClient.ServiceBusConnection, topicClient.Path, subscriptionName, ReceiveMode.PeekLock, RetryPolicy.Default);

An event listener is also set up so that any incoming messages can be handled:

    subscriptionClient.RegisterMessageHandler(ProcessMessageAsync, messageHandlerOptions);

The neat part is that when `InvalidateAsync()` is called, a message is sent to the topic that contains the cache type and the key that is being invalidated:

    public Task InvalidateAsync(TKey key)
    {
        var message = new Message();
        message.ReplyTo = subscriptionName;
        message.Label = GetHashKey(key);
        return topicClient.SendAsync(message);
    }

The topic receives the message and distributes it to all subscribers. Each subscriber is notified and the `ProcessMessageAsync` event gets triggered. The receiving machine can see which key was invalidated and invalidate its own copy as well.

    async Task ProcessMessageAsync(Message message, CancellationToken token)
    {
        var lockToken = message.SystemProperties.LockToken;
        try
        {
            var key = (message.Label);
            cache.TryRemove(key, out _);
        }
        catch
        {
        }
        await subscriptionClient.CompleteAsync(lockToken);
    }

The subscriptions are set up to have a short TTL (currently 1 day). This means that as long as someone is listening for messages, the subscription will remain, but once the listener is removed (perhaps your app auto-scaled back down and deprovisioned some of the web servers), the subscriptions will automatically delete themselves. That means that as machines come online, they automatically start listening for messages they care about, and when they go offline, the subscriptions automatically clean themselves up - no muss, no fuss!

*(Of course you could build out more advanced functionality for the caching portion: things like absolute or sliding expriation time, dependency invalidation, etc are easy to add, but they are just normal in-memory cache features and not tied to the distributed invalidation so I left them out for clarity.)*

### Wrap up

The source code for the `SynchronizedCache` class and an example project [are available on GitHub][SynchronizedCache]. Try it out and then [shoot me a message on Twitter][Twitter] to let me know if you are using this approach.

As for naming things: that is truly a problem that nobody will ever solve 🤣

[TwoHardThings]: https://martinfowler.com/bliki/TwoHardThings.html
[Redis]: https://redis.io/
[PubSub]: https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern
[Topics]: https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-queues-topics-subscriptions#topics-and-subscriptions
[SynchronizedCache]: https://github.com/briandunnington/SynchronizedCache
[Twitter]: https://twitter.com/briandunnington
