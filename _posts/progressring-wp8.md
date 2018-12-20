Title: ProgressRing for Windows Phone 8
Date: 2013.01.11
Summary: A Windows-8-style ProgressRing for Windows Phone 8

<div class="hero-unit">
<h1>ProgressRing for Windows Phone 8</h1>
<p>A Windows-8-style ProgressRing for Windows Phone 8</p>
</div>

The [ProgressRing][] control for Metro (Windows Store style) apps is a great way to show the familiar loading spinner while your app performs some long-running action. But what about Windows Phone 8? There is the [ProgressBar][] control (which has been updated for WP8 to [fix some performance issues][PerformanceProgressBar]), but I wanted the round spinny thing instead of the flat movey thing, so I created it.

The code below consists of a `.cs` file that contains the guts of the control, as well as the `<Style>` definition. A couple of fun things to note:

* The `<Style>` is the exact style copied from the Windows 8 resources, so the control looks and behaves identically

* Note the check for `hasAppliedTemplate` - this is required to avoid [some wonky behavior][wonky] that I blogged about recently where `VisualStateManager` doesnt like to be called before `OnApplyTemplate()` has been called.

Anyway, next time you need some spinning-dots goodness in your WP8 app, give it a try and let me know what you think.

 
*ProgressRing.cs*

<pre class="code"><code class="csharp"><span class="keyword">using</span> System;
<span class="keyword">using</span> System.Collections.Generic;
<span class="keyword">using</span> System.Linq;
<span class="keyword">using</span> System.Text;
<span class="keyword">using</span> System.Threading.Tasks;
<span class="keyword">using</span> System.Windows;
<span class="keyword">using</span> System.Windows.Controls;

