Title: About
Summary: See what tools are used to build this blog. Markdown, html5boilerplate, Twitter Bootstrap, Github Pages and more.

<div class="hero-unit">
<h1>About this Blog</h1>
<p>This blog is a bit of an experiment, as I am using several new (and new-to-me) techniques for creating and publishing this content</p>
</div>

This blog is a bit of an experiment, as I am using several new (and new-to-me) techniques for creating and publishing this content. All of the tools are based on standards (formal and de facto) and are freely available.

All of the content is written in [Markdown][], a human-readable syntax for formatting text. <s>I am using the free [Markdown Pad][] editor to write the content.</s> ***[Update]:*** Now I just write the markdown right inside of [Visual Studio Code][] since it has a nice preview function and I am usually already in there anyway.

The Markdown files are then ran through [Nest][], a custom tool that I created. Nest scans a directory for Markdown files (.md extension) and then converts the text to HTML (using the [Markdown Sharp][] library). The resulting HTML is then injected into a [Razor][]-based template using [RazorMachine][] to render the final page.

The template itself is using the [html5boilerplate][] library, with a side helping of [Twitter Bootstrap][] thrown in for the UI elements, all generated automatically by [Initializr][].

Finally, the pages are being hosted on [Github][] using their [Pages][Github Pages] feature. The static .html files are all version controlled and stored in Github as normal, but Github also makes them browsable.

Anyway, it sounds like a lot of moving parts, but it really makes things quite easy. Just open up a text editor and write the text without worrying about the HTML formatting too much, then click a button and everything is transformed into standards-compliant HTML magically. A simple `git push` and the pages are all updated with full backups, etc.


[Markdown]: http://daringfireball.net/projects/markdown/
[Markdown Pad]: http://markdownpad.com/
[Visual Studio Code]: https://code.visualstudio.com/
[Markdown Sharp]: http://code.google.com/p/markdownsharp/
[Example]: /first-post.md
[html5boilerplate]: http://html5boilerplate.com/
[Twitter Bootstrap]: http://twitter.github.com/bootstrap/
[Initializr]: http://www.initializr.com/
[Github]: https://github.com/
[Github Pages]: http://pages.github.com/
[Razor]: http://msdn.microsoft.com/en-us/vs2010trainingcourse_aspnetmvc3razor.aspx
[RazorMachine]: https://github.com/jlamfers/RazorMachine
[Nest]: https://github.com/briandunnington/Nest
