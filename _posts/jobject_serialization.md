Title: Serializing JObject to camelCase
Date: 2020.11.25
Summary: How to serialize a JSON.NET JObject to camelCase
Image: http://briandunnington.github.io/images/camel.jpg

<div class="hero-unit">
<h1>Serializing JObject to camelCase</h1>
<p>How to serialize a JSON.NET JObject to camelCase</p>
</div>

Yesterday one of my colleagues ([@harsimran_hs][harsimran]) pinged me about some weird behavior he was seeing when serializing some JSON to return from one of our APIs. The strange thing was that sometimes the results were camelCased and sometimes they were coming back PascalCased for the same method.

Some investigation uncovered that in one code path, the object being serialized was a regular C# class from our domain model and in those cases it was being correctly serialized as camelCase. But in the other code path, the object was coming in as a `JObject` ([docs][jobject]) due to some (intentional) generic code handling. In the cases where the object was a `JObject`, the JSON output was being PascalCased.

We initially thought it would be an easy fix and tried setting the `JsonConvert.DefaultSettings` ([docs][defaultsettings]) to force the serializer to use camelCasing. When that didn't work, we tried moving closer to the source and providing a custom `JsonSerializer` at the point where the object was being serialized. When that didn't work either, we started wondering what was really going on.

As it turns out, it is by-design that `JObject` serialization **ignores** contract resolvers. So any `JObject` will serialize its properties as-is. That means that if you end up with a `JObject` that already has PascalCased properties, there is no built-in way to handle this case.

We went searching for an answer and came across a [blog post from Andrew Lock][andrewlock] that offered up a few options. The first two solutions involved avoiding even ending up with a wrongly-cased `JObject` in the first place, which wouldn't work in our situation. In his final solution, he suggested essentially looping over the `JObject` properties and manually converting them to camelCase before serializing. While this approach does work, it involved a bunch of extension methods and more code than we wanted for what was essentially a hack (Andrew called it 'hacky' himself).

So in the quest to make the hack as painless as possible, we went back to the drawing board and came up with the following one liner. Be warned, it is not the nicest line of code you are ever going to see:

    myJObj = JObject.FromObject(myJObj.ToObject<ExpandoObject>(), JsonSerializer.Create(new JsonSerializerSettings { ContractResolver = new CamelCasePropertyNamesContractResolver() }));

Let's break down what it is doing:

`myJObj.ToObject<ExpandoObject>()` first converts the `JObject` into a regular object - we use `ExpandoObject` in this case to take advantage of its dynamic nature to support the unknown list of properties.

Then `JObject.FromObject()` converts the `ExpandoObject` back into a `JObject`, but this time we can apply the `JsonSerializerSettings` and they will be honored. The end result is a new `JObject` with all of the properties pre-serialized as camelCase.

Let's take this example:

<br><img src="/images/jobject_example.png" width="100%"><br><br>

And then view the resulting output, including the desired output, what we get by default, and with our new hacky-one-line solution applied (note that it even handles nested objects):

<br><img src="/images/jobject_output.png" width="100%"><br><br>

So in the end we still ended up with a hacky solution but it is all on one line now with a nice big comment explaining why we are up to these shenanigans.

If you have any better ideas, head on over to [my Twitter][briandunnington] and let me know!


[harsimran]: https://twitter.com/harsimran_hs
[jobject]: https://www.newtonsoft.com/json/help/html/T_Newtonsoft_Json_Linq_JObject.htm
[defaultsettings]: https://www.newtonsoft.com/json/help/html/DefaultSettings.htm
[andrewlock]: https://andrewlock.net/serializing-a-pascalcase-newtonsoft-json-jobject-to-camelcase/
[briandunnington]: https://twitter.com/briandunnington
