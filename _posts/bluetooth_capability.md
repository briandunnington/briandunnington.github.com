Title: Declare Bluetooth capability for Windows Store and Windows Phone 8.1 apps
Date: 2014.04.04
Summary: 

<div class="hero-unit">
<h1>Declare Bluetooth capability for Windows Store and Windows Phone 8.1 apps</h1>
<p></p>
</div>

If anybody ever needs to communicate with a Bluetooth device from a Windows Store or Windows Phone 8.1 app, you have to manually add the following to your `package.appxmanifest` file (it cannot be added through any UI):

<pre>
<span style='color:#a65700; '>&lt;</span><span style='color:#666616; '>m2</span><span style='color:#800080; '>:</span><span style='color:#5f5035; '>DeviceCapability</span> <span style='color:#274796; '>Name</span><span style='color:#808030; '>=</span><span style='color:#0000e6; '>"</span><span style='color:#0000e6; '>bluetooth.rfcomm</span><span style='color:#0000e6; '>"</span><span style='color:#a65700; '>></span>
	<span style='color:#a65700; '>&lt;</span><span style='color:#666616; '>m2</span><span style='color:#800080; '>:</span><span style='color:#5f5035; '>Device</span> <span style='color:#274796; '>Id</span><span style='color:#808030; '>=</span><span style='color:#0000e6; '>"</span><span style='color:#0000e6; '>any</span><span style='color:#0000e6; '>"</span><span style='color:#a65700; '>></span>
		<span style='color:#a65700; '>&lt;</span><span style='color:#666616; '>m2</span><span style='color:#800080; '>:</span><span style='color:#5f5035; '>Function</span> <span style='color:#274796; '>Type</span><span style='color:#808030; '>=</span><span style='color:#0000e6; '>"</span><span style='color:#0000e6; '>name:serialPort</span><span style='color:#0000e6; '>"</span> <span style='color:#a65700; '>/></span>
	<span style='color:#a65700; '>&lt;/</span><span style='color:#666616; '>m2</span><span style='color:#800080; '>:</span><span style='color:#5f5035; '>Device</span><span style='color:#a65700; '>></span>
<span style='color:#a65700; '>&lt;/</span><span style='color:#666616; '>m2</span><span style='color:#800080; '>:</span><span style='color:#5f5035; '>DeviceCapability</span><span style='color:#a65700; '>></span>
</pre>

Don't ask how long that took to figure out =)

__UPDATE:__ I finally found the [documentation on MSDN][MSDN] for this and it describes the valid values for the various attributes.

[MSDN]: http://msdn.microsoft.com/en-us/library/windows/apps/dn263090.aspx