---
title: WPF 使用消息插销（Plug）机制在多个组件之间传递消息
date: 2018-03-20 16:04:57
categories: WPF
tags:
    - WPF
    - cs
---
WPF 这个框架，有一些比较令人头疼的问题，组件之间的消息传递就是其中的一个。
《WPF 编程宝典》中提到了“命令”的方法，具有一些优越性。
除此之外，笔者在实践中设计了一种 Plug 机制，实现组件间的通讯。
<!-- more -->

# 组件间通讯

往往一些组件之间需要互相通讯，但是互相通讯的组件不一定互相可见。
比如在一个页面上，有一个列表 List 和一个地图 Map，列表与地图要实现联动。
由于在逻辑上，列表时主窗口的组件，地图也是主窗口的组件，因此他们互不可见。
（List 无法直接调用 Map 的方法或订阅 Map 的事件，Map 也是如此）。
这种问题会非常常见。

既然是通讯，就一定有发送方（Sender）和接收方（Receiver）；
既然不可见，就一定要有中间件（MiddleWare），来传递消息。
对于中间件，其必须被二者可见，或可见二者
（比如 MiddleWare 可见 Sender，Receiver 可见 MiddleWare，
即 MiddleWare 可以调用 Sender 的方法或订阅 Sender 的事件，
MiddleWare 可以调用 Receiver 的方法或订阅 Receiver 的事件）。

《WPF 编程宝典》中给出的“命令模型”，用在解决这个问题时，属于
“MiddleWare 可见二者”。本文所提出的消息插销机制，属于
“MiddleWare 被二者可见”。

# 命令模型

命令模型用于解决这个问题的过程是：需要发送方抛出命令，中间件接收命令，
并指挥接收方执行相应操作。

WPF 中定义了 `ICommand` 接口，来描述命令。所有的命令都继承自该接口。
但是一般使用实现了该接口的两个类 `RoutedCommand` 和 `RoutedUICommand`。
这两个类都实现了命令事件的冒泡，功能也差不多，只是 `RoutedUICommand`
包含了一个字符串。

发送命令之前，需要有一个已经实例化的命令对象，可以直接使用 `RoutedCommand`。
比如定义一个静态类，里面包含所有需要用到的命令对象。

``` cs
public class FireHandleCommans
{
    static RoutedCommand _showPinMarkersCommand = new RoutedCommand();

    /// <summary>
    /// 显示大头针命令
    /// </summary>
    static public RoutedCommand ShowPinMarkersCommand
    {
        get => _showPinMarkersCommand;
        set => _showPinMarkersCommand = value;
    }
}
```

这里定义好了之后，在需要抛出命令的地方使用 `Execute()` 方法：

``` cs
UIControlCommands.ShowPinMarkersCommand.Execute(
    new Tuple<IEnumerable<IDataObjectBaseViewModel>, IEnumerable<IDataObjectBaseViewModel>>(
        ItemData,
        DataItemListView.ItemsSource.Cast<IDataObjectBaseViewModel>()), Application.Current.MainWindow);
```

在主窗口中（或发送方与接收方共同的父控件）接收此命令：

``` xml
<Window.CommandBindings>
    <CommandBinding Command="{x:Static UICommands:UIControlCommands.ShowPinMarkersCommand}" Executed="ListPageChangedCommandBinding_Executed" />
</Window.CommandBindings>
```

然后在 `ListPageChangedCommandBinding_Executed()` 事件响应函数中
指挥接收方进行操作：

``` cs
private void ListPageChangedCommandBinding_Executed(object sender, ExecutedRoutedEventArgs e)
{
    if (e.Parameter is IEnumerable<IDataObjectBaseViewModel>)
    {
        List<IDataObjectBaseViewModel> listItems = new List<IDataObjectBaseViewModel>(e.Parameter as IEnumerable<IDataObjectBaseViewModel>);
        List<PointLatLng> listPoints = new List<PointLatLng>();
        // 在地图上添加大头针
        listItems.ForEach((item) =>
        {
            listPoints.Add(new PointLatLng(item.Data.GPS_Y, item.Data.GPS_X));
        });
        mapView.AddPinMarker(listPoints);
    }
}
```

