Title: Assmebly and File Version Information in WinRT
Date: 2013.03.14
Summary: How to get the assembly and file version information in WinRT apps

<div class="hero-unit">
<h1>Assembly and File Version Information in WinRT</h1>
<p>How to get the assembly and file version information in WinRT apps</p>
</div>

Short and simple post this time. If you need to get the assembly and file version information in your Windows Store app, here is what you need. Note that the `using System.Reflection` is important since the `GetTypeInfo()` method is an extension method in that namespace.

<pre><code class="csharp"><span class="keyword">using</span> System;
<span class="keyword">using</span> System.Reflection;

<span class="keyword">namespace</span> Example
{
    <span class="keyword">public</span> <span class="keyword">class</span> <span class="type">VersionInformation</span>
    {
        <span class="keyword">public</span> VersionInformation()
        {
            <span class="keyword">var</span> assembly = <span class="keyword">this</span>.GetType().GetTypeInfo().Assembly;
            <span class="keyword">var</span> assemblyVersion = assembly.GetName().Version.ToString();
            <span class="keyword">var</span> fileVersion = assembly.GetCustomAttribute&lt;<span class="type">AssemblyFileVersionAttribute</span>&gt;().Version;
            <span class="keyword">var</span> productVersion = assembly.GetCustomAttribute&lt;<span class="type">AssemblyInformationalVersionAttribute</span>&gt;().InformationalVersion;
            System.Diagnostics.<span class="type">Debug</span>.WriteLine(<span class="string">&quot;Assembly Version: &quot;</span> + assemblyVersion);
            System.Diagnostics.<span class="type">Debug</span>.WriteLine(<span class="string">&quot;File Version: &quot;</span> + fileVersion);
            System.Diagnostics.<span class="type">Debug</span>.WriteLine(<span class="string">&quot;Product Version: &quot;</span> + productVersion);
        }
    }
}
</code></pre>

Similar, but slightly different than in other versions of .NET since `Assembly.GetExecutingAssembly()` and `System.Diagnostics.FileVersionInfo` are not available. Hope that helps somebody out.

