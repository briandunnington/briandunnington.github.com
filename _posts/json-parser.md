Title: WinRT JSON Parser
Date: 2013.01.09
Summary: A nicer way to deal with JSON in WinRT (Windows Store) apps

<div class="hero-unit">
<h1>WinRT JSON Parser</h1>
<p>The classes in Windows.Data.Json are no fun to use. This handy class makes it better.</p>
</div>

The new WinRT libraries generally make a lot of things easier, faster, and better, but one area that seems *harder* to work with is JSON data. The classes in the [Windows.Data.Json][WindowsDataJson] namespace are simple, but verbose to use. Consider this example, using the following simple JSON:

	[{
	    "identities": {
	        "google": {
	            "userId": "Google:101303958185943883774",
	            "accessToken": "ya29.AHES6ZRqPjKgoFjlp7FeJUK56ca19HzqIlzQQSLIj",
	            "isValid": true,
	            "age": 38
	        }
	    }
	}]

To parse this using the built-in classes, it might look like this:

<pre class="code"><code class="csharp"><span class="keyword">var</span> obj = <span class="keyword">new</span> Windows.Data.Json.JsonObject();
<span class="keyword">if</span> (Windows.Data.Json.JsonValue.TryParse(json, <span class="keyword">out</span> obj))
{
    <span class="keyword">var</span> identities = obj.GetArray()[0].GetObject();
    <span class="keyword">var</span> identity = identities[<span class="string">&quot;identities&quot;</span>].GetObject()[<span class="string">&quot;google&quot;</span>].GetObject();
    <span class="keyword">var</span> token = identity[<span class="string">&quot;accessToken&quot;</span>].GetString();
    <span class="keyword">var</span> isValid = identity[<span class="string">&quot;isValid&quot;</span>].GetBoolean();
    <span class="keyword">var</span> age = identity[<span class="string">&quot;age&quot;</span>].GetNumber();
}
</code></pre>

And if you add in checking to see if the properties exist first, the code is even longer. It isnt horrible, but it isnt how we like to deal with JSON normally.

There are great open-source tools out there like the venerable [Json.NET][] that make like easier. But sometimes taking a dependency on yet another third-party library isnt desirable or possible. Something so fundamental like JSON parsing should be as easy and painless as possible.

So what if you could just do this:

<pre class="code"><code class="csharp"><span class="keyword">dynamic</span> arr = JsonParser.Parse(json);
<span class="keyword">var</span> identity = arr[0].identities.google;
<span class="keyword">var</span> token = identity.accessToken;
<span class="keyword">var</span> isValid = identity.isValid;
<span class="keyword">var</span> age = identity.age;
</code></pre>

That looks nicer and much more inline with how JSON is normally treated in Javascript. The complete code for JsonParser is below - it is just a single small C# class that you can include in your project with no other dependencies. The secret sauce is in the `ConvertObject()` method; by leveraging a dynamic ExpandoObject, we can access the properties directly. The resulting object also provides a `Contains()` method so you can easily check for the existence of the property before accessing it.

I am finding this __much__ easier and more intuitive to use, so I thought I would share it. Let me know if you try it out and how you like it.

*JsonParser.cs source:*

<pre class="code"><code class="csharp"><span class="keyword">using</span> System;
<span class="keyword">using</span> System.Collections.Generic;
<span class="keyword">using</span> System.Threading.Tasks;
<span class="keyword">using</span> Windows.Data.Json;

