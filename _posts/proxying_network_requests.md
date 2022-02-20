Title: Roku: Proxying network requests
Date: 2019.04.05
Summary: Some of the options available for inspecting network traffic from your Roku
Category: Tools

Roku does not have built-in support for proxies, and the HTTP stack (`roUrlTransfer`) does not expose the necessary methods to manually construct a valid proxy request. There are two approaches to get proxying working in spite of these limitations.

## System-level proxying

Although Roku does not explicitly support proxying, there is a way to capture the network traffic. The Hulu dev team [recently shared a post][HuluMITM] on how they set this up, but the basic steps are:

* Set up [mitmproxy][] on a host machine.
* Share the host machine's internet connection. This will appear as a normal WiFi network that the Roku can connect to.
* Configure some routing rules on the host machine. The steps to do this are specific to each OS but are [outlined in this forum thread][Routing].
* Modify your Roku app code to load the mitmproxy SSL certificate

Essentially what happens is that the Roku connects to the shared WiFi connection, which causes all traffic to flow through the host machine. The routing rules redirect the HTTP and HTTPS traffic through the mitmproxy, which allows you to inspect the results.

The Hulu blog post notes some caveats to be aware of:

* You can only set the mitmproxy SSL cert for network calls that your app makes. This means that the network calls from the Roku OS itself and some native video player/RAF calls cannot be configured to use the certificate. Those urls will need to be configured to pass through directly without being intercepted by mitmproxy.
* Setting up the routing rules is a bit tricky for the different OSes, and the forum post does not provide a working example for Windows at all.

## App-level proxying

There is another way to proxy both HTTP and HTTPS requests, but it requires modifying the actual outgoing request. As with the previous solution, it only works for apps that you develop with specific proxying support (it doesn't work for OS-level calls or other apps on the device). The advantage is that is does not require sharing your internet connection, setting up any special routing rules, or loading any special certificates. In order to support this type of proxying, two pieces are required.

### Modifying requests from the Roku

Include the following `configureRequestProxy()` function in your app. The function takes two arguments:

* `request` - This should be your original `roUrlTransfer` object that has already had `SetUrl()` called on it. _(The url will be munged to point to the proxy server but the original value will be preserved.)_
* `proxyAddress` - The IP address (and optional port) of your proxy server. Ex: `192.168.1.100:8888`

<!-- force end of list -->

    sub configureRequestProxy(request, proxy)
        if request <> invalid and proxy <> invalid
            proxyPrefix = "http://" + proxy + "/;"
            currentUrl = request.getUrl()
            if currentUrl.instr(proxyPrefix) = 0 then return
            proxiedUrl = proxyPrefix + currentUrl
            request.setUrl(proxiedUrl)
        end if
    end sub

Call this function any time you make a network request like so:

    function callApi(apiUrl, proxyAddress)
        port = CreateObject("roMessagePort")
        request = CreateObject("roUrlTransfer")
        request.SetCertificatesFile("common:/certs/ca-bundle.crt")
        request.InitClientCertificates()
        request.SetMessagePort(port)
        request.SetUrl(apiUrl)

        configureRequestProxy(request, proxyAddress) ' <-- THIS IS THE IMPORTANT PART

        requestSent = request.AsyncGetToString() ' or AsyncPostFromString()

        '...rest of response handling code as normal...
    end function

### Proxy server configuration

Since the `roUrlTransfer` object does not allow constructing an actual valid proxy request, the `createRequestProxy()` function instead modifies the outgoing url to pass the necessary info to the proxy server. The proxy server needs to be set up to understand these special requests and handle them appropriately.

#### Fiddler (Windows)

Rules > Customize Rules

In the `OnBeforeRequest()` method, add the following:

    // Roku Proxy
    var parts = oSession.fullUrl.Split([';'], 2)
    if (parts.Length > 1)
    {
        oSession.fullUrl = parts[1]
    }

<img src="/images/proxy_fiddler_settings.png" />

Other Notes: Make sure *'Allow remote computers to connect'* is enabled

#### Charles (Windows or Mac)

Tools > Rewrite

Import the [charles\_roku\_proxy.xml ruleset definition file][CharlesRules]. It should add a *'Roku Proxy'* ruleset with the necessary rules to enable the proxying.

<img src="/images/proxy_charles_settings.png" />

Other Notes: Make sure *'Rewrite'* is enabled whenever you run Charles. If you run Charles on a port other than `8888`, be sure to update the rule filter.


[mitmproxy]: https://mitmproxy.org/
[HuluMITM]: https://medium.com/hulu-tech-blog/automating-mitmproxy-and-improving-hulus-build-loading-tool-for-roku-629e7f763682
[Routing]: https://forums.roku.com/viewtopic.php?t=86886
[CharlesRules]: /charles_roku_proxy.xml
