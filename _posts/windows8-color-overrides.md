Title: Windows 8 Store Apps Color Overrides
Date: 2012.12.19
Summary: Tired of that purple color in your ComboBoxes, ListBoxes, and GridViews? Here is how to change it without retemplating all of the controls.

<!-- Main hero unit for a primary marketing message or call to action -->
<div class="hero-unit">
<h1>Windows 8 Store Apps Color Overrides</h1>
<p>Tired of that purple color in your ComboBoxes, ListBoxes, and GridViews? Here is how to change it without retemplating all of the controls.</p>
<!--<p><a class="btn btn-primary btn-large">Learn more &raquo;</a></p>-->
</div>

So your Windows 8 Store (née Metro) is coming along and you are trying to polish up the look and feel. However, that annoying purple highlight color keeps popping up in unexpected places, like the background for selected items in ComboBoxes and ListBoxes, or in the border for selected items in a GridView. 

![Purple](http://1wpf.files.wordpress.com/2012/06/example.png)

Purple might work for some apps, but you are a perfectionist and you gotta have the exact shade of green that matches your logo. Some of the controls expose properties that let you set the color in your xaml and you could set up a `<Style>` rule to apply them, but then you have to always remember to apply the style and some of the controls don't even expose the necessary properties to override the color, so what to do?

Maybe everybody out there other than me already knows this, but the list of pre-defined control colors can be found at:
`%PROGRAMFILES(x86)%\Windows Kits\8.0\Include\WinRT\Xaml\Design\generic.xaml`

In there, there are a few different sections for the Dark theme, Light theme, and High-Contrast theme. Looking through any of them, you can see a section called:

	<!--
	******************************************************************
	DEFAULT COMMON CONTROL COLORS
	******************************************************************
	-->;


That section contains a long list of all of the colors used by the built-in controls. Most of them are various shades and opacities of black or white and should be good for most apps. But I went through and pulled out all of the items that defined a purplish color so you can easily see which items to override:

    <!-- Purple system brushes -->
    <SolidColorBrush x:Key="ComboBoxItemSelectedBackgroundThemeBrush" Color="#FF4617B4" />
    <SolidColorBrush x:Key="ComboBoxItemSelectedPointerOverBackgroundThemeBrush" Color="#FF5F37BE" />
    <SolidColorBrush x:Key="ComboBoxSelectedBackgroundThemeBrush" Color="#FF4617B4" />
    <SolidColorBrush x:Key="ComboBoxSelectedPointerOverBackgroundThemeBrush" Color="#FF5F37BE" />
    <SolidColorBrush x:Key="HyperlinkForegroundThemeBrush" Color="#FF9C72FF" />
    <SolidColorBrush x:Key="HyperlinkPointerOverForegroundThemeBrush" Color="#CC9C72FF" />
    <SolidColorBrush x:Key="HyperlinkPressedForegroundThemeBrush" Color="#999C72FF" />
    <SolidColorBrush x:Key="ListBoxItemSelectedBackgroundThemeBrush" Color="#FF4617B4" />
    <SolidColorBrush x:Key="ListBoxItemSelectedPointerOverBackgroundThemeBrush" Color="#FF5F37BE" />
    <SolidColorBrush x:Key="ListViewItemDragBackgroundThemeBrush" Color="#994617B4" />
    <SolidColorBrush x:Key="ListViewItemSelectedBackgroundThemeBrush" Color="#FF4617B4" />
    <SolidColorBrush x:Key="ListViewItemSelectedPointerOverBackgroundThemeBrush" Color="#FF5F37BE" />
    <SolidColorBrush x:Key="ListViewItemSelectedPointerOverBorderThemeBrush" Color="#FF5F37BE" />
    <SolidColorBrush x:Key="ProgressBarForegroundThemeBrush" Color="#FF5B2EC5" />
    <SolidColorBrush x:Key="ProgressBarIndeterminateForegroundThemeBrush" Color="#FF8A57FF" />
    <SolidColorBrush x:Key="SliderTrackDecreaseBackgroundThemeBrush" Color="#FF5B2EC5" />
    <SolidColorBrush x:Key="SliderTrackDecreasePointerOverBackgroundThemeBrush" Color="#FF724BCD" />
    <SolidColorBrush x:Key="SliderTrackDecreasePressedBackgroundThemeBrush" Color="#FF8152EF" />
    <SolidColorBrush x:Key="ToggleSwitchCurtainBackgroundThemeBrush" Color="#FF5729C1" />
    <SolidColorBrush x:Key="ToggleSwitchCurtainPointerOverBackgroundThemeBrush" Color="#FF6E46CA" />
    <SolidColorBrush x:Key="ToggleSwitchCurtainPressedBackgroundThemeBrush" Color="#FF7E4FEC" />

In order to make sure these get set before any control templates that use them, I simply added them to their own ResourceDictionary xaml file and included it directly above the `Common\StandardStyles.xaml` in `App.xaml.cs`:

    <ResourceDictionary.MergedDictionaries>
        <ResourceDictionary Source="Common/ColorOverrides.xaml"/>

        <!-- 
            Styles that define common aspects of the platform look and feel
            Required by Visual Studio project and item templates
            -->
        <ResourceDictionary Source="Common/StandardStyles.xaml"/>
    </ResourceDictionary.MergedDictionaries>

Hope that helps somebody check of that one last to-do item so they can submit their app, sit back, and start raking in the millions =)
