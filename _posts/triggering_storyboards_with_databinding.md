Title: Triggering Storyboards with data binding
Date: 2014.04.24
Summary: A clean way for your viewmodel's properties to trigger storyboard animations

<div class="hero-unit">
<h1>Triggering Storyboards with data binding</h1>
<p>A clean way for your viewmodel's properties to trigger storyboard animations</p>
</div>

Creating animations in XAML is pretty easy using [Storyboards], but after you create them, you have to trigger them somehow. There are a bunch of ways to do this, but each has its own shortcomings if you want to trigger your animation based on viewmodel changes:

_DataTrigger_ & _Trigger_ - Not supported in WinRT apps so they are non-starters.

_EventTrigger_ - Triggers your animation in response to an event. These are still supported in WinRT but they can only hook up to the `Loaded` event so it is effectively inaccessible from a viewmodel.

_Storyboard.Begin() method_ - straight to the point, but not easy to databind to since it is a method.

_VisualStateManager_ - This is essentially the replacement for DataTriggers, but again, it requires calling a method (`VisualStateManager.GoToState()`) to trigger it.

_Behaviors_ - Code like [this][Behaviors] can trigger a storyboard using a custom behavior, but (and maybe this is just me) I rarely use behaviors and find them kind of clunky for some reason. Just my opinion.

So where does that leave us? How about this:

	<Storyboard common:StoryboardHelper.BeginIf="{Binding IsLoggedIn}">
        <DoubleAnimationUsingKeyFrames Storyboard.TargetProperty="(UIElement.RenderTransform).(CompositeTransform.TranslateY)" Storyboard.TargetName="StartSessionOverlay">
            <EasingDoubleKeyFrame KeyTime="0:0:0" Value="640"/>
            <EasingDoubleKeyFrame KeyTime="0:0:0.3" Value="0"/>
        </DoubleAnimationUsingKeyFrames>
    </Storyboard>

Using a custom [attached property][AttachedProperty], we can set up a mechanism that allows declarative databinding in the XAML and yet runs arbitrary code behind the scenes. The implementation is straight-forward:

<pre><code class="csharp"><span class="keyword">using</span> System;
<span class="keyword">using</span> System.Collections.Generic;
<span class="keyword">using</span> System.Text;
<span class="keyword">using</span> Windows.UI.Xaml;
<span class="keyword">using</span> Windows.UI.Xaml.Controls;
<span class="keyword">using</span> Windows.UI.Xaml.Media.Animation;

<span class="keyword">namespace</span> Demo
{
    <span class="keyword">public</span> <span class="keyword">class</span> StoryboardHelper : DependencyObject
    {
        <span class="keyword">public</span> <span class="keyword">static</span> <span class="keyword">bool</span> GetBeginIf(DependencyObject obj)
	    {
            <span class="keyword">return</span> (<span class="keyword">bool</span>)obj.GetValue(BeginIfProperty);
	    }

        <span class="keyword">public</span> <span class="keyword">static</span> <span class="keyword">void</span> SetBeginIf(DependencyObject obj, <span class="keyword">bool</span> <span class="keyword">value</span>)
	    {
            obj.SetValue(BeginIfProperty, <span class="keyword">value</span>);
	    }

        <span class="keyword">public</span> <span class="keyword">static</span> <span class="keyword">readonly</span> DependencyProperty BeginIfProperty = DependencyProperty.RegisterAttached(<span class="string">&quot;BeginIf&quot;</span>, <span class="keyword">typeof</span>(<span class="keyword">bool</span>), <span class="keyword">typeof</span>(StoryboardHelper), <span class="keyword">new</span> PropertyMetadata(<span class="keyword">false</span>, BeginIfPropertyChangedCallback));

        <span class="keyword">private</span> <span class="keyword">static</span> <span class="keyword">void</span> BeginIfPropertyChangedCallback(DependencyObject s, DependencyPropertyChangedEventArgs e)
        {
            <span class="keyword">var</span> storyboard = s <span class="keyword">as</span> Storyboard;
            <span class="keyword">if</span> (storyboard == <span class="keyword">null</span>)
                <span class="keyword">throw</span> <span class="keyword">new</span> InvalidOperationException(<span class="string">&quot;This attached property only supports Storyboards.&quot;</span>);

            <span class="keyword">var</span> begin = (<span class="keyword">bool</span>)e.NewValue;
            <span class="keyword">if</span> (begin) storyboard.Begin();
            <span class="keyword">else</span> storyboard.Stop();
        }
    }
}
</code></pre>

Attached properties have been solving a lot of little things like this for me lately so I will probably write about them some more in the future.



[Storyboards]: http://msdn.microsoft.com/en-us/library/ms742868%28v=vs.110%29.aspx
[Behaviors]: http://dotnetbyexample.blogspot.com/2012/10/a-winrt-behavior-to-start-storyboard-on.html
[AttachedProperty]: http://msdn.microsoft.com/en-us/library/windows/apps/hh965327.aspx