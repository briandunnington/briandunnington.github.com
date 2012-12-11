Title: GridView in a ScrollViewer
Date: 2012.12.10
Summary: How to use a GridView inside of a ScrollViewer in a Windows Store app while still retaining mouse-wheel scrolling and swipe-to-select behavior.

<!-- Main hero unit for a primary marketing message or call to action -->
<div class="hero-unit">
<h1>GridView in a ScrollViewer</h1>
<p>How to use a GridView inside of a ScrollViewer in a Windows Store app while still retaining mouse-wheel scrolling and swipe-to-select behavior.</p>
<!--<p><a class="btn btn-primary btn-large">Learn more &raquo;</a></p>-->
</div>

Here is the scenario: You are building a [Windows Store][WindowsStore] app and you want to use a [GridView][] to lay out groups of items in your app. However, you want each group to use a different layout for the individual items.

You might assume that you want to use the [GroupStyleSelector][] property, since the documentation says:

> The GroupStyleSelector returns different GroupStyle values to use for content based on the characteristics of that content.

However, it turns out that [GroupStyleSelector doesn't apply different styles to different groups][GroupStyleSelectorNoWorky]. Instead, it is intended to apply a different style to *all* groups conditionally (for instance, when the app is changed to Snapped view).

OK, so GroupStyleSelector is out. How about just putting each group of items in its own GridView and putting all of that in a big horizontal StackPanel wrapped in a ScrollViewer? (Hold on while I go try it.....) - Yup, that does the trick. Easy peasy.

Wait, something is not right. On a touch device, you can swipe back and forth to scroll and things are fine, but scrolling using the mouse wheel doesnt work. Turns out that the cause is that GridView contains its own ScrollViewer internally and that ScrollViewer swallows up the mouse wheel events.

Some [deeper digging][DeeperDigging] suggests that we can re-template the GridView to remove the inner ScrollViewer. Easy enough; we can add something like this in our App.xaml:

    <ControlTemplate x:Key="NonScrollingGridView" TargetType="GridView">
        <Border BorderBrush="{TemplateBinding BorderBrush}" BorderThickness="{TemplateBinding BorderThickness}" Background="{TemplateBinding Background}">
            <ItemsPresenter HeaderTemplate="{TemplateBinding HeaderTemplate}" Header="{TemplateBinding Header}" HeaderTransitions="{TemplateBinding HeaderTransitions}" Padding="{TemplateBinding Padding}"/>
        </Border>
    </ControlTemplate>

A quick check reveals that mouse wheel scrolling is now working! So that is that.

Or is it? Now back on the touch-enabled device, swipe-to-select is no longer working. Frustratingly, the internal ScrollViewer is required for swipe-to-select, but breaks mouse wheel scrolling when nested. So where does that leave us?

The solution (and I promise this is the actual solution this time) was suggested by [user Poleg on this MSDN thread][RealSolution]: we need to leave the internal ScrollViewer in place but prevent it from swallowing the mouse wheel events. Luckily, UIElement provides the [AddHandler][] method that we can use for situations like this. The last parameter is a boolean named  <code>handledEventsToo</code> which lets us hook into routed events even if a previous element already handled the event. So, we can subclass GridView (or VariableSizedWrapGrid or any other Panel with built-in scrolling), hook the OnPointerWheelChanged event using the AddHandler method, and set <code>e.Handled = false</code>:

    public class CustomGridView : GridView
    {
        protected override void OnApplyTemplate()
        {
            base.OnApplyTemplate();
            var sv = this.GetTemplateChild("ScrollViewer") as UIElement;
            if (sv != null)
                sv.AddHandler(UIElement.PointerWheelChangedEvent, new PointerEventHandler(OnPointerWheelChanged), true);
        }

        private void OnPointerWheelChanged(object sender, PointerRoutedEventArgs e)
        {
            e.Handled = false;
        }
    }

You can use this CustomGridView inside of nested ScrollViewers and it will provide touch scrolling, mouse wheel scrolling, *and* swipe-to-select behavior. Hope that helps somebody who was getting frustrated with it like I was.


[WindowsStore]: http://www.windowsstore.com/
[GridView]: http://msdn.microsoft.com/en-us/library/windows/apps/xaml/hh780618.aspx
[GroupStyleSelector]: http://msdn.microsoft.com/en-us/library/windows/apps/xaml/windows.ui.xaml.controls.itemscontrol.groupstyleselector.aspx
[GroupStyleSelectorNoWorky]: http://stackoverflow.com/questions/11418511/how-can-i-make-groups-in-a-metro-gridview-use-different-layouts
[DeeperDigging]: http://stackoverflow.com/questions/10737656/metro-style-scrolling-with-mouse-wheel
[RealSolution]: http://social.msdn.microsoft.com/Forums/en-US/winappswithcsharp/thread/18742227-4be6-4f0a-be2b-061f80c8e33c/
[AddHandler]: http://msdn.microsoft.com/en-us/library/ms598899.aspx