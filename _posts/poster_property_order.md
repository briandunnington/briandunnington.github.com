Title: Roku: Poster component property order
Date: 2018.12.17
Summary: Often when loading images from the web, we do not have control over how large the source images are. Loading images that are too large will eat up precious Roku memory and cause performance problems in your app. There is a way to solve this, but it has an interesting twist that can trip people up.
Category: Tips

Often when loading images from the web, we do not have control over how large the source images are. Loading images
that are too large will eat up precious Roku memory and cause performance problems in your app.

However, the `Poster` component does have the ability to automatically resize the incoming image as it is loaded so
that no excess memory is used. The [Roku docs][PosterDocs] indicate that you can use the following
properties to control how the image is loaded into memory:

* `loadWith`
* `loadHeight`
* `loadDisplayMode`

By using these properties, you can load an image that is normally too large but only consume the memory required for the
desired final size. For example:

    ::: class="language-markup" :::
    <Poster uri="http://big.image" loadDisplayMode="scaleToZoom" loadWidth="720" loadHeight="405" />

If you view the [texture memory][TextureMemory], you can see:

<img src="/images/poster_properties_1.png" />

But wait - why does it report that the image is 1920x1080 if we specified it to be 720x405? The Roku docs actually mention
the reason, but it is not always immediately obvious:

> In order for the load scaling options to work, the option fields must be set in XML markup before the `uri` field.

What that means is that **property source order is important**. As soon as the `uri` field is set, the image begins loading
with whatever options have been set. If the `load*` properties have not been set yet, they will be ignored. So we need to
change our markup to ensure that the `uri` is set *after* the other properties:

    ::: class="language-markup" :::
    <Poster loadDisplayMode="scaleToZoom" loadWidth="720" loadHeight="405" uri="http://big.image" />

Now when you view the texture memory, you see:

<img src="/images/poster_properties_2.png" />

That looks more like it. Even though the Roku docs mention this, it catches lots of folks off guard since xml property order is normally not significant.

[PosterDocs]: https://sdkdocs.roku.com/display/sdkdoc/Poster
[TextureMemory]: https://sdkdocs.roku.com/display/sdkdoc/Texture+Memory
