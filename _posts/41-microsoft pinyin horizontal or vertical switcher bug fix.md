---
tags:
- 杂七杂八
title: microsoft pinyin horizontal or vertical switcher bug fix
date: 2020-04-23
updated: 2020-04-23
issueid: 41
---
如果你使用`Windows 2004H`版本的`pinyin` 输入法，那么有一定概率 `horizontal/vertical` 失效

如果下图 `vertical` 模式，你不管点击多少次，永远不能切换成 `horizontal` 模式
![image](https://user-images.githubusercontent.com/24750337/80048104-cf915c80-8541-11ea-89c9-2c7dc10c907e.png)

在网上英文中文，`windows feedback`都查了个遍，反馈Bug也没人搭理，无果，只能靠自己的知识来解决了

首先按照常理，`windows`下系统自带软件的配置往往存放在注册表，而非本地配置文件，我们借助 `procmon`这个软件来监控 `SystemSettings.exe`进程对于注册表的操作，打开 `Settings` 然后点到输入法界面随便进行几点操作，发现对 `InputMethod`进行的操作

![image](https://user-images.githubusercontent.com/24750337/80048321-74139e80-8542-11ea-9e06-e8abddd27172.png)

欣喜若狂，打开注册表定位到具体的键值，发现Windows的输入法配置都存放在这里，根据上面英文单词的含义，我们找到了`Computer\HKEY_CURRENT_USER\SOFTWARE\Microsoft\InputMethod\Settings\CHS`文件夹，

*我这里没有CHS是因为我通过删除注册表来进行重制，所以CHS文件夹没了*
![image](https://user-images.githubusercontent.com/24750337/80048558-1895e080-8543-11ea-9294-a64e2e460461.png)

但删除之前我做了备份，CHS大体内容如下

![image](https://user-images.githubusercontent.com/24750337/80048553-16338680-8543-11ea-8a8e-b32ab02b6392.png)

为了确保删除不出岔子，我提前对`InputMethod`文件夹导出了注册表项，做备份

这时候直接删除CHS，然后输入框会直接消失不见，在 `Settings`下连续开关多次 ·IME toolbar·

![image](https://user-images.githubusercontent.com/24750337/80048660-66aae400-8543-11ea-868f-ee96fad2194b.png)

最终成功还原为`IME horizontal`

![image](https://user-images.githubusercontent.com/24750337/80048681-74f90000-8543-11ea-9480-7e12d66575ef.png)

*自己动手丰衣足食，如果自己不爱鼓捣计算机，平常不练习 ProcMon 这些系统管理软件的使用，那么碰到这个问题只能抓瞎，这个问题困扰我很久了，今天实在看不习惯这个vertical布局，才下决心解决这个问题*
