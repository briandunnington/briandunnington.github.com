Title: Sharing Html & Images via the Share Contract
Date: 2013.11.06
Summary: The right way to share html that contains images (and other external resources) via the Share charm.

<div class="hero-unit">
<h1>Sharing Html & Images via the Share Contract</h1>
<p>The right way to share html that contains images (and other external resources) via the Share charm.</p>
</div>

> tl;dr - Although all of Microsoft's samples indicate that you should use `ms-appx:///` for external content, you actually need to use `ms-appx-web:///` or the content won't show up in the receiving app.

Just a quick note to myself (and anybody else out there that stumbles upon this) so I remember how to do this correctly next time. When sharing content via the Share charm in Windows 8.x, there are several built-in supported data types, including text, image, links, and html. When sharing html that contains references to external resources (such as images or css) that are part of your app package, you have to handle those resources in a specific way in order for them to be shared properly.

Fortunately, Microsoft has a helpful link on MSDN called '[How to share HTML][msdn]' that walks you through everything you need to do:

<pre><code class="csharp"><span class="keyword">private</span> <span class="keyword">void</span> RegisterForShare()
{
    DataTransferManager dataTransferManager = DataTransferManager.GetForCurrentView();
    dataTransferManager.DataRequested += <span class="keyword">new</span> TypedEventHandler&lt;DataTransferManager, 
        DataRequestedEventArgs&gt;(<span class="keyword">this</span>.ShareHtmlHandler);
}

<span class="keyword">private</span> <span class="keyword">void</span> ShareHtmlHandler(DataTransferManager sender, DataRequestedEventArgs e)
{
    DataRequest request = e.Request;
    request.Data.Properties.Title = <span class="string">&quot;Share Html Example&quot;</span>;
    request.Data.Properties.Description = 
        <span class="string">&quot;Demonstrates how to share an HTML fragment with a local image.&quot;</span>;

    <span class="keyword">string</span> localImage = <span class="string">&quot;ms-appx:///Assets/Logo.png&quot;</span>;
    <span class="keyword">string</span> htmlExample = <span class="string">&quot;&lt;p&gt;Here is a local image: &lt;img src=\&quot;&quot;</span> + localImage + <span class="string">&quot;\&quot;&gt;.&lt;/p&gt;&quot;</span>;
    <span class="keyword">string</span> htmlFormat = HtmlFormatHelper.CreateHtmlFormat(htmlExample);
    request.Data.SetHtmlFormat(htmlFormat);

    <span class="comment">// Because the HTML contains a local image, we need to add it to the ResourceMap.</span>
    RandomAccessStreamReference streamRef = 
         RandomAccessStreamReference.CreateFromUri(<span class="keyword">new</span> Uri(localImage));
    request.Data.ResourceMap[localImage] = streamRef;
}
</code></pre>

Simple enough stuff: add the code, fire up your apps, invoke the Share charm, pick a target and your content is shared just like you expected. But actually, it isn't just like you expected. If you follow MS's code sample, your images (or css, etc) won't show up.

I ran into this and figured it has to be something to do with the [ResourceMap] that, umm...mapped the external resources. =) But I fiddled with it for way too long before deciding that I was going down the wrong path.

After more Googling and trial and error, I finally stumbled up on the solution: you must use `ms-appx-web:///` as the protocol for external resources that live in your app package and are referenced via shared html.

[Here is an article][contexts] that talks about the difference between the 'contexts' that `ms-appx:///` and `ms-appx-web:///` run under. The article is more from the perspective of a Windows Store app written in html/js, but the same distinctions apply to managed and unmanaged apps as well. The article has lots of good info, but it never came right out and mentioned my exact problem, so I thought I would write this up so I could find it easily next time.


[msdn]: http://msdn.microsoft.com/en-us/library/windows/apps/hh973055.aspx
[ResourceMap]: http://msdn.microsoft.com/en-us/library/windows/apps/windows.applicationmodel.datatransfer.datapackage.resourcemap.aspx
[contexts]: http://www.jasonfollas.com/blog/post/2012/07/09/Metro-Introducing-the-Local-and-Web-Contexts.aspx