Title: Azure Functions: Trigger Attribute Settings From Key Vault
Date: 2020.03.20
Summary: How to read trigger attribute settings from Key Vault
Image: http://briandunnington.github.io/images/azure_functions.png

<div class="hero-unit">
<h1>Azure Functions: Trigger Attribute Settings From Key Vault</h1>
<p>How to read trigger attribute settings from Key Vault</p>
</div>

Yesterday I saw this tweet on Twitter:

<br><img src="/images/azure_functions_key_vault_tweet.png" width="100%"><br><br>

We came up for a solution to this at work and use it in all of our services so I thought I would share it. (By the way, if you want to come work with me on cool stuff like this, [we are hiring][hiring])

The root of the issue is that the triggers in Azure Functions often need some kind of configuration. Things like Cosmos DB change feed triggers, event hub triggers, and queue triggers all need things like connection strings and queue names to know what resource they are monitoring. These values come from the application settings and the runtime knows how to resolve them properly.

Often, these values contain secrets so it is great to store them in Key Vault. To enable this common scenario, Azure Functions supports a [special Key Vault syntax][syntax] for application setting values:

    @Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/mysecret/ec96f02080254f109c51a1f14cdb1931)

    -or-

    @Microsoft.KeyVault(VaultName=myvault;SecretName=mysecret;SecretVersion=ec96f02080254f109c51a1f14cdb1931)

This is resolved by the runtime automatically and the secret value from Key Vault is made available.

The problem arises during local development. Application settings are provided by the `local.settings.json` file when developing locally, but the `local.settings.json` file does not support the Key Vault syntax.

<br><img src="/images/azure_functions_key_vault_nolocal.png" width="100%"><br><br>

 (The reason is that that feature is not part of the Azure Functions runtime but instead [provided by the underlying App Service functionality][underlying].)

So developers often resort to copying in connection strings/etc into their `local.settings.json` file while developing. They have to be extra careful not to commit this file to source control (the default template automatically adds it to the `.gitignore` list), but then it is hard to share with other folks on your team. In the end, it is not an elegant process. What we really want is a method that would let us use the secrets from Key Vault both in Azure and locally, remove the need for manually sharing the secrets between team members, while ensuring that we don't accidentally commit them to source control.

So - what is the solution? Well, since Azure Functions is built on top of ASP.NET Core, it is already using `IConfiguration` behind the scenes. All of the environment variables and application settings are read into this configuration and it is used to resolve the trigger attribute values. So we need a way to inject our Key Vault values into the runtime's `IConfiguration` instance.

    /* This is the meat of the logic. It finds the IConfiguration service that is already registered by the runtime, and then
        * creates an instance of the concrete class if necessary. It loads all of the Key Vault secrets into the config, and then
        * replaces the registered IConfiguration instance with the patched version.
        * 
        * Use at your own risk 😊
        * */
    public static IConfiguration AddAzureKeyVaultConfiguration(this IServiceCollection services, string keyVaultUrlSettingName)
    {
        // get the IConfiguration that is already registered with the host
        var configBuilder = new ConfigurationBuilder();
        var descriptor = services.FirstOrDefault(d => d.ServiceType == typeof(IConfiguration));
        if (descriptor?.ImplementationInstance is IConfigurationRoot configuration)
        {
            configBuilder.AddConfiguration(configuration);
        }
        else if (descriptor?.ImplementationFactory != null)
        {
            var sp = services.BuildServiceProvider();
            var constructedConfiguration = descriptor.ImplementationFactory.Invoke(sp) as IConfiguration;
            configBuilder.AddConfiguration(constructedConfiguration);
        }

        // build a temporary configuration so we can extract the key vault urls
        var tempConfig = configBuilder.Build();
        var keyVaultUrls = tempConfig[keyVaultUrlSettingName]?.Split(',');

        // add the key vault providers and build a new config
        configBuilder.AddAzureKeyVaults(keyVaultUrls);
        var config = configBuilder.Build();

        // replace the existing IConfiguration instance with our new one
        services.Replace(ServiceDescriptor.Singleton(typeof(IConfiguration), config));
        return config;
    }

First the code finds the already-registered `IConfiguration` type in the services collection. If necessary, it creates an instance of the concrete class. Then it reads all of the secrets from Key Vault and injects them into the configuration. Finally, it replaces the service registration with the patched instance.

Let's look at an example. For this example, I have already set up my Key Vault and added a few secrets:

<br><img src="/images/azure_functions_key_vault_secrets.png" width="100%"><br><br>

In order to use it, you only have to pass in the *name of the setting* that contains your Key Vault url. Since the Key Vault url is not a secret itself, you can put it in your `local.settings.json` and commit it to source control.

<br><img src="/images/azure_functions_key_vault_localsettings.png" width="100%"><br><br>

Here is how you set it up in your `Startup.cs` file (you are using [dependency injection in your Functions][di], right?)

<br><img src="/images/azure_functions_key_vault_startup.png" width="100%"><br><br>

Note that this also makes the `config` variable available to your code so you can use any of the settings to configure your other dependencies as well.

Then in your function, you can just use the Key Vault secret names like normal application settings. Notice that it works with both regular setting names and with the `%{name}%` resolver syntax.

<br><img src="/images/azure_functions_key_vault_function.png" width="100%"><br><br>

When you run it, everything Just Works™. Whether you are running locally or in Azure, the code only needs one setting to be configured: the url to your Key Vault. Because it leverages the built-in Key Vault handling, the authentication portion automatically uses Managed Service Identity when running in Azure but falls back to your personal credentials when running locally.

All of the code and the example is [available on Github][source].

Please try it out and let me know if you find it useful. I will create a NuGet package if this is something that other folks would like to use in their projects.

[hiring]: https://careers.microsoft.com/us/en/job/803455/Senior-Software-Engineer
[syntax]: https://docs.microsoft.com/en-us/azure/app-service/app-service-key-vault-references
[underlying]: https://github.com/Azure/azure-webjobs-sdk/issues/746#issuecomment-458652023
[di]: https://docs.microsoft.com/en-us/azure/azure-functions/functions-dotnet-dependency-injection
[source]: https://github.com/briandunnington/AzureFunctionsKeyVaultSettingsExample