整个过程就是这样。

这个过程有个问题：

- 命令的参数没有显式指明类型。编程时往往会出错；
- 依赖公有父控件进行调度，但是有时候这本不是父控件的本职工作。

但是这种方法也有一些好处，由于命令是冒泡的，可以在不同的 UI 层级上做不同的操作。

> 理论上是这样。但是我从来没有成功过。

# 消息插销机制

消息插销机制模仿了 WPF 命令模型的设计，设计了一个 `IPlug` 的接口，
和 `PlugReceiveMessageDelegate` 的委托。
`IPlug` 接口包含一个发送方方法，和一个接收方事件。
这个委托和接口中的方法采用相同的参数列表。具体实现如下：

``` cs
/// <summary>
/// 插销接收方委托
/// </summary>
/// <typeparam name="PlugArgs"></typeparam>
/// <param name="sender"></param>
/// <param name="args"></param>
public delegate void PlugReceiveMessageDelegate<PlugArgs>(object sender, PlugArgs args);

interface IPlug<PlugArgs>
{
    /// <summary>
    /// 调用方方法
    /// </summary>
    /// <param name="sender"></param>
    /// <param name="args"></param>
    void PlugSendMessage(object sender, PlugArgs args);

    /// <summary>
    /// 接收方事件
    /// </summary>
    event PlugReceiveMessageDelegate<PlugArgs> PlugReceiveMessageEvent;
}
```

我们当然是不可以直接使用这个接口的，需要定义类实现此接口，比如：

``` cs
public class ResizePinMarkerEventArgs
{
    int _pinMarkerIndex;

    public ResizePinMarkerEventArgs(int index)
    {
        PinMarkerIndex = index;
    }

    public int PinMarkerIndex { get => _pinMarkerIndex; set => _pinMarkerIndex = value; }
}

public class EnlargePinMarkerPlug : IPlug<ResizePinMarkerEventArgs>
{
    /// <summary>
    /// 大头针变大接收方事件
    /// </summary>
    public event PlugReceiveMessageDelegate<ResizePinMarkerEventArgs> PlugReceiveMessageEvent;

    /// <summary>
    /// 大头针变大调用方方法
    /// </summary>
    /// <param name="index"></param>
    public void PlugSendMessage(object sender, ResizePinMarkerEventArgs e)
    {
        Console.WriteLine("Mouse Enter: " + e.PinMarkerIndex);
        PlugReceiveMessageEvent?.Invoke(sender, e);
    }
}
```

然后需要实例化此类，在一个静态类中加入静态属性：

``` cs
static public class Bolt
{
    /// <summary>
    /// 大头针变大插销
    /// </summary>
    public static EnlargePinMarkerPlug EnlargePinMarker { get; internal set; }

    static Bolt()
    {
        EnlargePinMarker = new EnlargePinMarkerPlug();
    }
}
```

发送方在要发送消息的地方调用消息插销的发送方方法：

``` cs
/// <summary>
/// 鼠标移出列表项
/// </summary>
/// <param name="sender"></param>
/// <param name="e"></param>
private void ListBoxItem_MouseLeave(object sender, MouseEventArgs e)
{
    if (e.OriginalSource is ListBoxItem)
    {
        ListBoxItem item = e.OriginalSource as ListBoxItem;
        Plug.Bolt.MinifyPinMarker.PlugSendMessage(item, new Plug.EventArgs.ResizePinMarkerEventArgs((item.DataContext as IDataObjectBaseViewModel).Indicator));
    }
}
```

接收方订阅该插销的事件，即可接收消息了。

这种方法解决了命令模型的几个不方便的地方，但是同样也失去了命令模型的优势。
如果要在不同地方相应同一个消息插销，那就只能在不同地方分别订阅接收方事件了。