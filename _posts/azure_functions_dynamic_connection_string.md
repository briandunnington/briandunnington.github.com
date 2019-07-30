Title: Azure Functions: Set Cosmos DB Trigger Connection String At Runtime
Date: 2019.07.30
Summary: How to specify the trigger connection string at runtime and avoid storing secrets during local development
Image: http://briandunnington.github.io/images/azure_functions.png

<div class="hero-unit">
<h1>Azure Functions: Set Cosmos DB Trigger Connection String At Runtime</h1>
<p>How to specify the trigger connection string at runtime and avoid storing secrets during local development</p>
</div>

By now you surely have noticed that I have a crush on [Azure Functions][AzureFunctions]. They are just about the perfect tool for so many scenarios. Lately, I have found myself using the [Cosmos DB trigger][CosmosDbTrigger] a lot to get notified when data is added or changed. Like many of the triggers that are triggered by another service, you have to specify the connection string to the DB in order for it to work. You don't actually specify the connection string, but the name of the setting that *contains* the connection string and you do it in the attribute that decorates the function, usually like this:

        [FunctionName("CosmosTrigger")]
        public static void Run([CosmosDBTrigger(
            databaseName: "ToDoItems",
            collectionName: "Items",
            ConnectionStringSetting = "CosmosDBConnection",
            LeaseCollectionName = "leases",
            CreateLeaseCollectionIfNotExists = true)]IReadOnlyList<Document> documents,
            ILogger log)
        {
            if (documents != null && documents.Count > 0)
            {
                log.LogInformation($"Documents modified: {documents.Count}");
                log.LogInformation($"First document Id: {documents[0].Id}");
            }
        }
    }

In the code above, `ConnectionStringSetting` is the name of the setting that contains the connection string. In this case, it is `CosmosDBConnection`, so you would want to make sure that setting is properly set in your Functions configuration.

Instead of storing the actual connection string, you can use the [Key Vault syntax][KeyVaultSyntax] to point the setting to a Key Vault secret that contains the connection string. The syntax looks like:

    @Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/mysecret/ec96f02080254f109c51a1f14cdb1931)

Now you have your connection string safely stored in Key Vault and your Function app is configured to read it directly. All is well - that is until you try to run the Function locally.

The Key Vault syntax is an App Service feature (not an Azure Functions feature), so it is not supported in the local Functions runtime that you use to run & debug locally. So you can't use the `@Microsoft.KeyVault()` value when you run your code locally. You could put the actual connection string into your `local.settings.json` and that will work, but that file is not usually checked into source control because you don't want to store your secrets in your source repository. That makes sharing the secret with other folks on your team a challenge - you could use old-fashioned communication techniques like *talking to each other*, but there has to be a better way. What we really want is to be able to read the connection string directly from Key Vault, even when running locally.

So what do we have to work with and what do we know:

- `ConnectionStringSetting` is a property on an attribute, so it must be a constant value (cannot be set to a variable value)
- The `@Microsoft.KeyVault()` syntax is a constant but only works when deployed to Azure, not when running locally
- We don't want to store secrets in the `local.settings.json` file

That seems to take away most of our options, but if you do a little diving into the [Cosmos DB Trigger source code][SourceCode], you will notice that it first tries to use the `ConnectionStringSetting` but if it is not set, will fallback to using `_options.ConnectionString`:


    internal string ResolveConnectionString(string unresolvedConnectionString, string propertyName)
    {
        // First, resolve the string.
        if (!string.IsNullOrEmpty(unresolvedConnectionString))
        {
            string resolvedString = _configuration.GetConnectionStringOrSetting(unresolvedConnectionString);

            if (string.IsNullOrEmpty(resolvedString))
            {
                throw new InvalidOperationException($"Unable to resolve app setting for property '{nameof(CosmosDBTriggerAttribute)}.{propertyName}'. Make sure the app setting exists and has a valid value.");
            }

            return resolvedString;
        }

        // If that didn't exist, fall back to options.
        return _options.ConnectionString;
    }

 So what is `_options.ConnectionString` and how can we leverage it? Well, it is an instance of `CosmosDBOptions` and it can be set using the standard Options registration patterns:

    builder.Services.PostConfigure<CosmosDBOptions>(options =>
    {
        options.ConnectionString = "dynamically set the connection string value here";
    });

(Note that the above code assumes that you are using an `IWebJobsStartup` class and dependency injection - and really, why wouldn't you be? 😁)

If you combine that with some code to read settings from Key Vault, you end up with something like this:

    var configBuilder = new ConfigurationBuilder();
    configBuilder.AddAzureKeyVault("https://myvault.vault.net");
    var config = configBuilder.Build();
    builder.Services.PostConfigure<CosmosDBOptions>(options =>
    {
        options.ConnectionString = config["CosmosDbUrl"]; // CosmosDbUrl is a secret in the Key Vault
    });

**One important thing to note:** The trigger will only fall back to using the `_options.ConnectionString` value if the connection string is *not* set by the attribute. So in your function definition, make sure to set `ConnectionStringSetting` to `""`:

        [FunctionName("DbFunction")]
        public static void Run([CosmosDBTrigger(
            databaseName: "databaseName",
            collectionName: "collectionName",
            ConnectionStringSetting = "",
            LeaseCollectionName = "leases")]IReadOnlyList<Document> input, ILogger log)
        {
            if (input != null && input.Count > 0)
            {
                log.LogInformation("Documents modified " + input.Count);
                log.LogInformation("First document Id " + input[0].Id);
            }
        }

Now even when you run the code locally, the code will fetch the connection string value from Key Vault, the `CosmosDBOptions` instance will be configured with that value, and it will all be made available to the trigger when resolving the connection string value.



[AzureFunctions]: https://azure.microsoft.com/en-us/services/functions/
[CosmosDbTrigger]: https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-cosmos-db-triggered-function
[KeyVaultSyntax]: https://docs.microsoft.com/en-us/azure/app-service/app-service-key-vault-references
[SourceCode]: https://github.com/Azure/azure-webjobs-sdk-extensions/blob/b7aa8f84c2b3c9065861fe955ce8180b9d19f939/src/WebJobs.Extensions.CosmosDB/Trigger/CosmosDBTriggerAttributeBindingProvider.cs#L261
