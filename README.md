
## 一：背景


### 1\. 讲故事


上篇聊到了 `C#程序编译成Native代码` 的宏观过程，有粉丝朋友提了一个问题，能不能在 dotnet publish 发布的过程中对`AOT编译器`拦截进行源码级调试，这是一个好问题，也是深度研究的必经之路，这篇我们就来分享下吧。


## 二：托管和非托管调试器


### 1\. 调试器介绍


相信大家现在都知道`AOT Compiler (ilc.exe)` 是用 C\# 代码写的，也就表明它是一个托管程序，对托管程序的调试有两种调试器：


* Visual Studio 托管调试器


调试 C\# 代码它是当仁不让，缺点在于对非托管部分的查看缺少了手段。


* WinDbg 非托管调试器


调试 C/C\+\+ 是一把利器，但用它调试托管的C\#代码，虽然可以用，但在变量显示各方面不是很直观。


截个图如下，总之各有利弊吧：
![](https://img2024.cnblogs.com/blog/214741/202410/214741-20241016164724961-1708573090.png)


### 2\. 测试代码


为了方便演示，先上一段测试代码，非常简单。



```

    internal class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello, World!");
            Console.ReadLine();
        }
    }


```

有了代码之后，接下来一起观赏下如何通过 Visual Studio 和 Windbg 实现各自的拦截。


## 三：调试器拦截实战


### 1\. WinDbg 拦截


作为 Windows平台上王者般存在的非托管调试器，用它来劫持`ilc.exe`非常方便，在注册表的 `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\ilc.exe` 下配置个 Deubgger 键值即可，截图如下：


![](https://img2024.cnblogs.com/blog/214741/202410/214741-20241016164724955-1014567737.png)


接下来使用 `dotnet publish` 发布程序，稍等片刻之后会看到 windbg 立即启动拦截了 ilc.exe，然后 `ctrl+o` 打开我们需要下断点的 cs 文件，比如核心的 `Compilation` 方法，下完断点之后直接 g 执行，截图如下：


![](https://img2024.cnblogs.com/blog/214741/202410/214741-20241016164724976-1066747860.png)


从卦中可以看到 `Compilation.ctor` 果然命中，而且用 dv 也能看到各个局部变量的内存地址，是不是挺有意思的。


总的来说这种方式使用起来简单粗暴，但用 windbg 这种非托管调试器调试C\# 总有点 `名不正言不顺`，更好的方式应该还是用 visual studio 这种专业级的家宝，不是吗？


### 2\. Visual Studio 拦截


上一篇文章跟大家说过执行 `dotnet publish` 调用的ilc.exe 是来自于目录 `.nuget\packages\runtime.win-x64.microsoft.dotnet.ilcompiler\8.0.8\tools` 下的，截图如下：


![](https://img2024.cnblogs.com/blog/214741/202410/214741-20241016164724949-28039069.png)


为了能够用上托管调试器，这里我们把 ilc.sln 项目手工编译出一个 ilc.exe 来替换这里的 ilc.exe 即可，截图如下：


![](https://img2024.cnblogs.com/blog/214741/202410/214741-20241016164724719-1560992788.png)


为了能够让 VS 附加到 ilc.exe 进程上，ilc 提供了一个 `--waitfordebugger` 参数，参考如下：



```

PS D:\csharpapplication\21.20240910\src\Example\Example_21_1> ilc -h
Description:
  .NET Native IL Compiler

Usage:
  ilc -file-path>... [options]

Arguments:
  -file-path>  Input file(s)

Options:
  -?, -h, --help                  Show help and usage information
  ...
  --waitfordebugger               Pause to give opportunity to attach debugger


```

这个参数的作用就是通过 `Console.ReadLine` 让程序暂停，好让你用 VS 去 Attach ，源码中是这么写的：



```

        public Program(ILCompilerRootCommand command)
        {
            _command = command;

            if (Get(command.WaitForDebugger))
            {
                Console.WriteLine("Waiting for debugger to attach. Press ENTER to continue");
                Console.ReadLine();
            }
        }


```

但在我手工编译的 ilc.exe 上用 `Console.ReadLine` 貌似拦不住，所以这里稍微改一下，参考如下：



```

        public Program(ILCompilerRootCommand command)
        {
            _command = command;

            while (!Debugger.IsAttached)
            {
                Console.WriteLine("Waiting for debugger to attach. Press ENTER to continue");
                //Console.ReadLine();
                Thread.Sleep(1000);
            }
        }


```

接下来重新编译项目，将生成目录 `runtime\artifacts\bin\coreclr\windows.x64.Debug\ilc\net8.0` 下的所有文件复制到nugut目录 `.nuget\packages\runtime.win-x64.microsoft.dotnet.ilcompiler\8.0.8\tools` 下，截图如下：


![](https://img2024.cnblogs.com/blog/214741/202410/214741-20241016164724713-173435219.png)


一切都准备好之后，接下来使用 `dotnet publish` 重新发布程序，从 cmd 输出中可以看到正在等待 attach 附加。



```

PS D:\csharpapplication\21.20240910\src\Example\Example_21_1> dotnet publish  -r win-x64 -c Debug -o D:\testdump
  正在确定要还原的项目…
  所有项目均是最新的，无法还原。
  Example_21_1 -> D:\csharpapplication\21.20240910\src\Example\Example_21_1\bin\Debug\net8.0\win-x64\Example_21_1.d
  ll
  Generating native code
  Waiting for debugger to attach. Press ENTER to continue
  Waiting for debugger to attach. Press ENTER to continue
  Waiting for debugger to attach. Press ENTER to continue
  Waiting for debugger to attach. Press ENTER to continue
  Waiting for debugger to attach. Press ENTER to continue


```

在VS菜单上 Debug \-\> Attach to Process 到我们的 ilc.exe 进程，可以看到果然就命中了，大家看看这调试体验是不是高了很多，截图如下：


![](https://img2024.cnblogs.com/blog/214741/202410/214741-20241016164724939-1685744177.png)


体验过这种方式的朋友我相信又有一些新的问题，那就是重复调试的时候太麻烦了，能不能直接以 `启动程序` 的方式来调试？这就是接下来我们要聊的。


### 3\. VS 对ilc的启动调试


看过上篇的朋友知道，每一次AOT编译之前在 native 目录下都会有一个 xxx.ilc.rsp ，这个文件是 AOT Compiler 的 input 来源，截图如下：


![](https://img2024.cnblogs.com/blog/214741/202410/214741-20241016164724956-227107441.png)


所以完全可以将它作为 ilc.sln 项目的启动参数，接下来我们将 `@D:\csharpapplication\21.20240910\src\Example\Example_21_1\obj\Debug\net8.0\win-x64\native\Example_21_1.ilc.rsp` 放到 ILCompiler 项目的 command line 中，截图如下：


![](https://img2024.cnblogs.com/blog/214741/202410/214741-20241016164724701-1179863418.png)


配置好之后，接下来把 Example\_21\_1\.ilc.rsp 中的 `Example_21_1.dll,Example_21_1.obj,Example_21_1.def` 三块都改成全路径，参考如下：



```

D:\csharpapplication\21.20240910\src\Example\Example_21_1\obj\Debug\net8.0\win-x64\Example_21_1.dll
-o:D:\csharpapplication\21.20240910\src\Example\Example_21_1\obj\Debug\net8.0\win-x64\native\Example_21_1.obj
-r:C:\Users\Administrator\.nuget\packages\microsoft.netcore.app.runtime.win-x64\8.0.8\runtimes\win-x64\lib\net8.0\WindowsBase.dll
-r:C:\Users\Administrator\.nuget\packages\runtime.win-x64.microsoft.dotnet.ilcompiler\8.0.8\sdk\System.Private.CoreLib.dll
...
--targetarch:x64
--dehydrate
-g
--exportsfile:D:\csharpapplication\21.20240910\src\Example\Example_21_1\obj\Debug\net8.0\win-x64\native\Example_21_1.def
...


```

直接 F5 启动 ILCompiler 项目，可以看到轻轻松松的成功调试，这种方式就很好的解决了反复调试的问题，截图如下：


![](https://img2024.cnblogs.com/blog/214741/202410/214741-20241016164724935-769342570.png)


## 三：总结


以劫持的方式对 AOT Compiler 自身进行源码级调试，这本身就是一个很有意思的话题，不断的介入Compiler编译的的各个阶段，相信能给大家深度学习AOT提供了一些不寻常的手段。
![图片名称](https://images.cnblogs.com/cnblogs_com/huangxincheng/345039/o_210929020104%E6%9C%80%E6%96%B0%E6%B6%88%E6%81%AF%E4%BC%98%E6%83%A0%E4%BF%83%E9%94%80%E5%85%AC%E4%BC%97%E5%8F%B7%E5%85%B3%E6%B3%A8%E4%BA%8C%E7%BB%B4%E7%A0%81.jpg)


 本博客参考[veee加速器](https://liuyunzhuge.com)。转载请注明出处！
