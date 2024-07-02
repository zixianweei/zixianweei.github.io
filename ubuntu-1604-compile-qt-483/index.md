# Ubuntu16下编译Qt4.8.3源码


本篇记录了在Ubuntu系统中编译Qt4的过程。

<!--more-->

## 编译环境

+ Ubuntu 16.04.7
+ GCC 4.9
+ Qt 4.8.3

首先贴出Qt源码的下载地址 `https://download.qt.io/archive/qt/`。这个网站相对国内的镜像站更全(国内的镜像站似乎没有Qt4源码)，但是缺点是比较慢，有梯子的话应该会更快。

下面就可以在本地计算机上进行编译相关的操作了，在这里我将下载好的源码放在 `$HOME/Downloads`路径下。大家可以根据自己的实际情况进行相关的修改。

## 解压源码压缩包

这里可以使用系统自带的GUI解压工具解压，也可以在终端中使用命令解压。终端中解压命令为:

```shell
tar -zxvf qt-everywhere-opensource-src-4.8.3.tar.gz
cd qt-everywhere-opensource-src-4.8.3
```

解压后，使用```cd```命令进入解压后的路径，后续的主要操作都在这个目录下进行。

## 安装依赖项

这里需要安装的依赖项很多。如果有模块不需要安装，则可以不安装相关依赖。不过这里推荐全部安装，万一以后会用到呢。

```shell
sudo apt install -y gcc-4.9 g++-4.9 make cmake gdb build-essential
sudo apt install -y libx-dev libxext-dev libxtst-dev
sudo apt install -y openssl libssl-dev
sudo apt install -y g++-multilib zlib1g-dev autoconf automake libtool
sudo apt install -y libgl1-mesa-dev libglu1-mesa-dev
sudo apt install -y libglib2.0-dev libalsa-ocaml-dev
sudo apt install -y xorg-dev gperf bison flex sqlite
sudo apt install -y libxrender-dev libxrandr-dev libedbus-dev libdbus-1-dev
sudo apt install -y libgstreamer0.10-dev libgstreamer-plugins-base0.10-dev
sudo apt install -y libxt-dev subversion libsqlite3-dev libpng12-dev 

# 64位机器
sudo apt install -y lib32ncurses5 lib32z1
# 32位机器
sudo apt install -y libx32ncurses5 libx32z1
```

## 修改相关文件

这里是Qt能否编译成功的关键所在。主要有3个地方需要修改，我将一一说明。

+ 将系统默认的GCC/G++命令链接到刚刚安装的4.9版本的GCC中。需要这一步的原因是，GCC5以上的GCC是始终无法打开WebKit编译选项的，所以如果需要WebKit就需要更换到4.9版本的GCC。

  ```shell
  # 建立gcc和g++的软链接
  sudo ln -sf /usr/bin/gcc-4.9 /usr/bin/gcc
  sudo ln -sf /usr/bin/g++-4.9 /usr/bin/g++
  ```

+ 将GCC的C++编译版本调整为C++98标准。默认情况下，编译器似乎是启用了C++11标准，将会出现很多奇怪的错误，例如```narrowing conversion```相关的错误。这里需要修改两个文件的CXX_FLAGS。

  ```shell
  # mkspecs/common/g++-base.conf 第18行
  QMAKE_CXX = g++ 改为 QMAKE_CXX = g++ -std=gnu++98
  # mkspecs/common/gcc-base.conf 第45行
  QMAKE_CXXFLAGS += $$QMAKE_CFLAGS 改为 QMAKE_CXXFLAGS += $$QMAKE_CFLAGS -std=gnu++98
  ```

+ 修改JavaScriptCore.pri文件中的内容，这个文件和WebKit编译相关，修改之后WebKit才能编译通过。具体的链接在[这里](https://bugs.webkit.org/show_bug.cgi?id=82824)，这是WebKit的一个bug。

  ```shell
  sed -i -e '/CONFIG\s*+=\s*text_breaking_with_icu/ s:^#\s*::' \
  	src/3rdparty/webkit/Source/JavaScriptCore/JavaScriptCore.pri
  ```

到这里，编译Qt4最关键的部分就完成了。

## 编译相关

Qt在默认情况下编译的是动态链接库，也可以通过添加```-static```选项编译静态库，但是编译静态库是无法启用WebKit的。(我也尝试编译了静态库，死在了最终链接的时候)

编译的流程是先通过configure生成Makefile文件，然后使用make编译，最终install。具体命令如下：

```shell
./configure -opensource -nomake demos -nomake examples -nomake tests -silent -webkit
make -j4
sudo make install
```

其中，```-nomake```选项关闭了Qt例子和测试，这样可以节省编译的时间，有需要的可以打开。

我使用的编译平台是：```Intel Core i3-2350M```+```DDR3-1333MHz 6GB```，编译全程耗时大概40分钟左右。如果你的机器更好可以更改```make -j4```后面的数字。默认情况下，安装路径为：```/usr/loca/Trolltech/Qt-4.8.3```。如果需要自定义的话，可以在configure的时候使用```-prefix```选项进行修改，这里我使用的是默认安装路径。

## 添加环境变量

在编译安装完成后，我们在终端中输入```qmake```，会发现仍未找到相关命令或者并不是我们需要的4.8.3版本。这是因为还需要在系统中添加环境变量才可以使我们自行安装的Qt生效。这里我选择的是更改```$HOME/.bashrc```文件。这个文件是当前账户bash环境的环境变量控制文件，使用zsh的在```$HOME/.zshrc```文件中进行相关修改即可。

在环境变量文件的最后添加下面四行：

```shell
export PATH="/usr/loca/Trolltech/Qt-4.8.3/bin:$PATH"
export LD_LIBRARY_PATH="/usr/loca/Trolltech/Qt-4.8.3/lib:$LD_LIBRARY_PATH"
export MANPATH="/usr/loca/Trolltech/Qt-4.8.3/man:$MANPATH"
export PKG_CONFIG_PATH="/usr/loca/Trolltech/Qt-4.8.3/lib/pkgconfig:$PKG_CONFIG_PATH"
```

稍微解释下每一个变量是干什么的：```PATH```主要负责可执行路径，```LD_LIBRARY_PATH```负责库文件，```MANPATH```负责终端文档，```PKG_CONFIG_PATH```负责pkgconfig文件配置。

## 总结

到这里，整个Qt4.8.3编译安装配置都完成了。其实看上去也没啥工作量，但是一开始尝试编译的时候各种报错还是挺难受的。我将我遇到的问题和解决方法整理在这里，欢迎讨论和私信我，我会尽量及时回复。

下一期将介绍FFMpeg编译的相关流程，敬请期待吧。:)

