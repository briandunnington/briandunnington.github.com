Title: Roku Round-up
Date: 2018.11.06
Summary: Quick updates on some Roku libraries that I recently released
TODO_Image: http://briandunnington.github.io/images/roku.png

<div class="hero-unit">
<h1>Roku Round-up</h1>
<p>Quick updates on some Roku libraries that I recently released</p>
</div>

I have previously posted about a few of my Roku library projects ([here][RedokuPost] and [here][RoactPost]), but I have recently
made some improvments and also released a couple more useful tools so I thought I would do a quick round-up of everything.

<div class="img">
<img src="images/roku.png" style="border: 0;margin:0 auto;"/>
</div>

### Redoku

[Redoku][Redoku] is a Redux-inspired state-management library for Roku. If you are a JS dev and have used Redux, then Redoku
should feel very familiar to you. All of the concepts (one-way data flow, actions, reducers, global state store, etc) are 
exactly the same with just a few changes to fit into the BrightScript/SceneGraph constraints.

There have been no specific updates to Redoku lately, but that is because it just continues to be rock solid for me. I use
it in every project and it 'just works'. If you are developing a Roku app, take a look and let me know what you think.

### Roact

Just as Redoku is the Redux of Roku, [Roact][] is the React of Roku. You build components and compose them together with
familiar methods like `componentDidMount`, `setState`, and `render`. It is a bit different than thinking in pure SceneGraph
terms, but if you have used React or React Native, you should feel right at home.

    function render(p)
        return h("Group", {}, [
                    h("Board"),
                    h("Label", {text: "Welcome to tic-tac-toe", translation: [1000,72]}),
                ])
    end function

I recently stated using this in a fairly complex app that really gave it a thorough testing. I added support for `componentDidUpdate`
for responding to changes in `props`, and fixed a few bugs (mostly related to removing child nodes that were no longer in
the visual tree).

Of course, Roact and Redoku work great together. But just like with React and Redux, Roact and Redoku can each be used 
independently. The Roact repository has examples of apps using Redoku and without, and I have built lots of apps with Redoku 
long before Roact even existed.

### Promises

If you have not used BrightScript, you might not know that there is no language-level support for asynchronous operations.
Instead, you create `Task` nodes (essentially threads) and 'observe' their fields and get notified when they change. This
results in a lot of boilerplate code and callback functions littered throughout your code.

The [roku-promise][RokuPromise] library tries to solve that by wrapping that pattern in a familiar Promise-like syntax.
The end result is that you can write code like this:

    createTaskPromise("TaskName", {
        input1: "some value",
        input2: 123
    }).then(sub(task)
        results = task.output
        ' do other stuff here with the results.
        ' m is the original context from the caller
        m.label.text = results.text
    end sub)

The best part is that the roku-promise library manages the scope/context for you so that when the code in the `then`
delegate gets called, the context is the same as the original caller (in other words, `m` is the same).

I recently released this as open-source and updated the code and docs to include examples of some other more
advanced use cases (long-running tasks, promises without a task, etc) and am excited to see what else other
folks can build on top of this.

### Fetch

Continuing the trend of Javascript-inspired patterns, I also recently released the [roku-fetch][RokuFetch] library.
Roku's HTTP framework is quite a bit different than most other programming languages, so roku-fetch wraps it up
in an API that should be familiar to anyone doing front-end development for the web.

The most basic usage is simply:

    response = fetch({url: "http://example.url"})
    if response.ok
        ?"The response was: " + response.text()
    end if

But it also has full support for headers, HTTP verbs, timeout, and more:

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
    if response.ok
        etag = response.headers["ETag"].value
        cookies = response.headers["Set-Cookie"]
        while cookies <> invalid
            ?cookies.value
            cookies = cookies.next
        end while
        json = response.json()
        ?json.items.total
    else
        ?"The request failed", response.statusCode, response.text()
    end if

I have really liked not having to worry about `roUrlTransfer` objects and `roMessagePort` loops while using this.

### Wrap up

All of these libraries have made Roku development faster, easier, and more fun for me personally, so I wanted to 
share them with the larger community. They can all be mixed and matched so try some (or all) of them out and 
let me know how it goes.


[RedokuPost]: redoku
[RoactPost]: roact
[Redoku]: https://github.com/briandunnington/Redoku
[Roact]: https://github.com/briandunnington/Roact
[RokuPromise]: https://github.com/briandunnington/roku-promise
[RokuFetch]: https://github.com/briandunnington/roku-fetch
