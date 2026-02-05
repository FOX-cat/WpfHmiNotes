# C# WPF上位机学习笔记

### 串口通讯（关键）

WPF 里没有像 WinForms 工具箱那样可拖拽的 **SerialPort 组件**，但不代表不能用串口——在 WPF 里通常就是**直接用代码引用 `System.IO.Ports.SerialPort`**（或者用更现代/更稳定的第三方库），然后把串口事件和 UI 用 MVVM/Dispatcher 方式串起来。

下面给你一套“WPF 正确打开方式”，照着做基本就能跑。

------

1) **直接用 `System.IO.Ports.SerialPort`（最常见做法）**

✅ 安装/引用

- .NET Framework 的 WPF：一般直接可用。
- .NET (Core/5/6/7/8) 的 WPF：如果项目里找不到 `System.IO.Ports`，就在 **NuGet** 安装：
  - `System.IO.Ports`

✅ 最小可用示例（推荐先跑通）

> 注意：`DataReceived` 不在 UI 线程，更新界面要用 `Dispatcher`.

```c#
using System;
using System.IO.Ports;
using System.Text;
using System.Windows;

public partial class MainWindow : Window
{
    private SerialPort _sp;

    public MainWindow()
    {
        InitializeComponent();
        InitSerial();
    }

    private void InitSerial()
    {
        _sp = new SerialPort("COM3", 115200, Parity.None, 8, StopBits.One);
        _sp.Encoding = Encoding.ASCII;          // 视协议而定
        _sp.DataReceived += Sp_DataReceived;
        _sp.NewLine = "\r\n";                   // 若你用 ReadLine()
        _sp.Open();
    }

    private void Sp_DataReceived(object sender, SerialDataReceivedEventArgs e)
    {
        try
        {
            string data = _sp.ReadExisting();   // 或者 ReadLine / Read(buffer,...)
            Dispatcher.Invoke(() =>
            {
                // 假设你界面有个 TextBox 叫 tbLog
                tbLog.AppendText(data);
                tbLog.ScrollToEnd();
            });
        }
        catch (Exception ex)
        {
            Dispatcher.Invoke(() => tbLog.AppendText($"[ERR]{ex.Message}\n"));
        }
    }

    protected override void OnClosed(EventArgs e)
    {
        base.OnClosed(e);
        if (_sp != null)
        {
            if (_sp.IsOpen) _sp.Close();
            _sp.Dispose();
        }
    }
}
```

------

2) **WPF 更“正宗”的写法：后台任务 + MVVM（更稳）**

很多人踩坑是因为：

- `DataReceived` 触发频繁、线程切换杂乱
- 粘包/拆包、半包导致解析混乱
- 串口断开重连不好做

更推荐：**开一个读线程/Task，持续读取 `BaseStream.ReadAsync`，自己做缓冲和协议解析**，UI 只订阅“新消息事件/ObservableCollection”。

如果你愿意我可以按你的协议（比如以 0xAA55 开头、长度字段、CRC 等）给你一套完整的“缓冲区 + 拆包器”。

- [ ] **MVVM是什么？**

---

### 界面布局

WPF提供了多种布局面板(Panel)，每种都有其特定的布局行为：

|  布局面板   |         **主要特点**         |   **适用场景**   |
| :---------: | :--------------------------: | :--------------: |
|    Grid     | 行列网格布局，支持单元格合并 |  表单、复杂界面  |
| StackPanel  |        单行或单列堆叠        | 简单列表、工具栏 |
|  DockPanel  |         边缘停靠布局         |   窗口框架布局   |
|  WrapPanel  |         自动换行布局         |   标签云、图库   |
|   Canvas    |         绝对坐标定位         |  绘图、游戏界面  |
| UniformGrid |         均匀分布网格         |    棋盘类布局    |

**Grid布局详解**

- **基本行列定义**

Grid是最强大、最常用的布局容器，通过行(Row)和列(Column)定义网格结构：

