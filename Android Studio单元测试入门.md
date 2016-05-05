## Android Studio单元测试入门

>通常在开发Android app的时候经常会写一些小函数并验证它是否运行正确，通常做法我们是把这个函数放到某个界面（Activity上）执行一下，运行整个工程跑一下app，通过打log的方式来验证。不过，现在我们活用Android Studio自带的单元测试功能即可免除这种麻烦，直接写测试用例像Junit那样来验证你的小函数
### 内容提要
* 配置
* 编写Java测试用例
* 编写Android测试用例
* 其他测试基类


### 配置
在Android Studio中进行单元测试并不需要什么插件或者过多的配置，Android Studio本身就集成了测试环境，无论是单纯的java代码单元测试还是依赖Android SDK的Android代码单元测试，都能得心应手。
首先在你的gradle中加入Junit的依赖，注意这里的依赖方式是测试期间的依赖（testCompile）：
```groovy
dependencies {
    testCompile 'junit:junit:4.12'
}
```
再在项目的app/src下面和main文件夹同级的建立androidTest和test目录，并且分别在各自目录下建议java/com/xxx/xxx类似的和主工程一致的包名目录，建立好后，你的项目在Android Studio的Project中应该是这样的：
![project structure](http://7xtefp.com2.z0.glb.clouddn.com/1.jpg)

### 编写Java测试用例
如果所写的测试代码没有使用android sdk（android.***下的代码），那么可以在test目录下新建，本例中即为ExampleUnitTest，例子中测试了一个RxJava的Observable的发射后被消费的结果。
注意测试用例即一个public void的方法，并且加上@Test注解，这是Junit的标准用法
```java
package com.jason.rxjavademo;
import org.junit.Test;
import rx.Observer;
import rx.subjects.PublishSubject;

public class ExampleUnitTest {

    @Test
    public void testPublishSubject() {
        PublishSubject<String> stringPublishSubject = PublishSubject.create();
        stringPublishSubject.subscribe(new Observer<String>() {
            @Override
            public void onCompleted() {
                System.out.println("Observable completed");
            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(String s) {
                System.out.println("Observer consumed " + s);
            }
        });
        stringPublishSubject.onNext("hello world");
        stringPublishSubject.onCompleted();
    }
}
```
这时候打开Android Studio左边的Structure的面板，可以看到测试用例方法testPublishSubject
![java structure](http://7xtefp.com2.z0.glb.clouddn.com/2.jpg)

右击并运行它，测试通过，返回了正确的值
![java result](http://7xtefp.com2.z0.glb.clouddn.com/3.jpg)

注意本测试用例试用了System.out.println所以测试结果直接打印在了控制台上，如果把打印的地方换成Log.d()呢，你会发现报错：
![java error](http://7xtefp.com2.z0.glb.clouddn.com/4.jpg)

这个实际是因为你在java的Unit test中引用了Android的代码，即android.util.log.Log。所以对于测试Android代码，需要在androidTest中


### 编写Android测试用例
Android测试用例我们可以
1. 在androidTest下新建一个java类，并且继承自InstrumentationTestCase
2. 编写一个public void的方法，但是必须要是方法名以**test**打头，比如testPublishSubject，并不需要@Test注解

```java

public class TestSubject extends InstrumentationTestCase {
    private static final String LOG_TAG = "test";

    public void testPublishSubject() {
        PublishSubject<String> stringPublishSubject = PublishSubject.create();
        stringPublishSubject.subscribe(new Observer<String>() {
            @Override
            public void onCompleted() {
                Log.d(LOG_TAG, "Observable completed");
            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(String s) {
                Log.d(LOG_TAG, "Observer consumed " + s);
            }
        });
        stringPublishSubject.onNext("hello world");
        stringPublishSubject.onCompleted();
    }
}
```
本例运行后，会在Android Monitor中以test这个LOGTAG打出和上一节一样的Log

Android Studio也提供了测试单个Activity或者多个Activities的测试用例方法基类，比如ActivityInstrumentationTestCase2，步骤为
1. 在androidTest下新建一个java类，并且继承自ActivityInstrumentationTestCase2，传入需要测试的Activity的类到泛型
2. 复写setUp方法，获得Context
3. 编写一个public void的方法，但是必须要是方法名以**test**打头，比如testStart，并不需要@Test注解

```java

public class TestActivity extends ActivityInstrumentationTestCase2<MainActivity> {

    private Context ctx;

    public TestActivity() {
        super(MainActivity.class);
    }

    @Override
    protected void setUp() throws Exception {
        super.setUp();
        ctx = getActivity().getApplicationContext();
    }

    public void testStart() {
        Intent intent = new Intent(ctx, MainActivity.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        ctx.startActivity(intent);
    }
}
```
运行这个测试用例，你会发现模拟器上单独启动了这个Activity

### 其他测试基类
除了InstrumentationTestCase和ActivityInstrumentationTestCase2外，android.test还提供了很多别的测试基类，比如
* ActivityUnitTestCase
* MockApplication
* ServiceTestCase