Title: HTTP on Roku
Date: 2019.01.15
Summary: Everything you ever wanted to know about making network calls on Roku
Category: Patterns

After getting your development environment all set up for Roku dev and creating your first `Hello World!` app, one of the first 'real' things you will need to do when developing a proper app is to fetch data from the network. However, Roku's HTTP stack is a little different than you might be used to on other platforms, so this post aims to walk through how it works and how to perform common operations.

## Overview

The object that you use to fetch data from the network in BrightScript is called [roUrlTransfer][]. This class implements several interfaces, the most important of which are [ifUrlTransfer][] and [ifHttpAgent][]. The Roku docs explain how to create an instance and what each of the methods do (though they can be a little bit sparse and don't include many real-world examples). One important thing to note is that the `roUrlTransfer` object can only be used from a `Task` node - since network calls are inherently asynchronous, Roku forces you to do these operations in a `Task` to avoid blocking the UI thread.

## Preparing a request

To make a request, the general steps are:

- Create an instance of the `roUrlTransfer` object
- Configure it appropriately, including setting the url
- Call one of the built-in methods to actually fetch the data

Let's start with some simple usage:

### `GET` request

To make a simple `GET` request, you can do this:

    request = CreateObject("roUrlTransfer")
    request.SetUrl("http://your.url/goes.here")
    response = request.GetToString()

In this case, `response` will be the actual HTTP body. The docs say that `GetToString` will *"connect to the remote service as specified in the URL and return the response body as a string. This function waits for the transfer to complete and it may block for a long time. This calls discards the headers and response codes. If that information is required, use AsyncGetToString instead."*

We will get to `AsyncGetToString` in a moment, but let's look at another example first.

### `POST` request

To `POST` data, the code looks similar:

    request = CreateObject("roUrlTransfer")
    request.SetUrl("http://your.url/goes.here")
    response = request.PostFromString(body)

Although the code is not much different, there are a couple of things to note here. The first is that the return value is not the same as `GetToString`. Instead, the docs say: *"The HTTP response code is returned. Any response body is discarded."* Probably not super useful in most cases, so we will see how to get the entire HTTP response (including body and headers) a little later on.

The second is that `body` is a string and is the exact HTTP body that you want to send. If your server is expecting `application/x-www-form-urlencoded` values (like would be received from an HTML `<form>`), then you need to format the data appropriately. [MDN says][MDN Post] *"the keys and values are encoded in key-value tuples separated by '&', with a '=' between the key and the value. Non-alphanumeric characters in both keys and values are percent encoded"*. You can use the [Escape][] function to url-encode the keys and values like this:

    request = CreateObject("roUrlTransfer")
    body = "firstname=" + request.Escape("Mary Ann") + "&lastname=" + request.Escape("Smith-Jones")
    request.SetUrl("http://your.url/goes.here")
    response = request.PostFromString(body)

If your server is expecting JSON, you can build an object and then use `[FormatJSON][]` to produce the string to send:

    request = CreateObject("roUrlTransfer")
    data = {
        firstname: "Mary Ann"
        lastname: "Smith-Jones"
    }
    body = FormatJSON(data)
    request.SetUrl("http://your.url/goes.here")
    response = request.PostFromString(body)

In most cases, you will also need to specify the `Content-Type` header as well, which brings us to...

### Request headers

To add headers to a request, you can either add them individually using the `AddHeader` method like this:

    request = CreateObject("roUrlTransfer")
    data = {
        firstname: "Mary Ann"
        lastname: "Smith-Jones"
    }
    body = FormatJSON(data)
    request.AddHeader("Content-Type", "application/json")
    request.AddHeader("X-Api-Key", "ABC-123-DEF")
    request.SetUrl("http://your.url/goes.here")
    response = request.PostFromString(body)

Or you can set them all at once by passing an AA to `SetHeaders` like this:

    request = CreateObject("roUrlTransfer")
    data = {
        firstname: "Mary Ann"
        lastname: "Smith-Jones"
    }
    body = FormatJSON(data)
    headers = {
        "Content-Type": "application/json"
        "X-Api-Key": "ABC-123-DEF"
    }
    request.SetHeaders(headers)
    request.SetUrl("http://your.url/goes.here")
    response = request.PostFromString(body)

Some headers (like `User-Agent` and `Content-Length`) are set automatically for you but can be overridden in special cases.

### Other HTTP verbs

Along with `GetToString` and `PostFromString`, there is also a built-in `Head` method that will do a `HEAD` request. But for all other HTTP verbs (such as `PUT`, `DELETE`, etc) you must use the `SetRequest` method:

    request = CreateObject("roUrlTransfer")
    request.SetRequest("DELETE")
    request.SetUrl("http://your.url/goes.here")
    response = request.GetToString()

Confusingly, you still have to use either `GetToString` or `PostFromString` to actually send the request, depending on if you need to also send a message body or not.

### HTTPS and SSL

Unlike most other HTTP stacks that you may be familiar with, `roUrlTransfer` does not support HTTPS in the default configuration. You must explicitly call `SetCertificatesFile` and pass it the local file path of a `.pem` file that contains the certificates from the Certificate Authority. You can provide your own `.pem` file, but for most scenarios it is easier to use the file provided by Roku with this special path: `common:/certs/ca-bundle.crt`. So the full usage would be:

    request = CreateObject("roUrlTransfer")
    request.SetCertificatesFile("common:/certs/ca-bundle.crt")
    request.SetUrl("https://your.secure.url/goes.here")
    response = request.GetToString()

If for some reason your server returns an invalid SSL certificate and you want to accept it anyway (perhaps in a testing or staging situation), you can set use `EnableHostVerification(false)` and `EnablePeerVerification(false)` to disable the SSL check.

*Note that there is also a method called `InitClientCertificates` but all that does is cause the Roku to send its own client cert to your server. There are cases where that might be useful (for authenticating a specific Roku device, for example), but it is not related to HTTPS.*

----------

## Handling a response

As mentioned earlier, `GetToString` will return the HTTP body, so if you:

- are making a `GET` call, and
- dont need the status code to know it was a success, and
- dont need any HTTP response headers, and
- dont mind the synchronous nature of `GetToString`

...then you can just use `GetToString` and grab the response and go about your business. However, in most cases, you *will* want to know if the request succeeded or not, or want to access the headers, or want to make the request asynchronously. In those cases, we need to move away from `GetToString` (and `PostFromString`) and use the asynchronous versions instead: `AsyncGetToString` and `AsyncPostFromString` (and `AsyncHead` if for some reason you need that).

For all of these methods, the request is sent asynchronously and later a `roUrlEvent` will be dispatched to the message port when the response is ready. This has several benefits:

- the call returns immediately so it does not block further execution
- the `roUrlEvent` contains not only the response body, but also access to the response status code and headers

In most non-trivial apps, these are the methods you will want to use.

### Handling the `roUrlEvent` message

In order to handle the asynchronous events, you need to configure the `roUrlTransfer` object to send its messages to a message port. Like other objects that implement the `ifSetMessagePort` interface, you do it like this:

    port = CreateObject("roMessagePort")
    request = CreateObject("roUrlTransfer")
    request.SetMessagePort(port)
    request.SetUrl("http://your.url/goes.here")
    request.AsyncGetToString()

Then, in your port-monitoring code, you can listen for the `roUrlEvent` message like this:

    msg = wait(0, port)
    if (type(msg) = "roUrlEvent")
        'TODO: handle response here
    else
        'Not a roUrlEvent
    end if

Once you have the `roUrlEvent` message, you can access the status code, body, and headers. The [roUrlEvent documentation][roUrlEvent] lists all available methods, but let's cover the common scenarios.

### Linking requests & responses

Note that the docs also have this to say about asynchronous requests:

> *"Each roUrlTransfer object can perform only one asynchronous operation at one time. After starting an asynchronous operation, you cannot perform any other data transfer operations using that object until the asynchronous operation has completed, as indicated by receiving an roUrlEvent message whose GetSourceIdentity value matches the GetIdentity value of the roUrlTransfer.  Furthermore, the roUrlTransfer object must remain referenced until the transfer has completed. That means that there must be at least one variable containing a reference to the object during the transfer.  Allowing the variable to go out of scope (for example, by returning from a function where the variable is declared, or reusing the variable to hold a different value) will stop the asynchronous transfer."*

If you use asynchronous requests, make sure that you keep a reference to the `roUrlTransfer` object until the response is complete or else the request will be cancelled. One approach is to create a dictionary of in-flight requests that will hold the references and can be used to look up the request when the response message comes in.

Each request generates a unique ID that can be read using the `GetIdentity` method like this:

    port = CreateObject("roMessagePort")
    request = CreateObject("roUrlTransfer")
    request.SetMessagePort(port)
    request.SetUrl("http://your.url/goes.here")
    request.AsyncGetToString()
    id = request.GetIdentity().ToStr()

When you receive a `roUrlEvent` message, it has a `GetSourceIdentity` method (I dont know why they have different names) that will return the ID of the response. You can use these IDs to correlate response messages with requests.

    msg = wait(0, port)
    if (type(msg) = "roUrlEvent")
        responseId = msg.GetSourceIdentity().ToStr()
        'If responseId = requestId, then this response goes with that request
    end if

You can use this ID as the key in the lookup dictionary to track multiple concurrent HTTP requests.

### HTTP status codes

When you get the `roUrlEvent` message, usually the first thing you want to do is check the status code to see if it succeeded. You can use the `GetResponseCode` method to read the status code:

    msg = wait(0, port)
    if (type(msg) = "roUrlEvent")
        responseId = msg.GetSourceIdentity().ToStr()
        statusCode = msg.GetResponseCode()
        'If statusCode >= 200 and < 300, then it was a success
    end if

If the status code is a positive value, it will represent the HTTP status code of the response. However, if the status code is a negative value, it indicates a low-level failure. These error codes are [documented here][ErrorCodes] (and notice that `roUrlTransfer` is really just a thin wrapper over `curl`).

### Handling the response body

You can get the actual response body with the `GetString` method. This will be the raw HTTP body so if you are expecting a structured format like JSON or XML, you will need to parse it accordingly:

    msg = wait(0, port)
    if (type(msg) = "roUrlEvent")
        responseId = msg.GetSourceIdentity().ToStr()
        statusCode = msg.GetResponseCode()
        body = msg.GetString()
        json = ParseJSON(body)
    end if

Note that if the status code is not in the 200-299 range, the body will be empty. You can change this behavior (so that you can get the full body for non-200 responses as well) by setting the `RetainBodyOnError(true)` method on the request

    port = CreateObject("roMessagePort")
    request = CreateObject("roUrlTransfer")
    request.SetMessagePort(port)
    request.SetUrl("http://your.url/goes.here")
    request.RetainBodyOnError(true)
    request.AsyncGetToString()

### Response headers

Strangely, there are two different ways to read the response headers. The first is `GetResponseHeaders` which returns an AA with the header names as the keys and the header values as the values. However, the documentation states: *"Headers are only returned when the status code is greater than or equal to 200 and less than 300."*. Also note that some headers are allowed to appear in the HTTP response multiple times, and since they are represented as an AA here (which cannot have duplicate keys), only the last HTTP value is represented.

    msg = wait(0, port)
    if (type(msg) = "roUrlEvent")
        responseId = msg.GetSourceIdentity().ToStr()
        statusCode = msg.GetResponseCode()
        if statusCode >= 200 AND < 300
            headers = msg.GetResponseHeaders()
            etag = headers["Etag"]
        end if
    end if

The alternative is `GetResponseHeadersArray`. This returns and array of AAs instead and the documentation states: *"Each associative array contains a single header name/value pair. Use this function if you need access to duplicate headers, since GetResponseHeaders() returns only the last name/value pair for a given name. All headers are returned regardless of the status code."* The down side of this is that you cannot directly access the header value by name (you have to loop over the array instead).

    msg = wait(0, port)
    if (type(msg) = "roUrlEvent")
        responseId = msg.GetSourceIdentity().ToStr()
        statusCode = msg.GetResponseCode()
        headers = msg.GetResponseHeadersArray()
        for each headerObj in headers
            for each headerName in headerObj
                ?headerName, headerObj[headerName]
            end for
        end for
    end if

### Putting it all together

    data = {
        firstname: "Mary Ann"
        lastname: "Smith-Jones"
    }
    body = FormatJSON(data)
    port = CreateObject("roMessagePort")
    request = CreateObject("roUrlTransfer")
    request.SetMessagePort(port)
    request.SetCertificatesFile("common:/certs/ca-bundle.crt")
    request.RetainBodyOnError(true)
    request.AddHeader("Content-Type", "application/json")
    request.SetRequest("PUT")
    request.SetUrl("https://your.secure.url/goes.here")
    requestSent = request.AsyncPostFromString(data)
    if (requestSent)
        msg = wait(0, port)
        if (type(msg) = "roUrlEvent")
            statusCode = msg.GetResponseCode()
            headers = msg.GetResponseHeaders()
            etag = headers["Etag"]
            body = msg.GetString()
            json = ParseJSON(body)
        end if
    end if

----------

## Helper Libraries

As you can see, Roku's HTTP handling is quite different than many of the other implementations you might be familiar with. If all of this `roUrlTransfer` stuff is confusing, you might consider using one of the open-source libraries out there that wrap it all up into a more user-friendly package. A couple to consider are:

[roku-requests][] - Python-inspired library that includes client-side caching. Lets you write code like:

    r = Requests().request("PUT", "https://httpbin.org/put", {"key":"value"})


[roku-fetch][] - Inspired by Javascript's `fetch` functionality, you can write code like:

    response = fetch({
        url: "http://example.url",
        timeout: 5000,
        method: "PUT",
        headers: {
            "Content-Type": "application/json",
            "If-None-Match": "abc123"
        }
        body: FormatJson({id: "xyz", amount: 8.29})
    })

*NOTE: Full disclosure - I am the author of the roku-fetch library*

----------

## Wrapping up

Hopefully that gave you a better understanding of how Roku's HTTP handling works and how to use it. If you have questions, head over to the [Roku Slack channel][RokuSlack] and ask away.



[roUrlTransfer]: https://sdkdocs.roku.com/display/sdkdoc/roUrlTransfer
[ifUrlTransfer]: https://sdkdocs.roku.com/display/sdkdoc/ifUrlTransfer
[ifHttpAgent]: https://sdkdocs.roku.com/display/sdkdoc/ifHttpAgent
[FormatJSON]: https://sdkdocs.roku.com/display/sdkdoc/Global+Utility+Functions#GlobalUtilityFunctions-FormatJson(jsonasObject,flags=0asInteger)asString
[MDN Post]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/POST
[Escape]: https://sdkdocs.roku.com/display/sdkdoc/ifUrlTransfer#ifUrlTransfer-Escape(textasString)asString
[roUrlEvent]: https://sdkdocs.roku.com/display/sdkdoc/roUrlEvent
[ErrorCodes]: https://sdkdocs.roku.com/display/sdkdoc/roUrlEvent
[roku-requests]: https://github.com/bvisin/roku-requests
[roku-fetch]: https://github.com/briandunnington/roku-fetch
[RokuSlack]: https://rokudevelopers.slack.com
