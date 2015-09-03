title: Build aapt for windows
layout: post
categories: Android
tags: [Android,aapt,Android source,English Post]
---

#Background
In order to practise my english, I am trying to writing some posts in english. So this article may confuse some people, I'm really sorry for that. By the way, if there some some typos, grammatical mistake and other mistakes, please feel free to point it out. I really appreciate it much. Thanks.


For some reasons, I decided to compile the appt from scratch.  
Target: **aapt for windows.**  
Then I moved on.  
However,It’s somehow ridiculous, if we want to  build aapt for windows we have to build it on linux according to the official manual.
``` bash
Full Windows SDK builds are only supported on Linux -- most of the framework is not designed to be built on Windows so technically the Windows SDK is build on top of a Linux SDK where a few binaries are replaced.
http://tools.android.com/build
```
It is said that Firefox for android also has to be built on \*unix system. But it has nothing to do with this article.
#Steps
Anyway, let’s do it step by step.
##Install Linux
First Install Linux 64 bit(important, other versions may be different with this post), I chose the  ubuntu-14.04.2-desktop.
http://releases.ubuntu.com/14.04.2/ubuntu-14.04.2-desktop-amd64.iso.torrent
I encountered the low screen resolution in Ubuntu, as installed it on a virtual machine. Eventually solved it through this link.http://www.linuxbsdos.com/2014/10/31/solutions-for-low-screen-resolution-in-ubuntu-14-0414-10-and-virtualbox/

##Install jdk
Install Java 7: For the latest version of Android
``` bash
$ sudo apt-get update
$ sudo apt-get install openjdk-7-jdk
```
Optionally, update the default Java version by running:
``` bash
$ sudo update-alternatives --config java
$ sudo update-alternatives --config javac
```

##Install some necessary tools
Install some other necessary tools.
``` bash
$ sudo apt-get install bison g++-multilib git gperf libxml2-utils make python-networkx zlib1g-dev:i386 zip
```
Something wrong happens.  
*The following packages have unmet dependencies:
 g++-multilib : Depends: gcc-multilib (>= 4:4.8.2-1ubuntu6) but it is not going to be installed
                Depends: g++ (>= 4:4.8.2-1ubuntu6) but it is not going to be installed
E: Unable to correct problems, you have held broken packages.*  
http://www.cnblogs.com/wanyuanchun/p/3738938.html

So intall gcc-multilib and g++ first.
``` bash
$ sudo apt-get install gcc-multilib g++
```
##Install repo and download source
Install repo to download source
http://source.android.com/source/downloading.html#installing-repo  
``` bash
$ mkdir ~/bin
$ PATH=~/bin:$PATH
$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
$ chmod a+x ~/bin/repo
```
config git  

``` bash
$ git config --global user.email "you@example.com"
$ git config --global user.name "Your Name"
```
init source  
``` bash
$ mkdir ~/android
$ repo init -u https://android.googlesource.com/platform/manifest -b master -g all,-notdefault,tools
```
For guys in mainland China, you should try the command below, for two reason one is  Google is blocked ,the other is it more faster.
``` bash
$ repo init -u git://aosp.tuna.tsinghua.edu.cn/android/platform/manifest -b master -g all,-notdefault,tools
```
Then start the downloading.  
``` bash
$ repo sync
```
And wait for the downloading to end, about 33G when the time I downloaded it.  
![source size][1]
So better downloaded at night, then have a good sleep. When the time you wake up, things is ready.  

##Install mingw32
For the build of windows' exe and dll files.
``` bash
$ sudo apt-get install mingw32 tofrodos
```
##Compile the source
When source downloaded, we can compile it now.
``` bash
$ cd ~/android
$ . build/envsetup.sh
$ lunch sdk-eng
$ make win_sdk
```
Note that this will build both a Linux SDK and a Windows SDK.  
The result is located at out/host/windows/sdk/android-sdk_eng.${USER}_windows/
If you encounter error,*"error while loading shared libraries: libncurses.so.5”. *  
Install lib32ncurses5, then use make win_sdk to build it again.
``` bash
$ sudo apt-get install lib32ncurses5
```

Later I got  another error when building libwebviewchromium.
*"collect2: error: ld terminated with signal 9 [Killed]
make: *** [out/target/product/generic/obj/SHA-j4 win-sdkRED_LIBRARIES/libwebviewchromium_intermediates/LINKED/libwebviewchromium.so] Error 1 "*
This because the system do not have a swap file.    Let's create it.
``` bash
$ dd if=/dev/zero of=/swapfile bs=1G count=2
```
Verify that file has been created on the server:  
``` bash
$ ls -lh /swapfile
```
Type the following chmod command and chown command to secure and set correct file permission for security reasons.
``` bash
sudo chown root:root /swapfile
sudo chmod 0600 /swapfile
```
Turn on the swap file  
``` bash
sudo mkswap /swapfile
sudo swapon /swapfile
```
Verify new swap file and settings on Ubuntu
``` bash
swapon -s
```

Now we can build the window sdk.
``` bash
make win_sdk
```

After some hours, which depends on your machine, all things will finish.  
We can get appt.exe from the out folder.  

If ou  only want to compile some tools, not all of them.
Just Do below.  
For linux:
``` bash
sudo make aapt
```
For windows(Please execute make aapt first to generate some libraries):
``` bash
sudo USE_MINGW=1 make aapt
```

#More
For custom needs, you can modify the source and rebuild it.  
Due to NDA(Non-Disclosure Agreement), I skip this chapter.  

#References
- http://www.timelesssky.com/blog/building-android-sdk-build-tools-aapt-for-debian-arm
-  http://source.android.com/source/initializing.html#choosing-a-branch
- https://android.googlesource.com/platform/sdk/+/master/docs/howto_build_SDK.txt
- http://tools.android.com/build
- http://source.android.com/source/using-repo.html
- http://www.cyberciti.biz/faq/ubuntu-linux-create-add-swap-file/


  [1]: https://github.com/waylife/blogimage/raw/master/2015/08/01/android_sdk_source_count.png