<span class="keyword">namespace</span> Monsters.WindowsPhone.Controls
{
    <span class="keyword">public</span> <span class="keyword">class</span> ProgressRing : Control
    {
        <span class="keyword">bool</span> hasAppliedTemplate = <span class="keyword">false</span>;

        <span class="keyword">public</span> ProgressRing()
        {
            <span class="keyword">this</span>.DefaultStyleKey = <span class="keyword">typeof</span>(ProgressRing);
            TemplateSettings = <span class="keyword">new</span> TemplateSettingValues(60);
        }

        <span class="keyword">public</span> <span class="keyword">override</span> <span class="keyword">void</span> OnApplyTemplate()
        {
            <span class="keyword">base</span>.OnApplyTemplate();
            hasAppliedTemplate = <span class="keyword">true</span>;
            UpdateState(<span class="keyword">this</span>.IsActive);
        }

        <span class="keyword">void</span> UpdateState(<span class="keyword">bool</span> isActive)
        {
            <span class="keyword">if</span> (hasAppliedTemplate)
            {
                <span class="keyword">string</span> state = isActive ? <span class="string">&quot;Active&quot;</span> : <span class="string">&quot;Inactive&quot;</span>;
                System.Windows.VisualStateManager.GoToState(<span class="keyword">this</span>, state, <span class="keyword">true</span>);
            }
        }

        <span class="keyword">protected</span> <span class="keyword">override</span> System.Windows.Size MeasureOverride(System.Windows.Size availableSize)
        {
            <span class="keyword">var</span> width = 100D;
            <span class="keyword">if</span>(!System.ComponentModel.DesignerProperties.IsInDesignTool)
                width = <span class="keyword">this</span>.Width != <span class="keyword">double</span>.NaN ? <span class="keyword">this</span>.Width : availableSize.Width;
            TemplateSettings = <span class="keyword">new</span> TemplateSettingValues(width);
            <span class="keyword">return</span> <span class="keyword">base</span>.MeasureOverride(availableSize);
        }

        <span class="keyword">public</span> <span class="keyword">bool</span> IsActive
        {
            <span class="keyword">get</span> { <span class="keyword">return</span> (<span class="keyword">bool</span>)GetValue(IsActiveProperty); }
            <span class="keyword">set</span> { SetValue(IsActiveProperty, <span class="keyword">value</span>); }
        }

        <span class="comment">// Using a DependencyProperty as the backing store for IsActive.  This enables animation, styling, binding, etc...</span>
        <span class="keyword">public</span> <span class="keyword">static</span> <span class="keyword">readonly</span> DependencyProperty IsActiveProperty =
            DependencyProperty.Register(<span class="string">&quot;IsActive&quot;</span>, <span class="keyword">typeof</span>(<span class="keyword">bool</span>), <span class="keyword">typeof</span>(ProgressRing), <span class="keyword">new</span> PropertyMetadata(<span class="keyword">false</span>, <span class="keyword">new</span> PropertyChangedCallback(IsActiveChanged)));

        <span class="keyword">private</span> <span class="keyword">static</span> <span class="keyword">void</span> IsActiveChanged(DependencyObject d, DependencyPropertyChangedEventArgs args)
        {
            <span class="keyword">var</span> pr = (ProgressRing) d;
            <span class="keyword">var</span> isActive = (<span class="keyword">bool</span>)args.NewValue;
            pr.UpdateState(isActive);
        }

        <span class="keyword">public</span> TemplateSettingValues TemplateSettings
        {
            <span class="keyword">get</span> { <span class="keyword">return</span> (TemplateSettingValues)GetValue(TemplateSettingsProperty); }
            <span class="keyword">set</span> { SetValue(TemplateSettingsProperty, <span class="keyword">value</span>); }
        }

        <span class="comment">// Using a DependencyProperty as the backing store for TemplateSettings.  This enables animation, styling, binding, etc...</span>
        <span class="keyword">public</span> <span class="keyword">static</span> <span class="keyword">readonly</span> DependencyProperty TemplateSettingsProperty =
            DependencyProperty.Register(<span class="string">&quot;TemplateSettings&quot;</span>, <span class="keyword">typeof</span>(TemplateSettingValues), <span class="keyword">typeof</span>(ProgressRing), <span class="keyword">new</span> PropertyMetadata(<span class="keyword">new</span> TemplateSettingValues(100)));


        <span class="keyword">public</span> <span class="keyword">class</span> TemplateSettingValues : System.Windows.DependencyObject
        {
            <span class="keyword">public</span> TemplateSettingValues(<span class="keyword">double</span> width)
            {
                MaxSideLength = 400;
                EllipseDiameter = width/10;
                EllipseOffset = <span class="keyword">new</span> System.Windows.Thickness(EllipseDiameter);
            }

            <span class="keyword">public</span> <span class="keyword">double</span> MaxSideLength
            {
                <span class="keyword">get</span> { <span class="keyword">return</span> (<span class="keyword">double</span>)GetValue(MaxSideLengthProperty); }
                <span class="keyword">set</span> { SetValue(MaxSideLengthProperty, <span class="keyword">value</span>); }
            }

            <span class="comment">// Using a DependencyProperty as the backing store for MaxSideLength.  This enables animation, styling, binding, etc...</span>
            <span class="keyword">public</span> <span class="keyword">static</span> <span class="keyword">readonly</span> DependencyProperty MaxSideLengthProperty =
                DependencyProperty.Register(<span class="string">&quot;MaxSideLength&quot;</span>, <span class="keyword">typeof</span>(<span class="keyword">double</span>), <span class="keyword">typeof</span>(TemplateSettingValues), <span class="keyword">new</span> PropertyMetadata(0D));

            <span class="keyword">public</span> <span class="keyword">double</span> EllipseDiameter
            {
                <span class="keyword">get</span> { <span class="keyword">return</span> (<span class="keyword">double</span>)GetValue(EllipseDiameterProperty); }
                <span class="keyword">set</span> { SetValue(EllipseDiameterProperty, <span class="keyword">value</span>); }
            }

            <span class="comment">// Using a DependencyProperty as the backing store for EllipseDiameter.  This enables animation, styling, binding, etc...</span>
            <span class="keyword">public</span> <span class="keyword">static</span> <span class="keyword">readonly</span> DependencyProperty EllipseDiameterProperty =
                DependencyProperty.Register(<span class="string">&quot;EllipseDiameter&quot;</span>, <span class="keyword">typeof</span>(<span class="keyword">double</span>), <span class="keyword">typeof</span>(TemplateSettingValues), <span class="keyword">new</span> PropertyMetadata(0D));

            <span class="keyword">public</span> Thickness EllipseOffset
            {
                <span class="keyword">get</span> { <span class="keyword">return</span> (Thickness)GetValue(EllipseOffsetProperty); }
                <span class="keyword">set</span> { SetValue(EllipseOffsetProperty, <span class="keyword">value</span>); }
            }

            <span class="comment">// Using a DependencyProperty as the backing store for EllipseOffset.  This enables animation, styling, binding, etc...</span>
            <span class="keyword">public</span> <span class="keyword">static</span> <span class="keyword">readonly</span> DependencyProperty EllipseOffsetProperty =
                DependencyProperty.Register(<span class="string">&quot;EllipseOffset&quot;</span>, <span class="keyword">typeof</span>(Thickness), <span class="keyword">typeof</span>(TemplateSettingValues), <span class="keyword">new</span> PropertyMetadata(<span class="keyword">new</span> Thickness()));
        }
    }
}
</code></pre>