```xaml
<Grid>
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto"/>  <!-- 自动高度 -->
        <RowDefinition Height="*"/>     <!-- 剩余空间1份 -->
        <RowDefinition Height="2*"/>    <!-- 剩余空间2份 -->
    </Grid.RowDefinitions>
    <Grid.ColumnDefinitions>
        <ColumnDefinition Width="Auto"/>
        <ColumnDefinition Width="*"/>
    </Grid.ColumnDefinitions>
</Grid>
```

高度/宽度支持四种单位：

​	Auto：根据内容自动调整

​	`*`：按比例分配剩余空间

​	固定值：如"100"（设备无关单位）

​	`*`前可加数字表示权重，如"`2*`"

- **单元格定位与跨行跨列**

```xaml
<Button Grid.Row="0" Grid.Column="0" Content="按钮1"/>
<Button Grid.Row="0" Grid.Column="1" Content="按钮2"/>
<Button Grid.Row="1" Grid.Column="0" Grid.ColumnSpan="2" Content="跨两列"/>
```

高级技巧：

​	使用`GridSplitter`实现可调整大小的分隔条

​	通过`SharedSizeGroup`实现多个Grid的列同步

​	使用`Grid.IsSharedSizeScope`启用共享尺寸

原文链接：**https://blog.csdn.net/qq_46062107/article/details/148079307**

---

### 报错

- 调用线程无法访问此对象，因为另一个线程拥有该对象

**WPF 的 UI 控件只能在创建它的 UI 线程里访问**
 **后台线程 / 串口线程 / Task 线程 → 必须用 Dispatcher 回 UI 线程**

你现在的代码大概率是这样👇（典型错误）

```
Task.Run(() =>
{
    textBox.Text = "刷新内容"; // ❌ 跨线程访问 UI
});
```

或者串口接收线程里：

```
_serialPort.DataReceived += (s, e) =>
{
    textBox.Text = "收到数据"; // ❌
};
```

**🔴 原因**

- WPF 的 `TextBox` 属于 **DispatcherObject**
- 它只能被 **创建它的线程（UI 线程）** 操作
- 其他线程直接访问 → **立刻抛异常**

**用 `Dispatcher` 切回 UI 线程**

```
Dispatcher.Invoke(() =>
{
    textBox.Text = "刷新内容";
});
```

或（推荐，不阻塞）

```
Dispatcher.BeginInvoke(() =>
{
    textBox.Text = "刷新内容";
});
```

---

### WPF中定时器的写法

System.Timer

```c#
//private readonly DispatcherTimer _dtimer = new DispatcherTimer();//感觉不是很好用
private System.Timers.Timer _timer;

_timer = new System.Timers.Timer(1000);    // 1000ms 周期
_timer.Elapsed += async (sender, args) => SendOnce(); //async有问题 除非将SendOnce改成async的函数
//_hbTimer.Elapsed += async (sender2, args2) => await HeartbeatTask();
_timer.AutoReset = true;
_timer.Start();

private void SendOnce()
{
    //定时器执行函数
}
private async Task HeartbeatTask()
{
    if (!plcConnected || _isHeartbeating) return;  // 上一次还在跑，就跳过
    _isHeartbeating = true;

    try
    {
        // 把读写都放到后台线程
        var result = await Task.Run(() =>
        {
            ushort beat = 0;
            ushort beat_send = 0;
            try
            {
                beat = (ushort)plc.Read("DB29.DBW0");
                plc.Write("DB29.DBW2", count_heartbeat);
                beat_send = (ushort)plc.Read("DB29.DBW2");

            }
            catch (Exception ex)
            {
                Console.WriteLine($"PLC Heartbeat read/write failed in Task.Run: {ex.Message}");
                plcConnected = false;
                thingerLED54.LedState = 0;

                this.Invoke((Action)(() =>
                {
                    textBox191.Text = "N/A";
                    textBox192.Text = "N/A";
                    textBox206.Text = "N/A";
                    textBox205.Text = "N/A";

                    textBox112.Text = "N/A";
                    textBox111.Text = "N/A";
                }));
            }
            return (beat, beat_send);
        });

        // UI 更新回到主线程
        this.Invoke((Action)(() =>
        {
            if (plcConnected)
            {
                textBox111.Text = result.beat.ToString();
                textBox112.Text = result.beat_send.ToString();
            }
            else
            {
                textBox111.Text = "N/A";
                textBox112.Text = "N/A";
            }
        }));

        // 计数器
        count_heartbeat = (Int16)((count_heartbeat == 1000) ? 0 : count_heartbeat + 1);
    }
    catch (Exception)
    {
        // 这里只记录或忽略心跳失败，别弹窗！
    }
    finally
    {
        _isHeartbeating = false;
    }
}
```



