---
title: 从0开始学习opencv（一）-配置环境
abbrlink: a2eaa8a6
date: 2019-10-27 01:43:39
tags: [opencv]
categories: 游戏
---

# Mac环境，安装OpenCV，VScode调试C++程序

## 环境说明

- macOS版本 catalina 10.15版本
- opencv3
- vscode



## 步骤

1. 安装XCode工具Command Line `**sudo** xcode-select --install`

   

2. 安装homebrew，可以用Homebrew安装很多东西

   ```shell
   /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
   ```

   brew下载后发生很慢的情况，是因为某种不可描述的原因。这时候需要国内镜像

   ```shell
   git -C "$(brew --repo)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git
   
   git -C "$(brew --repo homebrew/core)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git
   
   git -C "$(brew --repo homebrew/cask)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-cask.git
   
   brew update
   ```

   

3. 安装cmake，如果你用的`homebrew`方式安裝`opencv`那麼CMake就不是必須的.

   到https://cmake.org/download/下下载cmake安装包安装

   ![image](https://user-images.githubusercontent.com/7789698/67623984-33222200-f85e-11e9-88fd-a132f96d7a57.png)

   或者使用`brew install cmake`

   

4. 安装opencv,使用brew安装`brew install opencv@3`

   

   如果遇到`Cloning into '/Users/wangwenwei/Library/Caches/Homebrew/aom--git'...
   fatal: unable to access 'https://aomedia.googlesource.com/aom.git/': Failed to connect to aomedia.googlesource.com port 443: Operation timed out`肯定又是因为某种不可描述的问题，无法安装aom

   解决方法如下：

   ```
   $ wget https://raw.githubusercontent.com/Homebrew/homebrew-core/master/Formula/aom.rb
    
   # 用本站提供的一份代码拷贝来替代原来的地址
   $ sed -i "" "s/https:\/\/aomedia\.googlesource\.com\/aom\.git/https:\/\/www.mobibrw.com\/wp-content\/uploads\/2019\/04\/aom.zip/g" aom.rb
    
   $ brew uninstall --ignore-dependencies aom
    
   $ brew install --build-from-source aom.rb --env=std
   ```

   (如果某种不可描述的原因连wget都不行，则直接访问https://raw.githubusercontent.com/Homebrew/homebrew-core/master/Formula/aom.rb把内容拷贝到aom.rb)

   

5. 写个demo测试gcc可用性

   新建编辑main.cpp

   ```
   /// ./main.cpp
   #include <stdio.h>
   #include <iostream>
   
   int main(int argc, const char * argv[]) {
       std::cout << "absurd!\n";
       return 0;
   }
   ```

   命令：

   ```
   absurd$ g++ -g ./main.cpp -o ./main.o
   absurd$ ./main.o
   absurd!
   absurd$
   ```

   

6. 配置pkg-config

   由 `brew list opencv`可以看到现在opencv的路径，拷贝相关配置

   ```
   cp /usr/local/Cellar/opencv/4.1.2/lib/pkgconfig/opencv4.pc /usr/local/lib/pkgconfig/opencv.pc
   ```

   配置bash环境（永久的走/etc/profile）

   ```
   absurd$ echo PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/lib/pkgconfig >> ~/.bash_profile
   absurd$ echo export PKG_CONFIG_PATH >> ~/.bash_profile
   absurd$ source ~/.bash_profile
   ```

   配置完后执行

   `pkg-config opencv --libs --cflags opencv`

   

7. 测试opencv的DEMO

   ![image](https://github.com/QianMo/OpenCV3-Intro-Book-Src/blob/master/OpenCV3-examples/src/%E3%80%901%E3%80%91%E7%AC%AC%E4%B8%80%E7%AB%A0/%E3%80%901%E3%80%91OpenCV%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83%E7%9A%84%E9%85%8D%E7%BD%AE/1_HelloOpenCV/1.jpg?raw=true)

   新建test.cpp

   ```
   #include <opencv2/opencv.hpp> //头文件
   using namespace cv; //包含cv命名空间
   
   int main()
   {
   	// 【1】读入一张图片
   	Mat img=imread("1.jpg");
   	// 【2】在窗口中显示载入的图片
   	imshow("【载入的图片】",img);
   	// 【3】等待6000 ms后窗口自动关闭
   	waitKey(6000);
   }
   ```

   编译`g++ `pkg-config opencv --libs --cflags opencv` ./test.cpp -o ./test.o`,

   执行`./test.o`

   可以看到弹出图片

   

8. 下载安装vscode，https://code.visualstudio.com/

   安装插件C/C++、C++ Intellisense、C++ Clang Command Adapter、Chinese (Simplified) Language Pack for Visual Studio Code（中文语言包，看个人喜好）

   ![image](https://user-images.githubusercontent.com/7789698/67624402-af1e6900-f862-11e9-805f-5667049e317a.png)

9. 配置vscode

   ![image](https://user-images.githubusercontent.com/7789698/67629549-51197200-f8b2-11e9-9cea-2e9ff7906105.png)

   - “command+shift+p”打开命令行工具窗口，输入或者选择“Edit Configurations”命令。

     此时会在当前工作空间目录生成.vscode配置目录，同时在配置目录会生成一个c_cpp_properties.json文件。

     includePath使用的opencv通过` ~ brew list opencv@3`获取到include

     ```
     {
         "configurations": [
             {
                 "name": "Mac",
                 "includePath": [
                     "/usr/local/include",
                     "/Library/Developer/CommandLineTools/SDKs/MacOSX10.15.sdk/usr/bin",
                     "/usr/local/Cellar/opencv@3/3.4.5_6/include/",      
                     "${workspaceFolder}/**"
                 ],
                 "defines": [],
                 "macFrameworkPath": [
                     "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/System/Library/Frameworks"
                 ],
                 "compilerPath": "/usr/bin/clang",
                 "cStandard": "c11",
                 "cppStandard": "c++17",
                 "intelliSenseMode": "clang-x64"
             }
         ],
         "version": 4
     }
     ```

     

   - “command+shift+p”打开命令行工具窗口，输入或者选择“Tasks: Configure Task”

     配置 tasks.json文件

     

     ![image](https://user-images.githubusercontent.com/7789698/67629567-7a3a0280-f8b2-11e9-94c1-c9942d9d4bae.png)

     

     

     

     ```
     {
     // 有关 tasks.json 格式的文档，请参见
         // https://go.microsoft.com/fwlink/?LinkId=733558
         "version": "2.0.0",
         "tasks": [
             {
                 "type": "shell",
                 "label": "g++ build active file",
                 "command": "/usr/bin/g++",
                 "args": [
                     "-g",
                     "${file}",
                     "-o",
                     "${fileDirname}/${fileBasenameNoExtension}.out",
                     "`pkg-config",
                     "--libs",
                     "--cflags",
                     "opencv`"
                 ],
                 "options": {
                     "cwd": "/usr/bin"
                 },
                 "problemMatcher": [
                     "$gcc"
                 ],
                 "group": "build"
             }
         ]
     }
     ```

     

   - 配置launch.json。“command+shift+p”打开命令行工具窗口，输入或者选择**Debug: Open launch.json**命令。

     ```
     {
         // 使用 IntelliSense 了解相关属性。 
         // 悬停以查看现有属性的描述。
         // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
         "version": "0.2.0",
         "configurations": [
             {
                 "name": "g++ build active file",
                 "type": "cppdbg",
                 "request": "launch",
                 "program": "${workspaceFolder}/${fileBasenameNoExtension}.out",
                 "args": [],
                 "stopAtEntry": false,
                 "cwd": "${workspaceFolder}",
                 "environment": [ {"name": "PKG_CONFIG_PATH", "value": "/usr/local/lib/pkgconfig"},   // 這是opencv解壓碼後創建的release目錄下的unix-install, 要保證該目錄下下有opencv.pc文件
                     {"name": "DYLD_LIBRARY_PATH", "value": "/usr/local/opencv/build/lib"}   // 這個是你在編譯時，opencv make時`CMAKE_INSTALL_PREFIX`指定的目錄
                 ],
                 "externalConsole": true,
                 "MIMode": "lldb",
                 "preLaunchTask":"g++ build active file"
             }
         ]
     }
     ```

     

   

10. 开始调试

![image](https://user-images.githubusercontent.com/7789698/67629588-c422e880-f8b2-11e9-86c1-e3f20ab171c7.png)

调试如果出现图片就是成功了

![image](https://user-images.githubusercontent.com/7789698/67629814-00584800-f8b7-11e9-9173-d295d6daf500.png)



如果类似上图显示波浪形，就是c_cpp_properties.json的includePath没有引用到正确的库，配置下就好了

