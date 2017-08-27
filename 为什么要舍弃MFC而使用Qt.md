# 为什么要舍弃MFC而使用Qt
## From_MFC_to_Qt

Categories: 技术博客
Tags: 编程开发 MFC Qt

---
看到这样一个标题，也许有的人觉得是废话，MFC作为一个20多年前出现的技术早已过时，现在微软都要抛弃它了，不快点舍弃还留着过年？但同时也有的人觉得不同的工具有不同的应用场合，学MFC能帮助理解windows底层API，只要windows不倒，MFC就不会过时。网上关于MFC的争论一直众说纷纭，我就说一说自己的体会吧。

---
研一还没开学前，实验室让我们新生提前过来培训基础知识，其中最主要的部分就是用MFC来编写程序。因此，实验室同学基本上都在用MFC，只有很少的人尝试C#、python、Qt之类的新工具，而且也不会把它们作为主要编程工具。后来选修了一门计算机图形学，老师同样要求我们学习MFC，整个课程所有作业都是基于MFC完成的。由此可见，现在有很多人还在用着MFC。

受周边的同学影响，我也用MFC有一年以上，当时一边用一边学，看孙鑫的MFC视频，查MSDN和各种博客。也许是当时还没有接触过别的编程工具的原因吧，用着倒也没觉得有太多不舒服的地方，甚至还用它做过一个视觉系统项目。

其实关于MFC过时的论调一直都有听说，到研二的时候，终于下决心要了解一下所谓的“先进”编程工具。一开始我选择Qt最重要的原因是它的编程语言也是C++，上手简单，而且几乎完美兼容我先前在MFC上用的第三方库，比如opencv和工业相机API库。接触一小段时间后，我发现Qt果然在很多地方比MFC好用很多！果断把所有项目转移到Qt上，此后便一直以Qt作为主要开发工具。到现在Qt也用了近一年了，以下是我关于Qt比MFC好的感触最深的几点：

# 界面的创建
MFC创建的界面上控件没法自动调整位置和大小，当窗口大小发生变化（比如最大化）时，界面布局不会跟着调整。之前我用MFC时编写了一个控件自动调整函数来解决这个问题：
```cpp
void ControlAutoPosition(CWnd* pWnd, int nID, int cx, int cy, int* posPram)
{
	int sizeBoundary = (cx > cy ? cx : cy);
	for (int i = 0; i < 8; i++)
		if (posPram[i] > sizeBoundary || posPram[i] < -sizeBoundary) return;
	CWnd* pCrl = pWnd->GetDlgItem(nID);
	if (!pCrl) return;
	CRect rect;
	pCrl->GetWindowRect(&rect);
	pWnd->ScreenToClient(&rect);
	if (posPram[0] >= 1) rect.top = (cy >> (posPram[0] - 1)) + posPram[1];
	if (posPram[2] >= 1) rect.bottom = (cy >> (posPram[2] - 1)) + posPram[3];
	if (posPram[4] >= 1) rect.left = (cx >> (posPram[4] - 1)) + posPram[5];
	if (posPram[6] >= 1) rect.right = (cx >> (posPram[6] - 1)) + posPram[7];
	pCrl->MoveWindow(rect);
}
```
其中pWnd、nID和cx、cy分别是窗口句柄、控件ID、和窗口当前的大小。posPram是控制参数列表，用8个参数来控制控件的大小以及与窗口的相对位置，代表一个控件的调整策略。对于窗口上每一个需要自动调整位置大小的控件，编程时需要手动确定posPram，然后在窗口OnSize()函数中调用ControlAutoPosition()就能使该控件按照调整策略自动改变位置大小。

这个办法一定程度上解决了MFC控件位置大小的问题，但仍然有相当大的缺陷。首先posPram的计算比较费事，需要调试出控件与窗口初始的大小和相对位置。其次窗口上有任何的设计变更，比如改变控件位置甚至增减控件，很可能会导致所有控件的posPram需要重新计算。

反观Qt，**layout机制**能够非常轻松地自动组织控件的位置和大小。而且用**Qt Designer**建立界面比MFC更简单直观，比如Qt的Tab Widget可以在Qt Designer中任意增减和编辑Tab，但在MFC中使用Tab Control则需要自己创建子Dialog作为Tab，然后写Tab Control的OnTcnSelchangeTab()响应函数来显示子Dialog，而且要自己控制子Dialog大小和位置，相当麻烦。