问ai之后的结果：

**推荐的安全写法（工业上位机常用）**

✅ 推荐版本 1：防止重入（强烈推荐）

```c#
private System.Timers.Timer _timer;
private bool _isRunning = false;

private void InitTimer()
{
    _timer = new System.Timers.Timer(1000);
    _timer.AutoReset = true;
    _timer.Elapsed += Timer_Elapsed;
    _timer.Start();
}

private async void Timer_Elapsed(object sender, System.Timers.ElapsedEventArgs e)
{
    if (_isRunning) return;

    try
    {
        _isRunning = true;
        await SendOnceAsync();
    }
    catch (Exception ex)
    {
        // 记录日志
    }
    finally
    {
        _isRunning = false;
    }
}

private Task SendOnceAsync()
{
    // 定时器执行函数
    return Task.CompletedTask;
}
```

**好处：**

- 防止并发
- 支持异步
- 稳定运行

------

**如果你需要在 `SendOnce` 里操作 UI**

✅ 正确方式（Dispatcher）

```
Application.Current.Dispatcher.Invoke(() =>
{
    TextBox1.Text = "定时刷新";
});
```

或者异步：

```
await Application.Current.Dispatcher.InvokeAsync(() =>
{
    TextBox1.Text = "定时刷新";
});
```

------

**如果只是 UI 定时刷新（更推荐）**

如果你**只在 UI 线程用**，其实更推荐：

```
DispatcherTimer _timer = new DispatcherTimer();
_timer.Interval = TimeSpan.FromSeconds(1);
_timer.Tick += (s, e) => SendOnce();
_timer.Start();
```

| 类型                  | 线程   | 适合场景       |
| --------------------- | ------ | -------------- |
| `DispatcherTimer`     | UI线程 | UI刷新         |
| `System.Timers.Timer` | 线程池 | 通信、后台任务 |

**`DispatcherTimer` 天生不准，只要 UI 线程忙，Tick 就会被延后**

**工业上位机推荐的正确方案（重点）**

✅ 方案 1：后台高精度定时 + UI Dispatcher（强烈推荐）

**这是 PLC / 上位机 / 工控最常见方案**

```C#
private System.Timers.Timer _timer;

private void InitTimer()
{
    _timer = new System.Timers.Timer(1000);
    _timer.AutoReset = true;
    _timer.Elapsed += Timer_Elapsed;
    _timer.Start();
}

private async void Timer_Elapsed(object sender, ElapsedEventArgs e)
{
    // 后台线程，准
    await SendOnceAsync();

    // 需要更新 UI 时再切回 UI 线程
    Application.Current.Dispatcher.BeginInvoke(() =>
    {
        TextBlock1.Text = DateTime.Now.ToString("HH:mm:ss.fff");
    });
}
```

✔ 后台计时
 ✔ UI 不影响定时精度
 ✔ 工程稳定

**✅ 方案 2：严格 1 秒对齐（不是“执行完再等”）**

如果你要求的是：

> **每个整秒点触发（如 12:00:01.000）**

