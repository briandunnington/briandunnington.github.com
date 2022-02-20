Title: Roku: Handling roInputEvent
Date: 2019.03.19
Summary: How to handle roInputEvent messages without restarting your app
Category: Patterns

Many of Roku's latest round of certification requirements go into effect on March 31, 2019 so developers are scrambling to update their apps. One of the new requirements has to do with handling deep links while the channel is already running. Section 5.2 of the [current certification requirements][CertReq] states:

> 5.2 When the channel is already running, direct playback commands will deep link to content in the channel without requiring a channel launch delay by using roInputEvent. To support this, channels must process roInputEvent the same way deep link parameters are passed through the main entry point on launch. (Required after March 31, 2019)

Implementing this is actually pretty straight-forward. In your `main.brs`, just add something like this:

    input = CreateObject("roInput")
    input.setMessagePort(port)

The in your `while` loop, you can handle the incoming message like this:

    while(true)
        msg = port.GetMessage()
        msgType = type(msg)
        if msgType = "roInputEvent"
            info = msg.getInfo()
            'info is now just like args passed to your Main() method
            'it contains mediaType and contentId properties, so you
            'can use your normal deep-link handling code
            handleDeepLink(info)
        end if
    end while

Note that the `getInfo()` method returns an object that looks just like the `args` that are passed to your `Main()` method - it contains the same `mediaType` and `contentId` properties so you can usually just re-use your existing deep-link handling code.

## Testing it out

You can use the [External Control Protocol (ECP)][ECP] to test our your implementation. Using a tool like Postman or curl, you can construct a url that will send the appropriate `input` event. The command should look like:

    curl -d '' 'http://YOUR-IP:8060/input?mediaType=movie&contentId=happy-gilmore'

The command needs to be sent as a `POST` (hence the `-d ''`) and needs to include the `mediaType` and `contentId` parameters. Make sure your app is already running and then issue the command - your app should receive the command and handle the deep-link appropriately without causing the channel to relaunch.

> **NOTE:** If you try this out and your app still relaunches when receiving the input event, double check your url. Specifically, **do not** include `/dev` at the end - input events are only applicable for the currently-running channel so you should not include the channel ID in the url.

## Bonus Tip

The Roku [documentation about manifest files][manifest] lists `supports_input_launch` with this description:

> The Roku mobile app (previously using roInputEvent) was changed recently to always use 'launch' events rather than 'input' events.
That means that Deep Link launches should always work, and with this attribute, the launch will no longer force exiting and restarting the app if it was already running.

I have seen folks get confused and think this is required to support `roInputEvent` (I mean, the property is called *supports input launch* so it is understandable.) But if you read that description carefully, what it is actually saying is that if you send a `launch` command, your app will no longer restart if you have set this flag.

To try this out, try sending this command to your unmodified app:

    curl -d '' 'http://YOUR-IP:8060/launch/dev?mediaType=movie&contentId=happy-gilmore'

(Note that the channel ID (`/dev`) is required for `launch` events). Even if your app was already running, it should have exited and relaunched. Now add the following to your manifest file and launch your app:

    supports_input_launch=1

Now send the curl command again - this time your app should *not* restart: the manifest flag instructed the Roku to treat the `launch` event like an `input` event so it won't relaunch the channel.



[CertReq]: https://developer.roku.com/develop/channel-store/certification
[ECP]: https://sdkdocs.roku.com/display/sdkdoc/External+Control+API#ExternalControlAPI-input
[manifest]: https://sdkdocs.roku.com/display/sdkdoc/Roku+Channel+Manifest#RokuChannelManifest-LaunchRequirementAttributes