另外，**Qt中所有界面元素都可以自己通过代码创建**，比如：
```cpp
QPushButton *button = new QPushButton("&Download", this);
button->setGeometry(0, 0, 60, 20);
connect(button, SIGNAL(clicked()), this, SLOT(download()));
```
这三行代码就在窗口上创建了一个按钮，并设定了它单击触发的函数。因此，Qt的界面创建要比MFC灵活很多。

# 消息机制
MFC的消息映射机制与Qt的信号槽机制类似。每个MFC程序中都有一个消息映射表message map，定义了消息与响应函数的对应关系。举个例子，MFC中子线程是不能直接控制主线程的窗口进行UpdateData()操作的（Qt中就没有这样的问题，子线程可以直接控制主线程的窗口控件），只能通过消息映射来间接控制窗口刷新。首先添加自定义消息：
```cpp
#define WM_UPDATEDATAFALSE WM_USER+1
```
其次添加消息响应函数
```cpp
LRESULT CMyDlg::OnUpdateDataFalse(WPARAM wParam, LPARAM lParam)
{
	UpdateData(false);
	return LRESULT();
}
```
然后添加消息映射，把自定义消息与响应函数关联起来
```cpp
BEGIN_MESSAGE_MAP(CMyDlg, CDialogEx)
    ...
	ON_MESSAGE(WM_UPDATEDATAFALSE, &CMyDlg::OnUpdateDataFalse)
END_MESSAGE_MAP()
```
接下来就能在子线程中调用pThis->PostMessage(WM_UPDATEDATAFALSE)来让主线程窗口刷新了，其中pThis是主线程窗口指针。
这里要吐槽的是，MFC的线程函数和消息响应函数都有固定的格式，使用起来很不方便，尤其是需要传递多个参数时。

Qt的信号槽机制用起来给人的感觉则是简洁和灵活。任意两个不相关的类，即使在不同线程中，只要用connect()把它们的信号和槽连接起来就可以轻松地进行通信。引用[HarbinZJU的博客][1]，信号槽机制有以下特点：
>1. 类型安全：只有参数匹配的信号与槽才可以连接成功（信号的参数可以更多，槽会忽略多余的参数）。
2. 线程安全：通过借助QT自已的事件机制，信号槽支持跨线程并且可以保证线程安全。
3. 松耦合：信号不关心有哪些或者多少个对象与之连接；槽不关心自己连接了哪些对象的哪些信号。这些都不会影响何时发出信号或者信号如何处理。
4. 信号与槽是多对多的关系：一个信号可以连接多个槽，一个槽也可以用来接收多个信号。

补充一点：信号和信号之间也能连接，而且连接可以用disconnect()来移除。Qt中的QObject派生类（比如窗口、对话框、控件、线程、套接字等）都支持信号槽，想让自己定义的Class支持信号槽只需要同样继承QObject并在类中声明Q_OBJECT即可。信号和槽之间传递的参数没有固定格式，可以任意定义。

# 帮助文档
MFC的帮助文档是MSDN，如果想离线使用需要在Visual Studio中下载，也许是微软服务器的原因，感觉VS中下载非常慢，不如直接在线看。而且很多时候MSDN中的说明也不够详细，需要自己从别的地方找资料（不过MFC更容易在论坛博客中找到帮助确实是其优势）。

相对而言，Qt有着非常详尽的离线帮助文档和范例。假设你想知道Qt的标准对话框QFileDialog怎么用，按一下F1，很快就能找到完整详细的使用说明，这个Class有什么样的继承关系、函数、信号以及需要包括什么头文件都说的非常细致，比如它最常用的一个函数getOpenFileName()的说明：
![QFileDialog::getOpenFileName()][2]
用法、示例、注意点一应俱全，其中绿色部分是链接，方便访问相关的帮助文档。

我在做一个软件系统的时候需要用到局域网通讯，刚开始自己对UDP和TCP完全不了解，根本不知道从哪儿下手。好在Qt有很多相关的完整例子，在“示例”中搜索“network”：
![network examples][3]
打开其中任意一个示例能够看到全部源文件，可以直接编译运行。而且很多示例附带详细的说明文档，一步步教你如何编写这样一个程序。依靠这些示例，我几乎完全没有在网上查找别的教程就完成了自己的TCP Server和Client的编写。Qt的帮助文档做的非常用心，而且未来随着用Qt的人越来越多，论坛博客的资源肯定也不会是问题。

