Title: WinRT Xaml Image Source Bug
Date: 2013.06.13
Summary: Following Microsoft's 'Best Practices' advice when using images in Xaml can result in a strange bug.

<div class="hero-unit">
<h1>WinRT Xaml Image Source Bug</h1>
<p>Following Microsoft's 'Best Practices' advice when using images in Xaml can result in a strange bug.</p>
</div>

Using an image in Xaml is pretty easy, right? Just set the `Source` property and the image shows up.

	<Image Source="http://www.domain.net/image.png">

But what if your source image is large and you only want to display a small version of it? Of course the best thing to do is to use a smaller source image, but sometimes you dont/cant control the source. In those cases, your app will download the large image, process it in memory at full resolution, and then render it smaller on screen.

To alleviate some of the memory and processing hit, Microsoft introduced the [DecodePixelWidth] and [DecodePixelHeight] properties that allow the image to be processed and cached at the intended resolution. These properties are on the [BitmapImage] class and Microsoft offers up a [helpful example][BitmapImageExample] that even explains what is going on:

	<!-- Simple image rendering. However, rendering an image this way may not
     result in the best use of application memory. See markup below which
     creates the same end result but using less memory. -->
	<Image Width="200" Source="C:\Documents and Settings\All Users\Documents\My Pictures\Sample Pictures\Water Lilies.jpg"/>
	
	<Image Width="200">
	  <Image.Source>
	    <!-- To save significant application memory, set the DecodePixelWidth or  
	     DecodePixelHeight of the BitmapImage value of the image source to the desired 
	     height and width of the rendered image. If you don't do this, the application will 
	     cache the image as though it were rendered as its normal size rather then just 
	     the size that is displayed. -->
	    <!-- Note: In order to preserve aspect ratio, only set either DecodePixelWidth
	         or DecodePixelHeight but not both. -->
	    <BitmapImage DecodePixelWidth="200"  
	     UriSource="C:\Documents and Settings\All Users\Documents\My Pictures\Sample Pictures\Water Lilies.jpg" />
	  </Image.Source>
	</Image>

Great! Now you are following the best practices and saving memory - what isnt to love?

Let's switch gears for a minute and talk about something else: page caching. In Windows Store apps, you can set the [NavigationCacheMode] property of a Page so that the page remains cached in memory even when you navigate away from it. When you navigate back, the page renders immediately without having to rebuild its entire structure. Fine - but what does that have to do with image sources, and when am I going to get to this bug that I mentioned? Right now.

Say you have a page and you mark it with `NavigationCacheMode=Enabled` (or `Required`) and on that page you have an image you want to display. But that image can take a while to load, or maybe you are dynamically swapping out the image from time to time. No worries - you simply update the `Source` property (either directly or via databinding) and the new image pops into place.

But what if the image switch happens when the page is not visible on-screen? If you load up the page and then navigate away while the image is still loading, the image loading may complete while the original page is cached in memory but no longer visible on the screen. Oh well - when you navigate back, you will see the updated image, right? Nope.

When you navigate back, most likely you will simply see a black empty space. What? The image must not have loaded. Or maybe it was some kind of cross-thread error since the UI was not visible? Nope and Nope. The image was loaded just fine and no errors were encountered. And the image **is** there - you just can see it.

Using a tool like [XamlSpy] you can see that the image was indeed rendered, but for some reason is is invisible. I do not know why that is the case, but here is the catch: it only happens if you set your `Image.Source` declaratively by using a `BitmapImage`. If you use the normal string source, the image is rendered correctly and everything is peachy; of course you have to give up the benefits of DecodePixelWidth/Height to work around the issue.

This actually cropped up in a real-world app and after much investigation, it was a colleague of mine who figured out the root cause as I was about to tear my hair out. Thanks Mike!


[DecodePixelWidth]: http://msdn.microsoft.com/en-us/library/windows/apps/windows.ui.xaml.media.imaging.bitmapimage.decodepixelwidth
[DecodePixelHeight]: http://msdn.microsoft.com/en-us/library/windows/apps/windows.ui.xaml.media.imaging.bitmapimage.decodepixelheight.aspx
[BitmapImage]: http://msdn.microsoft.com/en-us/library/windows/apps/br243235.aspx
[BitmapImageExample]: http://msdn.microsoft.com/en-us/library/aa970269.aspx
[NavigationCacheMode]: http://msdn.microsoft.com/en-us/library/windows/apps/windows.ui.xaml.controls.page.navigationcachemode
[XamlSpy]: http://xamlspy.com/