Title: Compare Across Forks
Date: 2015.03.10
Summary: My new favorite feature on GitHub

<!-- Main hero unit for a primary marketing message or call to action -->
<div class="hero-unit">
<h1>Compare Across Forks</h1>
<p>My new favorite feature on GitHub</p>
</div>

There are lots of ways to work with Git and GitHub on Windows and I use the [GitHub for Windows] app, [TortoiseGit], and the [command line] daily for various tasks. But my current new favorite feature is actually built in to the GitHub website.

Say you have forked a repository, make some changes, and then push them up to your fork. If you made your changes on a new branch, GitHub will take notice and offer up a handy option to create a pull request:

<br><img src="/images/compare_across_forks2.png" width="570"><br><br>

But if you were a bad kid and pushed your changes on the `master` branch, GitHub silently shames you and does not offer up the handy link (though it is also useful if you want to create a PR for changes that were not recently made). By using the `Compare across forks`, you can simply select your fork and your branch and generate the PR quickly and painlessly.

<br><img src="/images/compare_across_forks.png" width="570"><br><br>

I am sure this feature is old news to most folks, but I have been finding it extra useful lately so I thought I would share.

[GitHub for Windows]: https://windows.github.com/
[TortoiseGit]: https://code.google.com/p/tortoisegit/
[command line]: http://git-scm.com/docs/gittutorial
