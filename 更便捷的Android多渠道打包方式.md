>本文先回顾了以往流行的多渠道打包方式，随后引入的mcxiaoke的packer-ng-plugin项目，介绍该项目在实际应用（配合友盟统计）中如何解决更方便的Android多渠道打包问题

* 多渠道打包方案解析 
* 实际应用集成

### 多渠道打包方案解析
android应用市场多种多样，应用宝、小米市场、豌豆荚...为了监督每个市场我们的应用下载和推广情况，对发布在每个市场上的apk打上烙印是必须的一步，这就是多渠道apk的问题，“渠道”就是给apk打上的烙印。
同时友盟统计可以帮我们统计渠道数据（Channel），方便产品经理对数据分析后做下一步产品决策。

目前多渠道打包方式大致有：
1. gradle自带的productFlavor方式，见我之前的[博客](http://www.cnblogs.com/soaringEveryday/p/5368540.html)
2. apktool重签名重打包
3. 在apk文件中的META-INF文件夹中写入以渠道号命名的空文件方式（美团）

第三种是比较快的方式，据说900多个渠道不到一分钟就能打完，[参考](http://tech.meituan.com/mt-apk-packaging.html)。但是缺点也是有的，你需要维护一个python脚本，为每一种渠道写入一个以渠道名命名的空文件。

本文将介绍的packer-ng-plugin的思路其实是有点类似的，由于apk就是一个zip文件，zip文件尾部有一个部分可以作为zip文件的注释，正确修改这一部分不会对ZIP文件造成破坏，利用这个字段，我们可以添加一些自定义的数据，PackerNg项目就是在这里添加和读取渠道信息。同时提供了读取渠道信息的接口。

### 实际应用集成
修改项目gradle，加入
```gradle
buildscript {
    ......
    dependencies{
    // add packer-ng
        classpath 'com.mcxiaoke.gradle:packer-ng:1.0.5'
    }
} 
```

修改moudle级别gradle，加入
```gradle
apply plugin: 'packer' 

dependencies {
    // add packer-helper
    compile 'com.mcxiaoke.gradle:packer-helper:1.0.5'
} 
```
>注意：packer-ng 和 packer-helper 的版本号需要保持一致

在你的项目根目录中加入渠道列表文件，比如文件名是market.txt，内容是
```
YingYongBao
XiaoMi
WanDouJia
Baidu
Qihoo
GooglePlay
...
```
就是每一行即一个渠道号

再在你的项目根目录加入一个bat脚本（windows），比如叫做`build.bat`，内容写上（即一个命令）
``` powershell
gradle -Pmarket=markets.txt clean apkRelease
```

大功告成，以后每次打渠道包，只要进入你的根目录，双击这bat脚本，packer-ng-plugin就开始自动帮你根据market.txt构建每一个渠道包；或者直接在android studio的`Terminal`中执行build.bat亦可。

观察控制台的输出，packer-ng最终是为每一个渠道包的打制形成了一个个的gradle task：
``` powershell
...
:app:apkRelease processed apk for XiaoMi (4)
:app:apkRelease processed apk for WanDouJia (5)
:app:apkRelease processed apk for Baidu (6)
:app:apkRelease processed apk for Qihoo (7)
:app:apkRelease processed apk for GooglePlay (8)
:app:apkRelease all 8 apks saved to D:\workspace\shine\build\archives
:app:apkRelease PackerNg: Market Packaging Successful!
BUILD SUCCESSFUL
Total time: 1 mins 23.269 secs 
```
根据这个输出，他一共生成了8个渠道包，所有的apk输出在了项目根目录的build/archives文件夹中

packer-ng-plugin也提供了一些自定义配置，比如输入的apk的命名方式，具体参考[插件配置说明](https://github.com/mcxiaoke/packer-ng-plugin#%E6%8F%92%E4%BB%B6%E9%85%8D%E7%BD%AE%E8%AF%B4%E6%98%8E%E5%8F%AF%E9%80%89)，同时提供了java和python的命令行脚本，供集成到持续集成环境中，具体参考[命令行打包脚本](https://github.com/mcxiaoke/packer-ng-plugin#%E5%91%BD%E4%BB%A4%E8%A1%8C%E6%89%93%E5%8C%85%E8%84%9A%E6%9C%AC)。

最后需要提的一点就是如何让**友盟统计**知道目前的apk是哪个渠道。首先你需要删除之前的productFlavor或者manifest的占位符的方式的代码，删除AndroidManifest中友盟的渠道Channel的META-Data的配置。
然后在app入口（Application）的onCreate中加入下列代码：
```java
final String market = PackerNg.getMarket(this,"defaul_channel);
AnalyticsConfig.setChannel(market); //AnalyticsConfig是友盟的代码方式设置渠道类
```
