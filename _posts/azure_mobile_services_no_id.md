Title: Azure Mobile Services: No 'id' member found on type
Date: 2014.01.06
Summary: When using Azure Mobile Services, you can sometimes get an error with the message "No 'id' member found on type" even when your type has an 'id' member. Here is why and how to solve it.

<div class="hero-unit">
<h1>Azure Mobile Services: No 'id' member found on type</h1>
<p>When using Azure Mobile Services, you can sometimes get an error with the message "No 'id' member found on type" even when your type has an 'id' member. Here is why and how to solve it.</p>
</div>

While developing my latest Windows Phone app, [Quandry], I ran into an issue that took some detective work to figure out, so I thought I would share the solution. I am using the most-excellent [Azure Mobile Services][ams] to handle backend data storage, authentication, and notifications, and if you haven't checked it out yet, you definitely should. Most of the time, it 'just works' and really solves a lot of the recurring things that mobile apps need.

Using Mobile Service for data storage is super easy. You just pass in an instance of your data model class, and the Azure SDK handles serializing it to JSON and sending it over the wire, where it gets stored into a SQL database. To facilitate data storage, each item needs a unique 'id' property. If you try to serialize and send data based on a class with no 'id' property, Azure will complain and remind you to add the property - makes sense.

I had created my model class and was happily saving data to Azure when I started getting an error: "No 'id' member found on type". Hmmm, I had been saving this same object previously and it worked fine. I quadruple checked that the type did indeed have an 'id' member (it did) and I just could not figure it out. Even stranger, sometimes it *would* work without any code changes, then go back to failing on the next run.

Luckily, the Azure folks have [open-sourced their SDK][sdk] so I cracked it open and took a peek. Stepping through the code, I was able to ascertain what was happening. Normally, you would write something like this:

`await client.GetTable<Item>().InsertAsync(new Item());`

In order to know which table to store your data in, the SDK uses the name of the type being passed in - in this case, 'Item'. I determined that the bug only manifests itself when you are using a data model class that inherits from another model class (ex: you have a SpecialItem type that inherits from Item). In this case, you might write:

`await client.GetTable<Item>().InsertAsync(new SpecialItem());`

But this will fail. The cause is that the logic to standardize the Id property to lowercase is not run in this scenario. In the CreateProperties method in MobileServiceContractResolver, one of the first things it does is:

`if (tableNameCache.ContainsKey(type)) {...}`

However, in this scenario, the tableNameCache does not contain the type, since the table is named 'Item' and the type is 'SpecializedItem'. As such, all of the logic to find the Id property is skipped. Later, in ResolveIdProperty, the Id property cannot be found because it does not match MobileServiceUrlBuilder.IdPropertyName (which is always "id").

The solution is to pass the subclassed type as the generic parameter in the GetTable call like this:

`await client.GetTable<SpecializedItem>().InsertAsync(new SpecializedItem());`

Of course this means that the Mobile Service client will try to insert your data into a table called 'SpecializedItem' (which you probably dont want) so you need to set the table name on the SpecializedItem class (using DataContract(Name="Item") or the JsonContainer or DataTable attributes).

This is probably a pretty rare edge case, but when I searched on the Mobile Services forums, I did see a few other folks encountering the same error so I [reported my findings][bugreport] to the team. <strike>They could probably fix it by looking at the base type name if the original type name is not found, but they probably shouldn't do that. Most of the time, you wouldn't need to insert a subclasses type into a base type's table, and if you do, you simply need to decorate the subclassed type with the appropriate DataContract attribute to indicate that you really intended that behavior.</strike>

<b>UPDATE:</b> Looks like the team went ahead and fixed this bug: [https://github.com/WindowsAzure/azure-mobile-services/pull/189][fix]


[Quandry]: http://www.windowsphone.com/en-us/store/app/quandry/281499cb-f9fe-427f-8906-7fa3a02673ab
[ams]: http://www.windowsazure.com/en-us/services/mobile-services/
[sdk]: https://github.com/WindowsAzure/azure-mobile-services
[bugreport]: http://social.msdn.microsoft.com/Forums/windowsazure/en-US/cacb2a22-ec54-4cdf-9f3c-fe8cf83cde41/bug-report-no-id-member-found-on-type?forum=azuremobile#33f5587f-fc1e-4a2b-99b5-c63c48409191
[fix]: https://github.com/WindowsAzure/azure-mobile-services/pull/189