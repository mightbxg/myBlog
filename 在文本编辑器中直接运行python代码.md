# 在文本编辑器中直接运行python代码
## Run_Python_in_Texteditor

Categories: 技术博客
Tags: 编程开发 python

---
python作为一种解释性语言用起来感觉和bat或者shell脚本差不多，随便找个文本编辑器敲入代码保存后便可以执行。虽然用起来已经很方便了，可是调试的时候每次运行都要切出去手动执行py文件还是感觉有点麻烦，尤其在Linux中还得开终端运行。为了把偷懒做到极致，得搞个法子在文本编辑器中直接运行python代码。

---
# Windows
Windows下的方法网上很容易找，比如[这里][1]。
首先文本编辑器要用Notepad++（其它带有“运行”功能的文本编辑器也行）。
在Notepad++中点击“运行”=>“运行”(F5)，在弹出的对话框中输入：
```bat
cmd /k python "$(FULL_CURRENT_PATH)" & PAUSE & EXIT
```
[![Notepad++ Run][2]][2]
这一句命令是三个bat命令的组合，引用[Blitz_Man的博客][1]：

> 1. cmd /k：执行后面命令，执行之后保留窗口。 
2. cmd /c：执行后面命令，执行之后关闭窗口。 
3. Python "$(FULL_CURRENT_PATH)"：FULL_CURRENT_PATH是notepad的宏定义，代表当前文件的完整路径。 
4. PAUSE：运行结束后暂停，等待按键。 
5. EXIT：关闭窗口。 
6. &：连接多条命令

***补充：*** Windows下Notepad++中"\$(FULL_CURRENT_PATH)"代表当前文件的完整路径，比如“D:\Users\Desktop\1.py”因此是可以直接作为python参数来执行的（对比后面Linux）。
如果想指定运行的python版本，可以把命令的中“python”替换成python完整路径，比如
"C:\Python27\python.exe"（python2.7），或者
"C:\Users\ABC\AppData\Local\Programs\Python\Python36\python.exe"（python3.6）。

最后，设置一下快捷键就行啦。打开“运行”=>“管理快捷键”，找到刚刚保存的运行命令加上快捷键即可，我喜欢设置为Ctrl+R（同Qt）。

---
# Linux
重点来了，我以Ubuntu为例介绍一下Linux系统的配置方案。
首先文本编辑器选用[Notepadqq][3]，它又被称为Linux版的Notepad++。安装方法：
```bash
sudo add-apt-repository ppa:notepadqq-team/notepadqq
sudo apt-get update
sudo apt-get install notepadqq
```
网上也有一些关于Notepadqq一键运行python的方法，但都不对，比如[这里][4]说Run的命令为：
```bash
gnome-terminal -x python %filename%
```
“gnome-terminal -x”表示开启终端并执行后面的命令，但是"%filename%"只是当前文件名，不包含文件路径，因此是不能被python执行的。Notepadqq中还有一个占位符"%fullpath%"是当前文件的完整路径（包含文件名），可是这个占位符多了"file://"前缀，比如："file:///home/VictorBian/Documents/1.py"，因此同样不能被python执行。

我的方法是用shell脚本来间接运行python，简单来说就是用shell脚本接收"%fullpath%"参数，然后去掉"file://"前缀并传递给python执行。用shell脚本就变得特别灵活了，有两个额外的好处：一是可以用shell命令控制python代码运行结束后不退出，类似bat中的pause；二是可以顺便给当前文件赋予执行权限（Linux中没有赋予执行权限之前python代码是不能执行的）。具体操作如下：

新建文档，输入以下代码，然后重命名为"runpy"，丢到/usr/bin目录下并用"chmod +x runpy"为其赋予执行权限。
```shell
#!/usr/bin/zsh

pause(){
MSG="$1"
if [ -z "$MSG" ]; then
	MSG="Press any key to continue..."
fi
echo ""
echo -n "$MSG"
SAVEDSTTY=`stty -g`
stty -echo
stty cbreak
dd if=/dev/tty bs=1 count=1 2> /dev/null
stty -raw
stty echo
stty $SAVEDSTTY
}

FILEPATH=${1#*file://}
if [ ! -x "$FILEPATH" ]; then
	chmod +x "$FILEPATH"
fi
python3 "$FILEPATH"

pause
```
***注意：***第一行是指定终端的位置，我用zsh所以是"#!/usr/bin/zsh"，默认终端是"#!/bin/sh"。"python3"可以改成"python"来调用默认的python版本。

***解析：***
**1. pause()函数**
参考自[陆小K的博客][5]，用于实现类似bat中pause的功能，也就是“按任意键继续”。无参数时输出默认消息"Press any key to continue..."，也可以接受一个参数来自定义输出消息，这个函数可以直接拷到别的shell脚本中，还是特别实用的。
**2. FILEPATH=\${1#\*file://}**
从命令行参数%1中截取"file://"之后的内容保存到"FILEPATH"中，即目标文件完整路径。
**3. [ ! -x "\$FILEPATH" ]**
判断目标文件是否有执行权限。
**4. chmod +x "\$FILEPATH"**
为目标文件赋予执行权限。
**5. python3 "$FILEPATH"**
用python3运行目标文件，由于我的系统默认python是2，需要强制指定python3。
**6. pause**
调用pause函数，输出等待消息，阻断终端窗口等待用户按键后退出。

然后配置Notepadqq，点击“Run=>Modify Run Commands”，弹出的对话框中“Text”输入“Run Python”（任意），“Command”输入以下命令后OK保存。
```shell
gnome-terminal -x runpy "%fullpath%"
```
[![Notepadqq Run][6]][6]
该命令表示在终端中运行runpy并传入命令行参数"%fullpath%"。最后点击“Settings=>Preferences”，在“Shortcuts”中找到“Run=>Run Python”，自己定义一个快捷键即可（比如Ctrl+R）。

至此，Notepadqq也能够直接运行python啦。写好代码之后先Ctrl+S保存，再Ctrl+R便自动打开终端运行，相当方便。而且runpy脚本也能在终端中使用，比如当前目录下刚刚创建了一个“1.py”文件，在终端中输入“runpy 1.py”便可以直接运行，省去了“chmod +x 1.py”操作。

---
作者：卞新光
来源：[blog.victorbian.rocks](http://blog.victorbian.rocks)
欢迎任何形式的转载，未经作者同意，请保留此段声明！



  [1]: http://blog.csdn.net/xiakepan/article/details/50560333 " notepad++自动运行Python"
  [2]: https://blog.victorbian.rocks/wp-content/uploads/2017/09/notepadpp_run.png "Notepad++ Run"
  [3]: http://notepadqq.altervista.org/s/ "Notepadqq"
  [4]: http://www.infocool.net/kb/Python/201703/308140.html "Linux下Notepadqq配置Python脚本一键运行命令"
  [5]: http://luxiaok.blog.51cto.com/2177896/1153190/ "shell脚本中的“请按任意键继续”"
  [6]: https://blog.victorbian.rocks/wp-content/uploads/2017/09/Ubuntu_runpy.png "Notepadqq Run"