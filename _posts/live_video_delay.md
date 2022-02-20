Title: Roku: Fix for live video delay
Date: 2018.12.20
Summary: How to make sure your live stream is really live
Category: Tips

The first time you implement live video playback in your Roku app and then go to try it out, you invariably find that the video is starting from some other point than the current live position. This one trips a lot of folks up (see [here][Example1], [here][Example2], [here][Example3], and [here][Example4]), but luckily it has an easy solution.

The trick is to set the `PlayStart` field of the `ContentNode` that you pass to `video.content`. [Roku's docs][PlayStart] say that "PlayStart defines the start position of the content, in seconds." Ok - but what is the start position of a live stream? There are ways to actually calculate it, but most folks find that it is easier to simply set it to "a sufficiently large number". What constitutes a sufficiently large number? Many folks use 99999 or something like that, but since `PlayStart` is an `int`, the largest possibly value it can take it is 2147483647, so that is what I use - just to be safe 😎

    content = CreateObject("roSGNode", "ContentNode")
    content.setFields({
        streamformat: "hls"
        url: "http://your.video.url"
        live: true
        playstart: 2147483647
    })
    video.content = content

[Example1]: https://forums.roku.com/viewtopic.php?t=39589
[Example2]: https://forums.roku.com/viewtopic.php?t=66558
[Example3]: https://forums.roku.com/viewtopic.php?t=37622
[Example4]: https://stackoverflow.com/questions/50021092/roku-is-playing-about-a-1-hour-delay-on-the-live-stream
[PlayStart]: https://sdkdocs.roku.com/display/sdkdoc/Content+Meta-Data
