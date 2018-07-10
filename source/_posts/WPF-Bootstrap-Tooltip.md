---
title: WPF制作带居中三角形指示的Tooltip样式的Tooltip
date: 2017-08-21 22:19:30
tags: 
    - WPF
    - XAML
categories: WPF
mathjax: true
description: 本文介绍了使用WPF制作带居中三角形指示的Tooltip样式的Tooltip。
---
本文所要实现的目标样式如下图。

{% asset_img 手工标注.jpg 目标样式 %}

可以看到这个Tooltip能够分解为一个**三角形**和一个**圆角矩形**，而且三角形要居中显示。经过在网上的充分搜索，没有找到可以直接使用的解决方案，那么就自己动手设计一个。
<!-- more -->
# 布局框架
既然要居中显示一个三角形，最方便的应该就是Gird布局了。因此使用XAML创建一个Grid布局。
``` xml
<Grid x:Name="g">
</Grid>
```
由于三角形和圆角矩形在某种程度上说是结合在一起的，因此无需设置行和列。所以这里使用一个`Border`来布局也是可以的。

# 内部布局
我们希望使用三角形“盖住”一部分圆角矩形的边框，因此需要将三角形放置在圆角矩形的下方，才能实现遮盖的效果。
除此之外还有以下要求：
- 采用`Canvas`面板来绘制三角形，此面板需要水平居中对齐、垂直顶部对齐；假设三角形高为6，宽为12，为等腰三角形。
- 采用`Border`控件实现圆角矩形，需要有一定边框宽度和颜色，水平拉伸、垂直拉伸，且与`Grid`面板的上边缘有一定的边距，边距大小略小于三角形高，这样可以让三角形遮盖一段边框。

因此采用如下设计：
``` xml
<Grid x:Name="g">
    <Border CornerRadius="3" BorderThickness="1" 
            BorderBrush="{StaticResource TooltipBorderBrush}" 
            HorizontalAlignment="Stretch" 
            VerticalAlignment="Stretch" 
            Background="White" 
            Margin="0,5,0,0" Padding="8">
    </Border>
    <Canvas HorizontalAlignment="Center" 
            VerticalAlignment="Top" 
            Height="6" Width="12">
    </Canvas>
</Grid>
```
# 三角形绘制
在一个`Canvas`面板中，使用`Polygon`绘制三角形，设置为白色。那么这个三角形三个点的坐标当然就是$(0,6)$、$(6,0)$、$(12,6)$了。这样就能绘制一个三角形，又遮盖一段圆角矩形的边框。
``` xml
<Polygon Points="0,6 6,0 12,6" StrokeThickness="0" Fill="White"/>
```
三角形的边框不能直接使用属性进行设置了，否则三角形的底也会被绘制上边框，无法达到效果。我们可以绘制一段多段线（`Polyline`）来实现边框的绘制。同样还是设置上面三个点，但是对象类型改为`Polyline`。
``` xml
<Polyline Points="0,6 6,0 12,6" 
          Stroke="{StaticResource TooltipBorderBrush}" 
          StrokeThickness="1"/>
```
此时三角形绘制完成。

> **注意**：需要明确指定`Canvas`面板的宽度，此处为12。否则，三角形无法真正居中，而是最左边对齐到圆角矩形的中间。

# 圆角矩形的设置
圆角矩形样式较好设置。内容的设置可以使用`ContentPresenter`，使得在使用时可以直接使用XAML代码来设置内容。该Presenter设置为水平居中、垂直居中即可。
``` xml
<ContentPresenter VerticalAlignment="Center" HorizontalAlignment="Center"/>
```

# 整体代码
``` xml
<Grid x:Name="g">
    <Border CornerRadius="3" 
            BorderThickness="1" BorderBrush="{StaticResource TooltipBorderBrush}" 
            HorizontalAlignment="Stretch" VerticalAlignment="Stretch" 
            Background="White" Margin="0,5,0,0" Padding="8">
        <ContentPresenter VerticalAlignment="Center" HorizontalAlignment="Center"/>
    </Border>
    <Canvas HorizontalAlignment="Center" VerticalAlignment="Top" Height="6" Width="12">
        <Polygon Points="0,6 6,0 12,6" StrokeThickness="0" Fill="White"/>
        <Polyline Points="0,6 6,0 12,6" 
                  Stroke="{StaticResource TooltipBorderBrush}" 
                  StrokeThickness="1"/>
    </Canvas>
</Grid>
```
可以将此段代码放置在自定义Tooltip样式的`Template`属性值下，实现通过样式进行设置。即
``` xml
<Style x:Key="FGisToolTipStyle" TargetType="ToolTip">
    <Setter Property="OverridesDefaultStyle" Value="true" />
    <Setter Property="HasDropShadow" Value="True" />
    <Setter Property="Foreground" Value="#333333" />
    <Setter Property="FontSize" Value="14" />
    <Setter Property="Placement" Value="Bottom"/>
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="ToolTip">
                <Grid x:Name="g">
                    <Border CornerRadius="3" 
                            BorderThickness="1" 
                            BorderBrush="{StaticResource TooltipBorderBrush}" 
                            HorizontalAlignment="Stretch" 
                            VerticalAlignment="Stretch" 
                            Background="White" 
                            Margin="0,5,0,0" Padding="8">
                        <ContentPresenter VerticalAlignment="Center" 
                                          HorizontalAlignment="Center"/>
                    </Border>
                    <Canvas HorizontalAlignment="Center" 
                            VerticalAlignment="Top" 
                            Height="6" Width="12">
                        <Polygon Points="0,6 6,0 12,6" StrokeThickness="0" Fill="White"/>
                        <Polyline Points="0,6 6,0 12,6" 
                                  Stroke="{StaticResource TooltipBorderBrush}" 
                                  StrokeThickness="1"/>
                    </Canvas>
                </Grid>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```
也可以放置在自定义的用户控件中，作为控件使用。