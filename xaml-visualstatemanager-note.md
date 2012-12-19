Title: Xaml VisualStateManager weirdness
Date: 2012.12.18
Summary: Sometimes GoToState() just wont work or works intermittently. Here is a note to myself with the answer.

<!-- Main hero unit for a primary marketing message or call to action -->
<div class="hero-unit">
<h1>Xaml VisualStateManager weirdness</h1>
<p>Sometimes GoToState() just wont work or works intermittently. Here is a note to myself with the answer.</p>
<!--<p><a class="btn btn-primary btn-large">Learn more &raquo;</a></p>-->
</div>

This is more a note to my future self, but maybe some other folks will find it helpful as well.

Recently, I was developing a custom templated control in a Windows Phone app and the template defined several VisualStates. In the control's code, I used the [`VisualStateManager.GoToState()`][GoToState] method to switch between the states when necessary. However, sometimes the state would not change. It wasnt reproducible consistently - sometimes it would change states back and forth just fine, and other times it would not change states at all.

After much debugging, lots of trial-and-error, and finally a bit of a revelation, I discovered the root cause:

> If you call `VisualStateManager.GoToState()` before `OnApplyTemplate()` has ran, it seems to corrupt the VisualStateManager and subsequent attempts to change state will become erratic.

In my particular case, I had bound some properties in the template to [DependencyProperties][DependencyProperty] in the control and was using the [PropertyChangedCallback] to update the visual state. However, this meant that databound values were set on the control early in its lifecycle and causes this issue to occur.

The fix for me was to simply ignore any calls to my internal `UpdateState()` method until `OnApplyTemplate()` had been called and then to update the state based on the current property values:

	public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        hasLoaded = true;
        UpdateState(this.IsActive);
    }

    void UpdateState(bool isActive)
    {
        if (hasLoaded)
        {
            string state = isActive ? "Active" : "Inactive";
            System.Windows.VisualStateManager.GoToState(this, state, true);
        }
    }


[GoToState]: http://msdn.microsoft.com/en-us/library/dd991369.aspx
[DependencyProperty]: http://msdn.microsoft.com/en-us/library/ms752914.aspx
[PropertyChangedCallback]: http://msdn.microsoft.com/en-us/library/system.windows.propertychangedcallback.aspx
