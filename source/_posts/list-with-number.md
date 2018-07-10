---
title: WPF 给列表加上自动编号
date: 2017-08-24 17:56:10
categories: WPF
tags: 
    - XAML
    - ListBox
    - ListView
description: 介绍了一种使用 AlternationCount 和 AlternationIndex 属性进行列表自动编号的方法。
toc: true
---
# 前言
给 WPF 列表添加自动编号是个非常头疼的问题。综合网上的解决方案，有几种：
- 使用数据绑定。即给数据对象加入一个表示编号的属性，利用数据绑定显示编号。此方法实现方便，但是难以自动更新。
- 使用代码。在网上看到了这样一种解决方法，文章很多，比如[这里](http://www.cnblogs.com/CSharpSPF/archive/2012/02/29/2373287.html)，看起来比较麻烦，我们目标是寻找一种纯 XAML 的解决方案。
- 还有一个使用 VB.Net 写的代码示例（[WPF中给listboxItem加上序号标签](http://download.csdn.net/download/u013305271/6874881)），声称可以。但是我不太了解 VB，不清除具体情况。
- 还有一个方案看起来是非常简单的，也是我所采用的解决方案。最早没有成功实现，后来参考了《WPF 编程宝典》才成功实现。但是现在一时半会儿找不到了。

<!-- more -->

# 代码实现
《WPF 编程宝典》中介绍了“条纹列表”样式的实现。样子大概如下：

{% asset_img 条纹列表.png 条纹列表样式 %}
## 条纹列表
实现条纹列表，需要用到 ListBox/ListView 控件的 `AlternationCount` 属性。MSDN对该属性的解释如下：
> 获取或设置 ItemsControl 中的交替项容器的数目，该控件可使交替容器具有唯一外观。

交替项容器的效果，就是对每一个 ListBoxItem/ListViewItem 有一个 `ItemsControl.AlternationIndex` 属性，表示该项的交替位置。例如，如果对一个 ListBox/ListView 控件设置 `AlternationCount` 属性为 3，则每一项的索引号和交替项索引号如下：

|列表索引|交替项索引|
|---|--|
|0|0|
|1|1|
|2|2|
|3|0|
|4|1|
|5|2|
|6|0|
|7|1|

于是，根据交替项索引，就可以实现条纹列表。在数据模板中，使用 `RelativeSource` 找到当前数据所在的 `ListBoxItem`，使用 `(ItemsControl.AlternationIndex)` 属性获取其交替项索引，对不同的索引进行处理。或者在生成的项的样式模板中，使用触发器修改条目样式，原书代码如下：
{% asset_img 条纹列表代码.png 条纹列表的代码 %}

## 编号列表
将上述方法进行推广，即可得到编号列表。我们可以将 `AlternationCount` 属性设置为列表项目的总数，这样在 `ItemsControl.AlternationIndex` 属性中，就可以获取当前列表项在列表中的位置。

### 数据绑定
在数据模板中，将一个 `TextBlock` 的 `Text` 属性绑定到该属性中，即可进行显示。
``` xml
<TextBlock Text="{Binding RelativeSource={RelativeSource Mode=FindAncestor, AncestorType=ListBoxItem}, 
                          Path=(ItemsControl.AlternationIndex)" />

```

### 转换器
当然，该位置是从 0 开始的，一般我们习惯于从 1 开始。因此，需要一个转换器。既然使用了转换器，就可以自定义很多表示方法，如“周日”、“周一”……“周六”；“第1名”、“第2名”……

例如如下设计的一个转换器：
``` cs
class IntToLevelStringConverter : IValueConverter
{
    static string[] LevelString = { "第一级", "第二级", "第三级", "第四级", "第五级" };

    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
        int levelRank = (int)value;
        if (levelRank < LevelString.Length)
        {
            return LevelString[levelRank];
        }
        else
        {
            return null;
        }
    }

    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
    {
        throw new NotImplementedException();
    }
}
```
### 使用图片指示等级
或者使用图片指示等级，如
![](/WPF/list-with-number/1ji.png)
![](/WPF/list-with-number/2ji.png)
![](/WPF/list-with-number/3ji.png)

那么，XAML 代码中应创建一个 `Image` 控件，将其 `Source` 属性绑定到 `ItemsControl.AlternationIndex` 属性，利用转换器转换成对应图片的 URL。

XAML的代码：
``` xml
<Image Source="{Binding RelativeSource={RelativeSource Mode=FindAncestor, AncestorType=ListBoxItem}, 
                        Path=(ItemsControl.AlternationIndex), 
                        Converter={StaticResource IntToIconStringConverter}}"/>

```
转换器的代码：
``` cs
class IntToIconStringConverter : IValueConverter
{
    static string[] IconPath =
    {
        "pack://application:,,,/FGISClient.UIControls;component/Icon/FireHandle/1ji.png",
        "pack://application:,,,/FGISClient.UIControls;component/Icon/FireHandle/2ji.png",
        "pack://application:,,,/FGISClient.UIControls;component/Icon/FireHandle/3ji.png"
    };

    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
        int levelRank = (int)value;
        if (levelRank < IconPath.Length)
        {
            return IconPath[levelRank];
        }
        else
        {
            return null;
        }

    }

    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
    {
        throw new NotImplementedException();
    }
}
```