<span class="keyword">namespace</span> Monsters.WindowsStore.Implementations
{
    <span class="keyword">public</span> <span class="keyword">class</span> JsonParser
    {
        <span class="keyword">public</span> <span class="keyword">static</span> Task&lt;<span class="keyword">dynamic</span>&gt; ParseAsync(<span class="keyword">string</span> json)
        {
            <span class="keyword">return</span> Task.Run(() =&gt;
                {
                    <span class="keyword">return</span> Parse(json);
                });
        }

        <span class="keyword">public</span> <span class="keyword">static</span> Task&lt;<span class="keyword">dynamic</span>&gt; ParseAsync(IJsonValue json)
        {
            <span class="keyword">return</span> Task.Run(() =&gt;
                {
                    <span class="keyword">return</span> Parse(json);
                });
        }

        <span class="keyword">public</span> <span class="keyword">static</span> <span class="keyword">dynamic</span> Parse(<span class="keyword">string</span> json)
        {
            <span class="keyword">var</span> j = Windows.Data.Json.JsonValue.Parse(json);
            <span class="keyword">return</span> Parse(j);
        }

        <span class="keyword">public</span> <span class="keyword">static</span> <span class="keyword">dynamic</span> Parse(IJsonValue json)
        {
            <span class="keyword">dynamic</span> obj = Convert(json);
            <span class="keyword">return</span> obj;
        }

        <span class="keyword">static</span> <span class="keyword">dynamic</span> Convert(IJsonValue json)
        {
            <span class="keyword">dynamic</span> obj = <span class="keyword">null</span>;
            <span class="keyword">switch</span> (json.ValueType)
            {
                <span class="keyword">case</span> JsonValueType.Array:
                    obj = ConvertArray(json.GetArray());
                    <span class="keyword">break</span>;
                <span class="keyword">case</span> JsonValueType.Boolean:
                    obj = json.GetBoolean();
                    <span class="keyword">break</span>;
                <span class="keyword">case</span> JsonValueType.Null:
                    obj = <span class="keyword">null</span>;
                    <span class="keyword">break</span>;
                <span class="keyword">case</span> JsonValueType.Number:
                    obj = json.GetNumber();
                    <span class="keyword">break</span>;
                <span class="keyword">case</span> JsonValueType.Object:
                    obj = ConvertObject(json.GetObject());
                    <span class="keyword">break</span>;
                <span class="keyword">case</span> JsonValueType.String:
                    obj = json.GetString();
                    <span class="keyword">break</span>;
            }
            <span class="keyword">return</span> obj;
        }

        <span class="keyword">static</span> <span class="keyword">dynamic</span> ConvertArray(JsonArray jsonArray)
        {
            <span class="keyword">dynamic</span>[] items = <span class="keyword">new</span> <span class="keyword">dynamic</span>[jsonArray.Count];
            <span class="keyword">for</span> (<span class="keyword">int</span> i = 0; i &lt; jsonArray.Count; i++)
            {
                items[i] = Convert(jsonArray[i]);
            }
            <span class="keyword">return</span> items;
        }

        <span class="keyword">static</span> <span class="keyword">dynamic</span> ConvertObject(JsonObject jsonObject)
        {
            <span class="keyword">dynamic</span> obj = <span class="keyword">new</span> System.Dynamic.ExpandoObject();
            <span class="keyword">var</span> d = (IDictionary&lt;<span class="keyword">string</span>, <span class="keyword">object</span>&gt;)obj;

            obj.Contains = <span class="keyword">new</span> Func&lt;<span class="keyword">string</span>, <span class="keyword">bool</span>&gt;((prop) =&gt;
                {
                    <span class="keyword">return</span> d.ContainsKey(prop);
                });
            obj.Get = <span class="keyword">new</span> Func&lt;<span class="keyword">string</span>, <span class="keyword">dynamic</span>&gt;((prop) =&gt;
                {
                    <span class="keyword">return</span> d[prop];
                });

            List&lt;<span class="keyword">string</span>&gt; keys = <span class="keyword">new</span> List&lt;<span class="keyword">string</span>&gt;();
            <span class="keyword">foreach</span>(<span class="keyword">var</span> key <span class="keyword">in</span> jsonObject.Keys)
            {
                keys.Add(key);
            }

            <span class="keyword">int</span> i = 0;
            <span class="keyword">foreach</span>(<span class="keyword">var</span> item <span class="keyword">in</span> jsonObject.Values)
            {
                d.Add(keys[i], Convert(item));
                i++;
            }
            <span class="keyword">return</span> obj;
        }
    }
}
</code></pre>


[WindowsDataJson]: http://msdn.microsoft.com/en-us/library/windows/apps/xaml/br240639.aspx
[Json.NET]: http://james.newtonking.com/projects/json-net.aspx