*ProgressRing Style*

	<!-- Default style for Windows.UI.Xaml.Controls.ProgressRing -->
	<Style TargetType="controls:ProgressRing">
	    <Setter Property="Foreground" Value="{StaticResource AccentBrush}" />
	    <Setter Property="IsHitTestVisible" Value="False" />
	    <Setter Property="HorizontalAlignment" Value="Center" />
	    <Setter Property="VerticalAlignment" Value="Center" />
	    <Setter Property="MinHeight" Value="20" />
	    <Setter Property="MinWidth" Value="20" />
	    <Setter Property="IsTabStop" Value="False" />
	    <Setter Property="Template">
	        <Setter.Value>
	            <ControlTemplate TargetType="controls:ProgressRing">
	                <Border x:Name="ProgressRingRoot" Background="{TemplateBinding Background}"
	                    BorderThickness="{TemplateBinding BorderThickness}"
	                    BorderBrush="{TemplateBinding BorderBrush}">
	                    <Border.Resources>
	                        <Style x:Key="ProgressRingEllipseStyle" TargetType="Ellipse">
	                            <Setter Property="Opacity" Value="0" />
	                            <Setter Property="HorizontalAlignment" Value="Left" />
	                            <Setter Property="VerticalAlignment" Value="Top" />
	                        </Style>
	                    </Border.Resources>
	                    <VisualStateManager.VisualStateGroups>
	                        <VisualStateGroup x:Name="SizeStates">
	                            <VisualState x:Name="Large">
	                                <Storyboard>
	                                    <ObjectAnimationUsingKeyFrames Duration="0"
	                                                                Storyboard.TargetName="SixthCircle"
	                                                                Storyboard.TargetProperty="Visibility">
	                                        <DiscreteObjectKeyFrame KeyTime="0">
	                                            <DiscreteObjectKeyFrame.Value>
	                                                <Visibility>Visible</Visibility>
	                                            </DiscreteObjectKeyFrame.Value>
	                                        </DiscreteObjectKeyFrame>
	                                    </ObjectAnimationUsingKeyFrames>
	                                </Storyboard>
	                            </VisualState>
	                            <VisualState x:Name="Small" />
	                        </VisualStateGroup>
	                        <VisualStateGroup x:Name="ActiveStates">
	                            <VisualState x:Name="Inactive" />
	                            <VisualState x:Name="Active">
	                                <Storyboard RepeatBehavior="Forever">
	                                    <ObjectAnimationUsingKeyFrames Duration="0"
	                                                                Storyboard.TargetName="Ring"
	                                                                Storyboard.TargetProperty="Visibility">
	                                        <DiscreteObjectKeyFrame KeyTime="0">
	                                            <DiscreteObjectKeyFrame.Value>
	                                                <Visibility>Visible</Visibility>
	                                            </DiscreteObjectKeyFrame.Value>
	                                        </DiscreteObjectKeyFrame>
	                                    </ObjectAnimationUsingKeyFrames>
	                                    <DoubleAnimationUsingKeyFrames
	                                    Storyboard.TargetName="E1"
	                                    Storyboard.TargetProperty="Opacity"
	                                    BeginTime="0">
	                                        <DiscreteDoubleKeyFrame KeyTime="0" Value="1" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.21" Value="1" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.22" Value="0" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.47" Value="0" />
	                                    </DoubleAnimationUsingKeyFrames>
	                                    <DoubleAnimationUsingKeyFrames
	                                    Storyboard.TargetName="E2"
	                                    Storyboard.TargetProperty="Opacity"
	                                    BeginTime="00:00:00.167">
	                                        <DiscreteDoubleKeyFrame KeyTime="0" Value="1" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.21" Value="1" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.22" Value="0" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.47" Value="0" />
	                                    </DoubleAnimationUsingKeyFrames>
	                                    <DoubleAnimationUsingKeyFrames
	                                    Storyboard.TargetName="E3"
	                                    Storyboard.TargetProperty="Opacity"
	                                    BeginTime="00:00:00.334">
	                                        <DiscreteDoubleKeyFrame KeyTime="0" Value="1" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.21" Value="1" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.22" Value="0" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.47" Value="0" />
	                                    </DoubleAnimationUsingKeyFrames>
	                                    <DoubleAnimationUsingKeyFrames
	                                    Storyboard.TargetName="E4"
	                                    Storyboard.TargetProperty="Opacity"
	                                    BeginTime="00:00:00.501">
	                                        <DiscreteDoubleKeyFrame KeyTime="0" Value="1" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.21" Value="1" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.22" Value="0" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.47" Value="0" />
	                                    </DoubleAnimationUsingKeyFrames>
	                                    <DoubleAnimationUsingKeyFrames
	                                    Storyboard.TargetName="E5"
	                                    Storyboard.TargetProperty="Opacity"
	                                    BeginTime="00:00:00.668">
	                                        <DiscreteDoubleKeyFrame KeyTime="0" Value="1" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.21" Value="1" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.22" Value="0" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.47" Value="0" />
	                                    </DoubleAnimationUsingKeyFrames>
	                                    <DoubleAnimationUsingKeyFrames
	                                    Storyboard.TargetName="E6"
	                                    Storyboard.TargetProperty="Opacity"
	                                    BeginTime="00:00:00.835">
	                                        <DiscreteDoubleKeyFrame KeyTime="0" Value="1" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.21" Value="1" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.22" Value="0" />
	                                        <DiscreteDoubleKeyFrame KeyTime="0:0:3.47" Value="0" />
	                                    </DoubleAnimationUsingKeyFrames>
	                                    <DoubleAnimationUsingKeyFrames
	                                    Storyboard.TargetName="E1R"
	                                    BeginTime="0"
	                                    Storyboard.TargetProperty="Angle">
	                                        <SplineDoubleKeyFrame KeyTime="0" Value="-110" KeySpline="0.13,0.21,0.1,0.7"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:0.433" Value="10" KeySpline="0.02,0.33,0.38,0.77"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:1.2" Value="93"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:1.617" Value="205" KeySpline="0.57,0.17,0.95,0.75"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:2.017" Value="357" KeySpline="0,0.19,0.07,0.72"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:2.783" Value="439"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:3.217" Value="585" KeySpline="0,0,0.95,0.37"/>
	                                    </DoubleAnimationUsingKeyFrames>
	                                    <DoubleAnimationUsingKeyFrames
	                                    Storyboard.TargetName="E2R"
	                                    BeginTime="00:00:00.167"
	                                    Storyboard.TargetProperty="Angle">
	                                        <SplineDoubleKeyFrame KeyTime="0" Value="-116" KeySpline="0.13,0.21,0.1,0.7"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:0.433" Value="4" KeySpline="0.02,0.33,0.38,0.77"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:1.2" Value="87"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:1.617" Value="199" KeySpline="0.57,0.17,0.95,0.75"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:2.017" Value="351" KeySpline="0,0.19,0.07,0.72"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:2.783" Value="433"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:3.217" Value="579" KeySpline="0,0,0.95,0.37"/>
	                                    </DoubleAnimationUsingKeyFrames>
	                                    <DoubleAnimationUsingKeyFrames
	                                    Storyboard.TargetName="E3R"
	                                    BeginTime="00:00:00.334"
	                                    Storyboard.TargetProperty="Angle">
	                                        <SplineDoubleKeyFrame KeyTime="0" Value="-122" KeySpline="0.13,0.21,0.1,0.7"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:0.433" Value="-2" KeySpline="0.02,0.33,0.38,0.77"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:1.2" Value="81"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:1.617" Value="193" KeySpline="0.57,0.17,0.95,0.75"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:2.017" Value="345" KeySpline="0,0.19,0.07,0.72"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:2.783" Value="427"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:3.217" Value="573" KeySpline="0,0,0.95,0.37"/>
	                                    </DoubleAnimationUsingKeyFrames>
	                                    <DoubleAnimationUsingKeyFrames
	                                    Storyboard.TargetName="E4R"
	                                    BeginTime="00:00:00.501"
	                                    Storyboard.TargetProperty="Angle">
	                                        <SplineDoubleKeyFrame KeyTime="0" Value="-128" KeySpline="0.13,0.21,0.1,0.7"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:0.433" Value="-8" KeySpline="0.02,0.33,0.38,0.77"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:1.2" Value="75"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:1.617" Value="187" KeySpline="0.57,0.17,0.95,0.75"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:2.017" Value="339" KeySpline="0,0.19,0.07,0.72"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:2.783" Value="421"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:3.217" Value="567" KeySpline="0,0,0.95,0.37"/>
	                                    </DoubleAnimationUsingKeyFrames>
	                                    <DoubleAnimationUsingKeyFrames
	                                    Storyboard.TargetName="E5R"
	                                    BeginTime="00:00:00.668"
	                                    Storyboard.TargetProperty="Angle">
	                                        <SplineDoubleKeyFrame KeyTime="0" Value="-134" KeySpline="0.13,0.21,0.1,0.7"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:0.433" Value="-14" KeySpline="0.02,0.33,0.38,0.77"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:1.2" Value="69"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:1.617" Value="181" KeySpline="0.57,0.17,0.95,0.75"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:2.017" Value="331" KeySpline="0,0.19,0.07,0.72"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:2.783" Value="415"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:3.217" Value="561" KeySpline="0,0,0.95,0.37"/>
	                                    </DoubleAnimationUsingKeyFrames>
	                                    <DoubleAnimationUsingKeyFrames
	                                    Storyboard.TargetName="E6R"
	                                    BeginTime="00:00:00.835"
	                                    Storyboard.TargetProperty="Angle">
	                                        <SplineDoubleKeyFrame KeyTime="0" Value="-140" KeySpline="0.13,0.21,0.1,0.7"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:0.433" Value="-20" KeySpline="0.02,0.33,0.38,0.77"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:1.2" Value="63"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:1.617" Value="175" KeySpline="0.57,0.17,0.95,0.75"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:2.017" Value="325" KeySpline="0,0.19,0.07,0.72"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:2.783" Value="409"/>
	                                        <SplineDoubleKeyFrame KeyTime="0:0:3.217" Value="555" KeySpline="0,0,0.95,0.37"/>
	                                    </DoubleAnimationUsingKeyFrames>
	                                </Storyboard>
	                            </VisualState>
	                        </VisualStateGroup>
	                    </VisualStateManager.VisualStateGroups>
	                    <Grid x:Name="Ring"
	                        Margin="{TemplateBinding Padding}"
	                        MaxWidth="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.MaxSideLength}"
	                        MaxHeight="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.MaxSideLength}"
	                        Visibility="Collapsed"
	                        RenderTransformOrigin=".5,.5"
	                        FlowDirection="LeftToRight">
	                        <Canvas RenderTransformOrigin=".5,.5">
	                            <Canvas.RenderTransform>
	                                <RotateTransform x:Name="E1R" />
	                            </Canvas.RenderTransform>
	                            <Ellipse
	                            x:Name="E1"
	                            Style="{StaticResource ProgressRingEllipseStyle}"
	                            Width="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseDiameter}"
	                            Height="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseDiameter}"
	                            Margin="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseOffset}"
	                            Fill="{TemplateBinding Foreground}"/>
	                        </Canvas>
	                        <Canvas RenderTransformOrigin=".5,.5">
	                            <Canvas.RenderTransform>
	                                <RotateTransform x:Name="E2R" />
	                            </Canvas.RenderTransform>
	                            <Ellipse
	                            x:Name="E2"
	                            Style="{StaticResource ProgressRingEllipseStyle}"
	                            Width="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseDiameter}"
	                            Height="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseDiameter}"
	                            Margin="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseOffset}"
	                            Fill="{TemplateBinding Foreground}"/>
	                        </Canvas>
	                        <Canvas RenderTransformOrigin=".5,.5">
	                            <Canvas.RenderTransform>
	                                <RotateTransform x:Name="E3R" />
	                            </Canvas.RenderTransform>
	                            <Ellipse
	                            x:Name="E3"
	                            Style="{StaticResource ProgressRingEllipseStyle}"
	                            Width="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseDiameter}"
	                            Height="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseDiameter}"
	                            Margin="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseOffset}"
	                            Fill="{TemplateBinding Foreground}"/>
	                        </Canvas>
	                        <Canvas RenderTransformOrigin=".5,.5">
	                            <Canvas.RenderTransform>
	                                <RotateTransform x:Name="E4R" />
	                            </Canvas.RenderTransform>
	                            <Ellipse
	                            x:Name="E4"
	                            Style="{StaticResource ProgressRingEllipseStyle}"
	                            Width="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseDiameter}"
	                            Height="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseDiameter}"
	                            Margin="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseOffset}"
	                            Fill="{TemplateBinding Foreground}"/>
	                        </Canvas>
	                        <Canvas RenderTransformOrigin=".5,.5">
	                            <Canvas.RenderTransform>
	                                <RotateTransform x:Name="E5R" />
	                            </Canvas.RenderTransform>
	                            <Ellipse
	                            x:Name="E5"
	                            Style="{StaticResource ProgressRingEllipseStyle}"
	                            Width="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseDiameter}"
	                            Height="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseDiameter}"
	                            Margin="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseOffset}"
	                            Fill="{TemplateBinding Foreground}"/>
	                        </Canvas>
	                        <Canvas RenderTransformOrigin=".5,.5"
	                            Visibility="Collapsed"
	                            x:Name="SixthCircle">
	                            <Canvas.RenderTransform>
	                                <RotateTransform x:Name="E6R" />
	                            </Canvas.RenderTransform>
	                            <Ellipse
	                            x:Name="E6"
	                            Style="{StaticResource ProgressRingEllipseStyle}"
	                            Width="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseDiameter}"
	                            Height="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseDiameter}"
	                            Margin="{Binding RelativeSource={RelativeSource TemplatedParent}, Path=TemplateSettings.EllipseOffset}"
	                            Fill="{TemplateBinding Foreground}"/>
	                        </Canvas>
	                    </Grid>
	                </Border>
	            </ControlTemplate>
	        </Setter.Value>
	    </Setter>
	</Style>

[ProgressRing]: http://msdn.microsoft.com/en-us/library/windows/apps/windows.ui.xaml.controls.progressring
[ProgressBar]: http://msdn.microsoft.com/en-us/library/system.windows.controls.progressbar%28v=vs.95%29.aspx
[PerformanceProgressBar]: http://www.jeff.wilcox.name/2010/08/performanceprogressbar/
[wonky]: /xaml-visualstatemanager-note
