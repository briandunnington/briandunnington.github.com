Title: Deploying Azure Web App Certificates from Key Vault
Date: 2018.11.27
Summary: How to work around an ARM template limitation when using a Key Vault certificate in a Web App
Image: http://briandunnington.github.io/images/deploy_certificates_from_key_vault.png

<div class="hero-unit">
<h1>Deploying Azure Web App Certificates from Key Vault</h1>
<p>How to work around an ARM template limitation when using a Key Vault certificate in a Web App</p>
</div>

Azure Web Apps have supported using certificates from Key Vault for a couple of years now, but there is currently no easy way
from the Azure portal to set this up. From the [instructions on the App Service team blog][Instructions]:

> Currently, Azure portal doesn’t support deploying external certificate from Key Vault, you need to call Web App ARM APIs 
directly using [ArmClient][]/[Resource Explorer][ResourceExplorer]/[Template Deployment Engine][TemplateDeploymentEngine].

That last one means that you can use an [ARM template like this one][ARMTemplate] to link your Key Vault-stored
certificate to your Web App. But...there is an annoying bug with the way certificates are handled by ARM templates.

Normally when an ARM template specifies a resource, Azure will first check if the resource exists, and if so, only
update the existing resource instead of trying to recreate the entire resource. But for some reason, this is not
how Web App certificates are handled. If the certificate is not yet linked to the Web App, everything will work
as expected. But if you run the ARM template again (ex: as part of a CI/CD pipeline), the ARM template deployment will
fail with the message:

    "Another certificate exists with same thumbprint XXXXXXXXXXXXXXXXXXXX at location xxxx 
    in the Resource Group xxxxxx."

I am not sure why this particular operation is not idempotent like every other ARM action, but there is a 
[feedback request][Feedback] to fix it.

I dont really like one-time use ARM templates, so I decided to just do this with direct Powershell commands instead:

    # Change these to your appropriave values
    $SubscriptionId = "xxxxxxxxxxxxxxxxx"
    $ResourceLocation = "West US"
    $ResourceGroupName = "myresourcegroup"
    $ResourceName = "mycertificate"
    $KeyVaultName = "mykeyvault"
    $KeyVaultId = "/subscriptions/xxxxxxxxxxxxx/resourceGroups/xxxxxxxx/providers/Microsoft.KeyVault/vaults/xxxxxxxxx"
    $KeyVaultSecretName = "certificatesecret"
    $ServerFarmId = "/subscriptions/xxxxxxxxxxxx/resourceGroups/xxxxxxxx/providers/Microsoft.Web/serverfarms/xxxxxxxxx"

    # Log in and select the correct subscription
    Login-AzureRmAccount 
    Set-AzureRmContext -SubscriptionId $SubscriptionId 

    # If you get a 'The service does not have access to...' error, then run these commands first (one time) to grant the necessary permissions:
    #Set-AzureRmKeyVaultAccessPolicy -VaultName $KeyVaultName -ServicePrincipalName abfa0a7c-a6b6-4736-8310-5855508787cd -PermissionsToSecrets get

    # Create certificate
    $PropertiesObject = @{
        keyVaultId = $KeyVaultId;
        keyVaultSecretName = $KeyVaultSecretName;
        serverFarmId = $ServerFarmId;
    }
    New-AzureRmResource -ResourceName $ResourceName -Location $ResourceLocation -PropertyObject $PropertiesObject -ResourceGroupName $ResourceGroupName -ResourceType Microsoft.Web/certificates -ApiVersion 2018-02-01 -Force

    # List certificates
    # Get-AzureRmResource -ResourceGroupName $ResourceGroupName -ResourceType Microsoft.Web/certificates -IsCollection -ApiVersion 2018-02-01

Few things to note:

* In order to grant the Azure tools the necessary permissions to access the Key Vault, you have to run the `Set-AzureRmKeyVaultAccessPolicy` command. This is only required once, and `abfa0a7c-a6b6-4736-8310-5855508787cd` is the Resource
Provider service principal name and is the same for all Azure subscriptions in the public cloud. (The link above also notes: 
*"Note for Azure Gov cloud environment you will need to use `6a02c803-dafd-4136-b4c3-5a6f318b4714` as the RP service principal name in the above command instead of `abfa0a7c-a6b6-4736-8310-5855508787cd`"*)

* By specifying the `-Force` option on `New-AzureRmResource`, this will cause the certificate to be redeployed even if it already exists. This lets you run the command multiple times without getting an error.

Still not as nice as just being able to deploy your certificate along with all of your other resources in a single ARM template, but
this works and can be scripted as part of a build pipeline.


[ARMTemplate]: https://github.com/Azure/azure-quickstart-templates/tree/master/201-web-app-certificate-from-key-vault
[Feedback]: https://feedback.azure.com/forums/281804-azure-resource-manager/suggestions/35348716-make-certificate-deployment-idempotent
[Instructions]: https://blogs.msdn.microsoft.com/appserviceteam/2016/05/24/deploying-azure-web-app-certificate-through-key-vault/
[ArmClient]: https://github.com/projectkudu/ARMClient
[ResourceExplorer]: https://resources.azure.com/
[TemplateDeploymentEngine]: https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-template-deploy
