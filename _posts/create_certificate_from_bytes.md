Title: Quick Fix: Creating an X509Certificate from bytes
Date: 2019.07.10
Summary: How to fix a strange error when creating an X509Certificate from an array of bytes
Image: http://briandunnington.github.io/images/create_certificate_from_bytes.png

<div class="hero-unit">
<h1>Quick Fix: Creating an X509Certificate from bytes</h1>
<p>How to fix a strange error when creating an X509Certificate from an array of bytes</p>
</div>

**UPDATED** - see [update 1](#update1) and [update 2](#update2) for even more info.

Dealing with certificates in .NET code is pretty straight-forward most of the time. If you need to access a certificate that is installed on the local machine, you can use the `X509Store` class to find certificates by name or other properties. But sometimes you need to load a certificate from raw bytes, and the `X509Certificate2` class has a constructor that allows you to do just that.

For example, here is how to load a certificate from a KeyVault secret:

    var azureServiceTokenProvider = new AzureServiceTokenProvider();
    var keyVaultClient = new KeyVaultClient(new KeyVaultClient.AuthenticationCallback(azureServiceTokenProvider.KeyVaultTokenCallback));
    var secret = await keyVaultClient.GetSecretAsync(keyVaultUrl, secretName);
    var certificate = new X509Certificate2(Convert.FromBase64String(secret.Value));

Now you can use your `certificate` as normal.

But here is where the strange part kicks in. If you include this code in an ASP.NET Core project and run it locally, it will work as expected. But if you deploy it to Azure (as a Web App or Azure Function), you will get this exception:

    The system cannot find the file specified

What gives? We aren't even trying to access a file. Well, when running in Azure, the code is actually running behind IIS. And when that is the case, it causes this code to stop working. So what is the fix? Use the constructor overload that accepts the `X509KeyStorageFlags` and make sure to pass these flags in:

    var certificate = new X509Certificate2(Convert.FromBase64String(secret.Value), 
                        (string)null, 
                        X509KeyStorageFlags.MachineKeySet | X509KeyStorageFlags.PersistKeySet | X509KeyStorageFlags.Exportable);

Turns out this has been biting folks for many years as [this post on StackOverflow from 7 years ago][StackOverflow] shows, and if you want to get deep into the weeds about what each of these flags do and why it works this way, I will point you over to [this explanation (also on StackOverflow)][Rationale] for some light reading.

<a name="update1"></a>
**UPDATE 2023.08.01**: This post is four years old, but this cropped up in a slightly new way again today. If you are using the [Azure Key Vault SDK for certificates][sdk] and use either the `DownloadCertificate()` or `DownloadCertificateAsync()` methods that take the certificate name, you can encounter this same issue. Behind the scenes, it is [doing the same thing as the original post][behindthescenes]. Luckily, the fix is easy: just use the overloads that take a `DownloadCertificateOptions` parameter and you can set the `KeyStoreFlags` property:

    var certificate = await certificateClient.DownloadCertificateAsync(new DownloadCertificateOptions(name) {
        KeyStoreFlags = X509KeyStorageFlags.MachineKeySet | X509KeyStorageFlags.PersistKeySet | X509KeyStorageFlags.Exportable
    });

<a name="update2"></a>
**UPDATE 2023.11.04**: We recently had a use case where we loaded a certificate (plus private key) from key vault to decrypt some data. When we ran it in Azure, though, we got `Keyset not found` - turns out this is a known issue and [the guidance][guidance] is to use the `EphemeralKeySet` flag to avoid saving the private key to disk at all. We made the switch and it did indeed solve our problem BUT we had another use case where we use a certificate with an `SslStream` for TLS client authentication and when we switched to `EphemeralKeySet` we started getting `No credentials are available in the security package` errors. Turns out that is also a [known issue][sslstream] (due to some deep internals of Windows) and the cert must be peristed. So there is no one set of flags that works for every scenario. These certificate keyset options are the gift that keep on giving.

[StackOverflow]: https://stackoverflow.com/questions/9951729/x509certificate-constructor-exception
[Rationale]: https://stackoverflow.com/questions/52750160/what-is-the-rationale-for-all-the-different-x509keystorageflags/52840537#52840537
[sdk]: https://www.nuget.org/packages/Azure.Security.KeyVault.Certificates/
[DownloadCertificate]: https://learn.microsoft.com/en-us/dotnet/api/azure.security.keyvault.certificates.certificateclient.downloadcertificate?view=azure-dotnet
[DownloadCertificateAsync]: https://learn.microsoft.com/en-us/dotnet/api/azure.security.keyvault.certificates.certificateclient.downloadcertificateasync?view=azure-dotnet
[behindthescenes]: https://github.com/Azure/azure-sdk-for-net/blob/79671d0524558c907c1d0bfbfd9d2e17615cbe8b/sdk/keyvault/Azure.Security.KeyVault.Certificates/src/CertificateClient.cs#L231C21-L231C97
[guidance]: https://github.com/projectkudu/kudu/wiki/Best-X509Certificate2-Practices
[sslstream]: https://github.com/dotnet/runtime/issues/23749