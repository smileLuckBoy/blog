---
title: MonkeyRunner简述
date: 2017-02-27 22:10:36
toc: true
tags: 自动化
categories: JAVA
---

闲来有空，研究了下ANDROID下APP自动化测试，说来也简单，就是调用ANDROID SDK下的MonkeyRunner的API，这里有python版以及java版本的，这里选取了java版本的作为研究对象，不过这API居然特么的没有文档说明，简直是坑，现在做个简单的记录，方便自己和后来人阅读。

<!-- more -->

### 环境搭建

#### JDK
这个就不必多说了，这里采用的JDK8版本
#### ANDROID SDK
这个还是简单介绍下，我们只需要用到platform-tools和tools文件下的内容，所以在下载完成sdk后，保留这两个文件夹即可，之后配置好环境变量，在cmd窗口下输入adb后能正常输出内容即配置成功，具体可参考[Android SDK的安装与环境变量配置](http://www.cnblogs.com/fangbo/p/3941178.html)，这里就不做赘述了。
#### 模拟器
ANDROID模拟器现在可谓五花八门，什么夜神啦，天天啦，海马玩啦，总之很多很多，这里推荐Geymotion，一款流畅到和真机媲美的模拟器，不过Geymotion不支持ARM架构的APP，需要装一个Genymotion-ARM-Translation.zip桥即可，需要注意的是，ARM桥需要和模拟器ANDROID版本匹配，当然，最好采用真机进行测试啦（土豪请随意）。


### API简介
#### 第三方JAR
环境搭建好就可以撸代码了，我们主要用到了**tools/lib**文件夹下的如下几个包：
himpchat.jar、common.jar、ddmlib.jar、juava.jar、hierarchyviewer2lib.jar、jython-standalone.jar、monkeyrunner.jar、sdklib.jar、tools.jar

还有另外一个需要用到的：
org.eclipse.swt.jar

其中比较重要的是chimpchat.jar和ddmlib.jar，这两个包封装了对模拟器操作的核心代码，之后的大部分二次开发都是基于这两个包。

#### API
##### AdbBackend
这个类主要提供发现ADB，连接设备等功能，相当于一个连接池等作用。

##### 发现设备
adbBackend.waitForConnection(millsTimeout, deviceId);
这个API作用就是连接安卓设备，这里既包含了模拟器，也包含了真机，第一个参数比较好理解，即连接超时时间，第二个参数是安卓机设备号，一般来说，使用`adb devices`即可看到设备号，当然不同的模拟器显示设备号的方式也不一样，大家可以自行Google下。

##### 页面跳转
adbDevice.startActivity(null, null, null, null, categories, new HashMap<String, Object>(), activity, 0);
其余参数暂时先不用考虑，现在只需关注倒数第二个参数activity，在ANDROID app中，一般一个页面对应一个activity，所以找到对应的activity，就可以启动相应的页面，查找activity比较简单的方法是使用tools/hierarchyviewer.bat，它可以显示当前页面的activity，如图：
![activity](http://od6ojrbik.bkt.clouddn.com/images/blog/2017-02-27-1.png)
当前页面的activity为**com.sankuai.meituan/com.sankuai.meituan.activity.MainActivity**，其中斜杠前面的部分为此APP的packageName。

##### 事件
MonkeyRunner同样提供了点击事件，输入事件等API
* 根据坐标点击：
adbDevice.touch(x, y, TouchPressType.DOWN_AND_UP);
屏幕分辨率不同，相同页面元素的坐标也不一样，可以通过设备自带的坐标功能查看元素坐标。
* 根据元素ID点击：
easyDevice.touch(By.id(id), TouchPressType.DOWN_AND_UP);
此方法是monkeyrunner.jar包提供，查找页面元素ID同样使用hierarchyviewer查看器，这里需要注意的是，貌似一个页面多个元素可以共用一个ID的，所以使用此API时**<font color="red">需谨慎</font>**
![元素查看器](http://od6ojrbik.bkt.clouddn.com/images/blog/2017-02-27-2.png)
* 输入文本：
adbDevice.type(text);

#### shell命令
adbDevice.shell(cmd);
具体的adb shell命令大家可以Google下

### 问题及解决方案
大部分的App自动化测试工具可能都存在如下问题：
在页面A点击页面元素E直至弹出一个新的页面B，这个过程中间会有一个等待时间，我们无法判断页面B什么时候出现，所以大部分我们的做法是经过测试得出一个相对可靠的等待时间，然而这样的会出现两个问题：
* 超过等待时间仍然没有出现页面B（BUG）
* 页面B弹出后仍然没有超过等待时间（时间浪费）

针对MonkeyRunner来说，现在做了如下优化：
点击E后轮询判断是否出现了页面页面B，如果出现了，那么则可以进行下一步，未出现，继续等待，代码如下：
```java

/**
 * 等待直到进入希望跳入的VIEW
 * @param viewName
 * @param condition
 * @throws Exception
 */
public void waitForGetIntoViewWithCondition(String viewName ,Condition condition) throws Exception
    {
        //判断当前view是否为希望转入的view
        int cnt = (int)(VIEW_TIMEOUT_MILL_SECS / 1000);
        int thisCnt = 0;
        while (true)
        {
            HierarchyViewer viewer = getHierarchyViewer();
            if (null != viewer)
            {
                String windowName = viewer.getFocusedWindowName();
                if (viewName.equals(windowName))
                {
                    return;
                }
              
                //判断viewer是否符合condition
                if (null != condition)
                {
                    boolean call = condition.callback();
                    if (call)
                    {
                        return;
                    }
                }
            }
            CommonUtil.sleep(1000);
            thisCnt ++;
            if (thisCnt >= cnt)
            {
                break;
            }
        }
        throw new Exception("get into viewer:" + viewName + " fail!");
    }
/**
 * 进入HierarchyViewer
 * @return
 */
public HierarchyViewer getHierarchyViewer()
    {
        int cnt = (int)(VIEW_TIMEOUT_MILL_SECS / 1000);
        int thisCnt = 0;
        
        
        while(null == DeviceBridge.loadWindowData(new Window(new ViewServerDevice(adbDevice.getDevice()), "", -1)))
        {
            CommonUtil.sleep(1000);
            thisCnt ++;
            if (thisCnt >= cnt)
            {
                return null;
            }
        }
        return adbDevice.getHierarchyViewer();
    }
```
Condition为自定义的条件，因为有时候点击一个元素后进入的可能是不同的页面，所以这里增加了一个Conditon条件的判断，同时有最大时长限制，通过这样的方式，可以有效的保证在最短的时间内进入页面B


### DEMO
最后附上一个简单的Deom供大家测试使用
```java
public class MonkeyRunnerTest {
	
    public static void main()
    {
        AdbBackend adbBackend = new AdbBackend();
        AdbChimpDevice device = (AdbChimpDevice)adbBackend.waitForConnection(30000, "127.0.0.1:62001");
        
        //权限设置
        Collection<String> categories = new ArrayList<String>();

        //启动美团s
        //这个地方可以使用优化过的waitForGetIntoViewWithCondition方法
        String activity = "com.sankuai.meituan/com.sankuai.meituan.activity.MainActivity";
        device.startActivity(null, null, null, null, categories, new HashMap<String, Object>(), activity, 0);
        
        //sleep 3s
        CommonUtil.sleep(3000);

        //根据坐标点击美食选项（分辨率768*1280）
        device.touch(96, 360, TouchPressType.DOWN);

        //sleep 3s
        CommonUtil.sleep(3000);
        
        //关闭美团
        device.shell("am force-stop com.sankuai.meituan");
        
    }
}
```


