Title: Streaming live drone footage using Azure Media Services
Date: 2018.02.06
Summary: Directly from your drone to the world, via the the cloud

<div class="hero-unit">
<h1>Streaming live drone footage using Azure Media Services</h1>
<p>Directly from your drone to the world, via the the cloud</p>
</div>

<style>
.maincontent h4 { margin: 16px 0 8px 0; }
.img img { display: block; margin-bottom: 16px; border: solid 1px #666666; }
.img span { text-align: center; }
blockquote p { font-size: 90%; }
blockquote pre { margin: 10px 0; }
</style>

I have a [DJI Phantom 3 Standard][Phantom] drone and love to fly it around. You can get some really amazing shots from all sorts of unique perspectives. 

<div class="img">
<img src="images/dronecast_phantom.jpg"/>
</div>

Usually, I simply use the DJI GO app to record video during flight and then download it from the SD card later for processing/editing. But DJI also offers the ability to live-stream the video to sites like YouTube and Facebook. That is pretty cool, but I wanted to live-stream the video to a custom site. The DJI GO app does support streaming to a custom RTMP endpoint, and since Azure Media Services allows ingesting live video via RTMP, I figured it would be a great fit.

Microsoft's [own documentation][livedoc] does an OK job of explaining how to set up live video ingest, but it glosses over a few important bits of info. There are also a couple of gotchas when configuring the drone streaming settings, so here is the complete and accurate steps to stream live video from your drone.

### 1. Create a Media Services account

<div class="img">
<img src="images/dronecast_createmediaservice.png"/>
</div>

### 2. Create a live streaming channel

<div class="img">
<img src="images/dronecast_step1.png"/>
</div>

<div class="img">
<img src="images/dronecast_step2.png"/>
</div>

<div class="img">
<img src="images/dronecast_step3.png"/>
</div>

<div class="img">
<img src="images/dronecast_step4.png"/>
</div>

### 3. Copy the resulting RTMP url

<div class="img">
<img src="images/dronecast_rtmpurl.png"/>
</div>

### 4. Create a new live event

<div class="img">
<img src="images/dronecast_liveevent.png"/>
</div>

### 5. Enable the Streaming Endpoint

<div class="img">
<img src="images/dronecast_enableendpoint.png"/>
</div>

### 6. Start the channel

<div class="img">
<img src="images/dronecast_startchannel.png"/>
</div>

At this point, Azure is ready to ingest the incoming video stream and encode it into multi-bitrate streams that can be consumed by clients.

Over in the DJI GO app...

### 1. General Settings > Select Live Broadcast Platform

<div class="img">
<img src="images/dronecast_djigo1.png"/>
</div>

### 2. Choose RTMP

<div class="img">
<img src="images/dronecast_djigo2.png"/>
</div>

### 3. Enter the RTMP url

<div class="img">
<img src="images/dronecast_djigo3.png"/>
</div>

**NOTE:** You must add an additional `/mystream` to the end of the url that you copied from the Azure portal. The value is not important, but the ingest will not work without the extra path info. 

<div class="img">
<img src="images/dronecast_djigo4.png"/>
</div>

<div class="img">
<img src="images/dronecast_djigo5.png"/>
</div>

When you are live streaming, the DJI GO app will show an indicator in the upper left corner.

At this point, you are broadcasting your live stream to the world. The Azure portal has a handy feature that lets you view the live stream directly in the portal (as well as providing the live stream url).

<div class="img">
<img src="images/dronecast_liveview.png"/>
</div>


[Phantom]: https://www.dji.com/phantom-3-standard
[livedoc]: https://docs.microsoft.com/en-us/azure/media-services/media-services-portal-creating-live-encoder-enabled-channel
