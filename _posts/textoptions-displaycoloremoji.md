Title: Windows Phone Color Emoji Override
Date: 2013.10.15
Summary: What to do when your font glyphs start showing up in multi-color when you dont want them to

<div class="hero-unit">
<h1>Windows Phone Color Emoji Override</h1>
<p>What to do when your font glyphs start showing up in multi-color when you dont want them to</p>
</div>

I recently released a new game for Windows Phone called [Quandry]. As part of the design, I was leveraging the font glyphs in [Segoe UI Symbol][SegoeUISymbol] instead of using images in a few places. The result looked like this:

<img src="/images/wp_ss_20131015_0002.png" width="570">

I was pretty happy with it and didnt give it much more thought after I implemented it. Then Microsoft announced their [Windows Phone Preview for Developers][Preview] program which allows registered developers to install OS updates without waiting for carrier approval and general availability. Cool - I clicked a few buttons and within a few minutes, I was running "Update 3" (formerly knowns as GDR3). I fired up Quandry and noticed something weird:

<img src="/images/wp_ss_20131015_0001.png" width="570">

Some (but not all) of my glyphs had been replaced with colored variants. What the ?

I remembered reading about [Microsoft's new colored-font proposal][Proposal] awhile back and how it added some new color information to the font tables to enable 'layers' that could be colorized:

<img src="http://opentype.info/blog/wp-content/uploads/2013/07/winemoji.png" width="570"/>

Neat stuff, but it was breaking my app! I tried a few things to fix the issue, but didnt have much luck. Then I stumbled across a property I had not heard of before: [TextOptions.DisplayColorEmoji][DisplayColorEmoji]. Apparently it is set to `true` by default, so I set it to `false` on my affected `TextBlock` controls and everything was back to normal.

Maybe everybody already knows about this little detail, but I didnt and figured I would share it in case anybody else ran into the same issue.

Shameless plug: Download [Quandry] from the marketplace and give it a try. If you do, challenge me at [@briandunnington][briandunnington] and we will have a game.


[Quandry]: http://windowsphone.com/s?appId=281499cb-f9fe-427f-8906-7fa3a02673ab
[SegoeUISymbol]: http://www.adamdawes.com/windows8/win8_segoeuisymbol.html
[Preview]: https://dev.windowsphone.com/en-us/featured/update3
[Proposal]: http://opentype.info/blog/2013/07/03/color-emoji-in-windows-8-1-the-future-of-color-fonts/
[DisplayColorEmoji]: http://msdn.microsoft.com/en-US/library/windowsphone/develop/system.windows.media.textoptions.displaycoloremoji%28v=vs.105%29.aspx
[briandunnington]: http://twitter.com/briandunnington