# OpenCV
由于我的研究方向是计算机视觉，整天跟OpenCV打交道，而with Qt编译的OpenCV加入了一些很有意思新功能（这些功能在MFC里是没有的，with Qt编译的OpenCV由于使用了QWidget而不能用在MFC中），参照OpenCV官方帮助文档[Qt New Functions][4]。
![OpenCv Gui][5]
调用cv::imshow()时显示的窗口有如图所示的用户交互界面，更让人兴奋的是里面的图像可以用鼠标滚轮放大，而且当放大到一定倍数的时候会直接显示每个像素点的通道值！
![cv::imshow()][6]
之前在MFC中想通过调试知道Mat中的值是特别麻烦的，因为Debug不能显示Mat的值。图像Mat经常有上百万个元素，有时候不知道元素的准确坐标，用遍历的方式查找相当不方便。而在Qt中这个问题动动鼠标就解决了，imshow一下便一目了然，这对程序的调试帮助非常大！强力推荐使用OpenCV的同学用Qt做开发工具。

---
顺带提一下，Qt其实也有一些不太好的地方：
## 静态编译
静态编译能够让程序发布变得更简单。Qt本身是开源项目，但同时也有商业版本。由于涉及版权问题，社区版Qt不能静态编译，网上虽然有很多自己静态编译的教程，但并不能用来发布涉及商业利益的程序。在只能动态链接的情况下，发布程序需要把Qt的dll也一起打包，主要有Qt5Core、Qt5Gui、Qt5Svg、Qt5Widgets和qwindows等。

## CUDA的使用
目前网上关于CUDA的教程基本都是基于Visual Studio的，所以在MFC中使用CUDA相对比较容易。刚转到Qt时CUDA这个问题我整了好久都没解决，相关资料特别难找，最终在Github上找到一个[示例][7]才勉强在Qt中用起了CUDA。其中涉及到混合编译的问题还是比较麻烦的。

## 中文支持
刚开始用Qt并没发现这个问题，因为有时候tr("中文")并不会出问题，用的多了才发现使用中文经常报错：
**error C2001: 常量中有换行符。**
这个问题其实也不难解决，首先确保cpp文件编码格式为**UTF-8**，再为cpp文件添加**UTF-8 BOM**，然后再用**QString::fromLocal8Bit("中文")**或者**QStringLiteral("中文")**修饰中文字符即可。

---
总而言之，Qt确实在很多方面比MFC更优秀，还在纠结于MFC的同学不妨尝试一下Qt。

参考资料：
\[1] [dulinbo：Qt vs MFC][8]
\[2] [hufeng825：比较QT和MFC两个界面库][9]
\[3] [FreeWick：彻底放弃没落的MFC，对新人的忠告！][10]
\[4] [知乎：MFC 过时了吗？][11]

---
作者：卞新光
来源：[blog.victorbian.rocks](http://blog.victorbian.rocks)
欢迎任何形式的转载，未经作者同意，请保留此段声明！



  [1]: http://blog.csdn.net/harbinzju/article/details/10813635 "【深入QT】信号槽机制浅析"
  [2]: http://ov1kvsbhe.bkt.clouddn.com/From_MFC_to_Qt/getOpenFileName.png "QFileDialog::getOpenFileName()"
  [3]: http://ov1kvsbhe.bkt.clouddn.com/From_MFC_to_Qt/Examples_network.png "network examples"
  [4]: http://docs.opencv.org/master/dc/d46/group__highgui__qt.html "Qt New Functions"
  [5]: http://ov1kvsbhe.bkt.clouddn.com/From_MFC_to_Qt/Qt_OpenCvGui.png "OpenCv Gui"
  [6]: http://ov1kvsbhe.bkt.clouddn.com/From_MFC_to_Qt/Qt_New_Functions.png "cv::imshow()"
  [7]: https://github.com/xixingyouyao/HelloCuda "HelloCuda"
  [8]: http://www.jxgolden.com/blog_csdn_net/dulinbo/article/details/4370581 "Qt vs MFC"
  [9]: http://blog.csdn.net/hufengvip/article/details/5748652 "比较QT和MFC两个界面库"
  [10]: http://bbs.csdn.net/topics/391817496 "彻底放弃没落的MFC，对新人的忠告！"
  [11]: https://www.zhihu.com/question/20730109 "MFC 过时了吗？"