那普通 Timer 都不够。

**专业写法（时间补偿）**

```C#
async Task StartPreciseTimer()
{
    var sw = Stopwatch.StartNew();
    long nextTick = 1000;

    while (true)
    {
        var delay = nextTick - sw.ElapsedMilliseconds;
        if (delay > 0)
            await Task.Delay((int)delay);

        SendOnce();

        nextTick += 1000;
    }
}
```

**优点：**

- 自动补偿延迟
- 长时间运行不会漂移

---

### 样式Style写法

![WPF样式与触发器 - 图1](https://geekdaxue.co/uploads/projects/filnado@dotnet/e808577d845084d75668ef233b9f4e0f.png)

```xaml
<Window.Resources>
    <Style x:Key="TextBlockStyle" TargetType="{x:Type TextBlock}">
        <Setter Property="FontFamily" Value="宋体"/>
        <Setter Property="FontSize" Value="32"/>
        <Setter Property="Foreground" Value="Red"/>
        <Setter Property="FontWeight" Value="Bold"/>
    </Style>
</Window.Resources>
<Grid>
    <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center">
        <TextBlock Text="Text1" Foreground="Green" Style="{StaticResource TextBlockStyle}"/>
        <TextBlock Text="Text2" Style="{StaticResource TextBlockStyle}"/>
        <TextBlock Text="Text3" Style="{StaticResource TextBlockStyle}"/>
        <TextBlock Text="Text4" Style="{StaticResource TextBlockStyle}"/>
    </StackPanel>
</Grid>
```

等效于

```xaml
<StackPanel HorizontalAlignment="Center" VerticalAlignment="Center">
    <TextBlock Text="Text1" FontFamily="宋体" FontSize="18" Foreground="Red" FontWeight="Bold"/>
    <TextBlock Text="Text1" FontFamily="宋体" FontSize="18" Foreground="Red" FontWeight="Bold"/>
    <TextBlock Text="Text1" FontFamily="宋体" FontSize="18" Foreground="Red" FontWeight="Bold"/>
    <TextBlock Text="Text1" FontFamily="宋体" FontSize="18" Foreground="Red" FontWeight="Bold"/>
</StackPanel>
```

---

### 触发器写法

- Trigger：监测依赖属性的变化、触发器生效。

```xaml
<Trigger Property="IsMouseOver" Value="True">
    <Setter Property="Background" Value="Blue"/>
    <Setter Property="BorderBrush" Value="Red"/>
</Trigger>
```

- MultiTrigger：通过多个条件的设置、达到满足条件、触发器生效

```xaml
<MultiTrigger>
	<MultiTrigger.Conditions>
        <Condition Property="IsMouseOver" Value="true"/>
        <Condition Property="IsFocused" Value="True"/>
    </MultiTrigger.Conditions>
    <MultiTrigger.Setters>
        <Setter Property="Background" Value="red"/>
    </MultiTrigger.Setters>
</MultiTrigger>
```

- DataTrigger：通过数据的变化、触发器生效。
- MultiDataTrigger：多个数据变化的触发器。
- EventTrigger：事件触发器，触发了某类事件时，触发器生效。

```xaml
<EventTrigger RoutedEvent="MouseMove">
    <EventTrigger.Actions>
        <BeginStoryboard>
            <Storyboard>
                <DoubleAnimation Duration="0:0:0.02" Storyboard.TargetProperty="FontSize" To="18"/>
            </Storyboard>
        </BeginStoryboard>
    </EventTrigger.Actions>
</EventTrigger>

<EventTrigger RoutedEvent="MouseLeave">
    <EventTrigger.Actions>
        <BeginStoryboard>
            <Storyboard>
                <DoubleAnimation Duration="0:0:0.02" Storyboard.TargetProperty="FontSize" To="13"/>
            </Storyboard>
        </BeginStoryboard>
    </EventTrigger.Actions>
</EventTrigger>
```



```xaml
<Window.Resources>
    <Style x:Key="TextBlockStyle" TargetType="{x:Type TextBlock}">
        <Setter Property="FontFamily" Value="宋体"/>
        <Setter Property="FontSize" Value="32"/>
        <Setter Property="Foreground" Value="Red"/>
        <Setter Property="FontWeight" Value="Bold"/>
        <Setter Property="HorizontalAlignment" Value="Center"/>
    </Style>
    <Style x:Key="BoderStyle" TargetType="{x:Type Border}">
        <Setter Property="BorderThickness" Value="5"/>
        <Style.Triggers>
            <Trigger Property="IsMouseOver" Value="True">
                <Setter Property="Background" Value="Blue"/>
                <Setter Property="BorderBrush" Value="Red"/>
            </Trigger>

            <Trigger Property="IsMouseOver" Value="False">
                <Setter Property="Background" Value="Red"/>
                <Setter Property="BorderBrush" Value="Blue"/>
            </Trigger>
        </Style.Triggers>
    </Style>
    <Style x:Key="TextBoxStyle" TargetType="{x:Type TextBox}">
        <Setter Property="BorderThickness" Value="1"/>
        <Style.Triggers>
            <MultiTrigger>
                <MultiTrigger.Conditions>
                    <Condition Property="IsMouseOver" Value="true"/>
                    <Condition Property="IsFocused" Value="True"/>
                </MultiTrigger.Conditions>
                <MultiTrigger.Setters>
                    <Setter Property="Background" Value="red"/>
                </MultiTrigger.Setters>
            </MultiTrigger>
        </Style.Triggers>
    </Style>
    <Style x:Key="ButtonStyle" TargetType="Button">
        <Setter Property="BorderThickness" Value="1"/>
        <Style.Triggers>
            <EventTrigger RoutedEvent="MouseMove">
                <EventTrigger.Actions>
                    <BeginStoryboard>
                        <Storyboard>
                            <DoubleAnimation Duration="0:0:0.02" Storyboard.TargetProperty="FontSize" To="18"/>
                        </Storyboard>
                    </BeginStoryboard>
                </EventTrigger.Actions>
            </EventTrigger>

            <EventTrigger RoutedEvent="MouseLeave">
                <EventTrigger.Actions>
                    <BeginStoryboard>
                        <Storyboard>
                            <DoubleAnimation Duration="0:0:0.02" Storyboard.TargetProperty="FontSize" To="13"/>
                        </Storyboard>
                    </BeginStoryboard>
                </EventTrigger.Actions>
            </EventTrigger>
        </Style.Triggers>
    </Style>
</Window.Resources>
<Grid>
    <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center">
        <TextBlock Text="Text1" Foreground="Green" Style="{StaticResource TextBlockStyle}"/>
        <TextBlock Text="Text2" Style="{StaticResource TextBlockStyle}"/>
        <TextBlock Text="Text3" Style="{StaticResource TextBlockStyle}"/>
        <TextBlock Text="Text4" Style="{StaticResource TextBlockStyle}"/>
        <Border Width="100" Height="50" Style="{StaticResource BoderStyle}"/>
        <TextBox Width="100" Height="50" Style="{StaticResource TextBoxStyle}"/>
        <Button Width="100" Height="50" Content="Hello" FontSize="13" Style="{StaticResource ButtonStyle}"/>
    </StackPanel>
</Grid>
```



---

### 绑定

```xaml
<StackPanel VerticalAlignment="Center" HorizontalAlignment="Center">
    <Slider Name="slider" Width="200"/>
    <TextBlock Text="{Binding ElementName=slider,Path=Value}" HorizontalAlignment="Center"/>
</StackPanel>
```

绑定的模式就类似我们商业中的合作，是一次性回报还是持续获益，是否可以单方面终止，是否具有投票权等，在WPF中绑定的模式又分为五种:

- OneWay（单向绑定）：当源属性发生变化更新目标属性，类似上面的例子中，滑动变化更新文本的数据。

```xaml
<StackPanel VerticalAlignment="Center" HorizontalAlignment="Center">
    <Slider Name="slider" Width="200"/>
    <TextBox Width="200" Text="{Binding ElementName=slider,Path=Value,Mode=OneWay}" HorizontalAlignment="Center"/>
</StackPanel>
```

![WPF绑定Binding - 图2](https://geekdaxue.co/uploads/projects/filnado@dotnet/91f26674d60ee3d9386c789ce185e8a0.gif)

- TwoWay（双向绑定）：当源属性发生变化更新目标属性，目标属性发生变化也更新源属性。

```xaml
<StackPanel VerticalAlignment="Center" HorizontalAlignment="Center">
    <Slider Name="slider" Width="200"/>
    <TextBox Width="200" Text="{Binding ElementName=slider,Path=Value,Mode=TwoWay}" HorizontalAlignment="Center"/>
</StackPanel>
```

![WPF绑定Binding - 图3](https://geekdaxue.co/uploads/projects/filnado@dotnet/fcaf9f752e2e054da8b855feec80a769.gif)

- OneTime（单次模式）：根据第一次源属性设置目标属性，在此之后所有改变都无效。
  - 如果第一次绑定数据源为0，那么无论后面如何改变2、3、4……都无法更新到目标属性上。

```xaml
<StackPanel VerticalAlignment="Center" HorizontalAlignment="Center">
    <Slider Name="slider" Width="200"/>
    <TextBox Width="200" Text="{Binding ElementName=slider,Path=Value,Mode=OneTime}" HorizontalAlignment="Center"/>
</StackPanel>
```

![WPF绑定Binding - 图4](https://geekdaxue.co/uploads/projects/filnado@dotnet/333158686a98df5b675f45e35218ecaa.gif)

- OneWayToSource：和OneWay类型相似，只不过整个过程倒置。

```xaml
<StackPanel VerticalAlignment="Center" HorizontalAlignment="Center">
    <Slider Name="slider" Width="200"/>
    <TextBox Width="200" Text="{Binding ElementName=slider,Path=Value,Mode=OneWayToSource}"              HorizontalAlignment="Center"/>
</StackPanel>
```

![WPF绑定Binding - 图5](https://geekdaxue.co/uploads/projects/filnado@dotnet/f31fe246167969002e574ad6175c12a0.gif)

- Default：既可以是双向，也可以是单向，除非明确表明某种模式，否则采用该默认绑定

**绑定到非元素上**

上面的代码中，使用的绑定方式是根据元素的方式：ElementName=xxx，如果需要绑定到一个非元素的对象，则有以下几个属性：

- Source：指向一个数据源，示例：`TextBox`使用绑定的方式用`Source`指向一个静态资源ABC：

```xaml
<Window.Resources>    
    <TextBox x:key="txt1">ABC</TextBox>
</Window.Resources>
<Grid>    
    <TextBox Text="{Binding Source={StaticResource txt1},Path=Text}"}/>
</Grid>
```

- RelativeSource：使用一个名为`RelativeSource`的对象来根据不同的模式查找源对象。

```xaml
<StackPanel Width="200">    
    <StackPanel Width="300"/>    
    <!-- TextBlock 的Text值为200-->   
    <TextBlock Text="{Binding Path=Width, RelativeSource={RelativeSource                      Mode=FindAncestor,AncestorType={x:Type StackPanel}}}"/>
</StackPanel>
```

- DataContext：从当前的元素树向上查找到第一个非空的`DataContext`属性为源对象。

```C#
public partial class MainWindow : Wondow
{    
    public MainWindow()    
    {        
        InitializeComponent();        
        this.DataContext = new Test(){Name = "小明"};    
    }
}
public class Test
{    
    public string Name{get;set;}
}
```

```xaml
<Grid>    
    <!--绑定后台生成的Name-->    
    <TextBlock Text="{Binding Name}"/>
</Grid>
```



### WPF MVVM框架

MVVM 模式由三个组件组成： 

- Model（模型） 负责应用程序的数据和业务逻辑。 它完全不知道 View 和 ViewModel 的存在，它只是一个纯粹的数据和逻辑层，可以被任何部分复用。

- View（视图） 负责应用程序的 UI 部分。

- ViewModel（视图模型） 从 Model 中获取原始数据，并处理、转换为 View 需要的格式。

![img](https://blog.mindtian.cn/wp-content/uploads/2025/10/1762428388-%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE-2025-11-06-192610.png)

#### .NET MVVM 框架对比表

|    框架     |                            Prism                             |                          MVVM Light                          |                        Caliburn.Micro                        |                    CommunityToolkit.Mvvm                     |
| :---------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| GitHub 链接 | [PrismLibrary/Prism](https://github.com/PrismLibrary/Prism)  | [lbugnion/mvvmlight](https://github.com/lbugnion/mvvmlight)  | [Caliburn-Micro/Caliburn.Micro](https://github.com/Caliburn-Micro/Caliburn.Micro) | [CommunityToolkit/dotnet](https://github.com/CommunityToolkit/dotnet) |
|  最新版本   |                        9.0.537 (2025)                        |                         5.4.1 (2018)                         |                        4.0.222 (2024)                        |                         8.3.0 (2025)                         |
|   许可证    |                             MIT                              |                             MIT                              |                             MIT                              |                             MIT                              |
|    Stars    |                            ~6.3k                             |                            ~2.8k                             |                            ~2.7k                             |                     ~2.5k (整个 Toolkit)                     |
|  核心功能   | - 模块化（动态加载模块）- 依赖注入（DryIoc/Unity）- 区域导航（RegionManager）- 事件聚合 | - 轻量级 MVVM- Messenger 消息传递- 简单命令绑定- 基本 DI 支持 |       - 约定优于配置- 自动视图绑定- 事件聚合- 屏幕导航       | - Source Generator（减少样板代码）- ObservableObject/RelayCommand- 轻量消息传递- 跨平台支持 |
|   UI 集成   |    - 强大的 XAML 区域管理- 支持 Material Design/AdonisUI     |             - 简单 XAML 数据绑定- 需手动配置 UI              |         - 自动绑定 View/ViewModel- 适合复杂 UI 导航          |          - 简单绑定，跨 XAML 平台- 集成 WPF UI 控件          |
|    性能     |         - 中等（模块化和 DI 增加开销）- 适合大型项目         |                 - 轻量，启动快- 小型项目高效                 |             - 轻量，约定减少代码- 中小型项目高效             |    - 高性能（Source Generator 优化）- 适合现代 .NET 项目     |
|  学习曲线   |                    中等偏高（模块化复杂）                    |                       低（简单易上手）                       |                      中等（约定需熟悉）                      |                  低（现代化 API，文档清晰）                  |
| 社区活跃度  |                   高（官方维护，频繁更新）                   |                    低（2018 后无大更新）                     |                  中等（社区驱动，更新较慢）                  |                   高（微软支持，活跃开发）                   |
|  适用场景   |   - 大型企业级 WPF 应用- 需要模块化、动态 UI- 复杂导航需求   |         - 小型或原型项目- 简单 MVVM 需求- 初学者友好         |          - 中小型项目- 偏好约定式开发- 快速 UI 原型          |   - 现代 .NET 6/8 项目- 跨平台（WPF/MAUI）- 追求性能和简洁   |
|  依赖注入   |                      内置 DryIoc/Unity                       |                  简单 DI（ServiceLocator）                   |                         内置简单容器                         |                无内置 DI，需外部（如 MS DI）                 |
|  导航支持   |                   强大（RegionNavigation）                   |                       基本（需自定义）                       |                   强大（Screen/Conductor）                   |                        基本（需扩展）                        |
| 文档 / 示例 |                  丰富（Prism-Samples-Wpf）                   |                      中等（官方教程少）                      |                      丰富（社区示例多）                      |                   优秀（微软文档 + 示例）                    |
|   跨平台    |                   WPF 为主，部分支持 MAUI                    |                        WPF / 银光为主                        |                        WPF / 部分 UWP                        |                       WPF/MAUI/WinUI 3                       |

### MvvmLight -> CommunityToolkit.Mvvm

基于MvvmLightLibsStd10版本***已弃用***，需要下载新的CommunityToolkit.Mvvm来替换。

新版本的框架需要在程序中引用如下：

Model：

```c#
using CommunityToolkit.Mvvm.ComponentModel;
```

ViewModel

```c#
using CommunityToolkit.Mvvm.Input;
```

View：保持和之前一致即可



---

# VS更改项目名

1.重命名解决方案
2.重命名项目名
3.修改程序集和命名空间名称
4.全局替换项目名
5.修改项目文件夹名称
6.修改[.sln文件](https://zhida.zhihu.com/search?content_id=230878975&content_type=Article&match_order=1&q=.sln文件&zhida_source=entity)

示例将 OldName 重命名为 NewName

**1.重命名解决方案**

右键解决方案，选择重命名，将 OldName 重命名为 NewName

![img](https://pica.zhimg.com/v2-5e3653eeac2751c6bb9731fc8f5e6fd4_1440w.jpg)

![img](https://pic1.zhimg.com/v2-899489fb6854d7af6639a08bfcc03fc0_1440w.jpg)

**2.重命名项目名**

右键项目，选择重命名，将 OldName 重命名为 NewName

![img](https://picx.zhimg.com/v2-935f26282c7cb89ef4e5e347a7cd1a35_1440w.jpg)

![img](https://pic4.zhimg.com/v2-1f262c8dc1363f4cb20c56b7aadcf2b5_1440w.jpg)

**3.修改程序集和命名空间名称（这一步可忽略）**

右键项目，选择属性，在打开的窗口中修改程序集和默认命名空间

![img](https://pic1.zhimg.com/v2-0bb5c0a434eb875f3c87c2592a9ddc40_1440w.jpg)

![img](https://pic1.zhimg.com/v2-536d2309f35e1dca47563e13532dd6be_1440w.jpg)

**4.全局替换项目名**

Ctrl+F 打开搜索， 搜索 OldName，将其替换为 NewName

![img](https://pica.zhimg.com/v2-db087a094c5871ec07716a8883a68872_1440w.jpg)

**5.修改项目文件夹名称**

1.右键解决方案，选择在文件资源管理器中打开
2.在打开的文件管理器中找到项目文件夹，重命名为 NewName

![img](https://pic4.zhimg.com/v2-9d4ac1db384545390c82c457b34bcfcd_1440w.jpg)

![img](https://picx.zhimg.com/v2-994373fb7b1e0940585dfeb4007f1d0f_1440w.jpg)

**6.修改.sln文件**

在项目文件夹下找到.sln文件，用文本编辑器打开，将文件中的旧项目名称修改为新项目名称

![img](https://pica.zhimg.com/v2-94e0e945e2d92b0f0f7f80e2b4d295ae_1440w.jpg)

**修改完成**

---

# 程序占用小技巧

**当你遇到“已经在另一个程序中打开”的提示时，通常是因为文件被其他程序占用。以下是几种常见的解决方法。**

**1.** **关闭相关程序**

**2.** **检查后台进程**

**3.** **使用资源监视器**

如果任务管理器未能找到占用进程，可以使用资源监视器进一步排查：

- 按 **Win + R** 输入 `resmon` 并回车打开资源监视器
- [在“CPU”选项卡下，展开“关联的句柄”，在搜索框中输入文件名，找到占用该文件的进程并结束它

---

