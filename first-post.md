Title: Hello World!
Date: 2012.03.19
Summary: See what tools are used to build this blog. Markdown, html5boilerplate, Twitter Bootstrap, Github Pages and more.

<!-- Main hero unit for a primary marketing message or call to action -->
<div class="hero-unit">
<h1>Hello, world!</h1>
<p>This is the first post on this blog. It is a bit of an experiment, as I am using several new (and new-to-me) techniques for creating and publishing this content</p>
<!--<p><a class="btn btn-primary btn-large">Learn more &raquo;</a></p>-->
</div>

This is the first post on this blog. It is a bit of an experiment, as I am using several new (and new-to-me) techniques for creating and publishing this content. All of the tools are based on standards (formal and de facto) and are freely available.

All of the content is written in [Markdown][], a human-readable syntax for formatting text. I am using the free [Markdown Pad][] editor to write the content.

The Markdown files are then ran through a custom tool that I created. The tool scans a directory for Markdown files (.md extension) and then converts the text to HTML (using the [Markdown Sharp][] library). The resulting HTML is then injected into an HTML template to create the final page. You can see the raw Markdown source used to generate any page by replacing the .html extension with .md (Example: [index.md][Example]).

As of this writing (2012.03.19), the template itself is using the [html5boilerplate][] library, with a side helping of [Twitter Bootstrap][] thrown in for the UI elements, all generated automatically by [Initializr][].

Finally, the pages are being hosted on [Github][] using their [Pages][Github Pages] feature. The static .html files are all version controlled and stored in Github as normal, but Github also makes them browsable.

Anyway, it sounds like a lot of moving parts, but it really makes things quite easy. Just open up a text editor and write the text without worrying about the HTML formatting too much, then click a button and everything is transformed into standards-compliant HTML magically. A simple push to Github and the pages are all updated with full backups, etc.

Let's see how it goes.

[Markdown]: http://daringfireball.net/projects/markdown/
[Markdown Pad]: http://markdownpad.com/
[Markdown Sharp]: http://code.google.com/p/markdownsharp/
[Example]: /first_post.md
[html5boilerplate]: http://html5boilerplate.com/
[Twitter Bootstrap]: http://twitter.github.com/bootstrap/
[Initializr]: http://www.initializr.com/
[Github]: https://github.com/
[Github Pages]: http://pages.github.com/
