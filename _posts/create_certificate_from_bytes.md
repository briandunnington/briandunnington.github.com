Title: Quick Fix: Creating an X509Certificate from bytes
Date: 2019.07.10
Summary: How to fix a strange error when creating an X509Certificate from an array of bytes
Image: http://briandunnington.github.io/images/create_certificate_from_bytes.png

<div class="hero-unit">
<h1>Quick Fix: Creating an X509Certificate from bytes</h1>
<p>How to fix a strange error when creating an X509Certificate from an array of bytes</p>
</div>

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


[StackOverflow]: https://stackoverflow.com/questions/9951729/x509certificate-constructor-exception
[Rationale]: https://stackoverflow.com/questions/52750160/what-is-the-rationale-for-all-the-different-x509keystorageflags/52840537#52840537
