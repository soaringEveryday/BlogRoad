## 用Dart&Henson玩转Activity跳转

>Extra是Android标准的组件之间（Activity/Fragment/Service等）传递数据的方式。本文介绍了开源项目Dart的使用，它优雅的处理了组件间跳转和数据传递
### 内容提要
* 传统的方式
* Dart & Henson
* 小改进建议

本文中所演示的例子sample代码位于[DartHensonSample](https://github.com/soaringEveryday/DartHensonSample)

### 传统的方式
会Android的人都会这个的，这里就简单说下，一般流程是
1. 定好传递数据对应的Key
2. SecondActivity设置好属性，通过`getIntent().getXXXExtra()`获得数据
3. FirstActivity通过`intent.putExtra()`设置数据，并执行`startActivity()` 

具体这里就不在贴上代码了，主要讲讲这里会导致的一些问题。
1. 首先这个Key要保持维护状态，有时候前后两个Activity不是同一个写的，Key的使用交流会出现误解或者指定错误
2. SecondActivity中有些传入的数据可能是`必须的`，但是对于这个FirstActivity的作者可并不知道啊

好在[Dart](https://github.com/f2prateek/dart)这个开源项目顺利的处理了上述等问题，而且处理的非常优雅。

### Dart & Henson
Dart的原理和ButterKnife类似，都是通过注解处理器在编译阶段生成一些代码。所以你再写好一些注解后，必须要构建一些项目才能生成一些后面你所需要的代码（后面会详细说明）。

首先引入android-apt，在项目gradle中加入插件
```gradle
buildscript {
  repositories {
    mavenCentral()
   }
  dependencies {
    classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
  }
}
```
再在build.gradle引用
```gradle
apply plugin: 'android-apt'
```

引入Dart & Henson(Henson实际是Dart项目的子项目)
```gradle
compile 'com.f2prateek.dart:dart:2.0.0'
provided 'com.f2prateek.dart:dart-processor:2.0.0'
compile 'com.f2prateek.dart:henson:2.0.0'
provided 'com.f2prateek.dart:henson-processor:2.0.0'
```

这里我们假设要从MainActivity跳转到DetailActivity，DetailActivity中要接受三个参数分别是`String name`,`int age`和`User user`。这里的user是一个自定义类型，我们知道要想传递数据，必须序列化，文的例子中引入了[Parceler](https://github.com/johncarl81/parceler)项目通过同样一个注解`@Parcel`自动在编译器声场Parcelable的繁杂的代码：[User.java](https://github.com/soaringEveryday/DartHensonSample/blob/master/app/src/main/java/com/jason/darthensonsample/User.java)

DetailActivity需要接受上述三个参数，仅仅通过`@InjectExtra`注解即可，然后在`onCreate`中执行`Dart.inject(this)`，详细的代码为：
```java
public class DetailActivity extends AppCompatActivity {
    @InjectExtra
    String name = "default name";

    @InjectExtra
    int age = 0;

    @Nullable
    @InjectExtra
    User user;

    @BindView(R.id.tvName)
    TextView tvName;

    @BindView(R.id.tvAge)
    TextView tvAge;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_detail);
        ButterKnife.bind(this);
        Dart.inject(this);
        initView();
    }

    private void initView() {
        // 使用name,age,user
    }
}
```

`String name = "default name";`这句话给了name一个默认值，但是当`Dart.inject`执行后会被传递过来的数据覆盖。
`@Nullable`加在了user上说明这个数据可以不用传递。接受端就这么多代码，下面让我们看看发送端如果发送数据，如何跳转到DetailActivity。

注意编写上述代码后，我们要先编译下项目，编译好后Henson会通过注解处理器生成可以跳转到DetailActivity的DSL（领域特定语言）段，方便其他组件对DetailActivity的跳转，我们在MainActivity上只要写上下列代码，即可完成界面跳转和数据传递：
```java
User user = new User();
user.setAge(Integer.parseInt(age.getText().toString()));
user.setName(name.getText().toString());
startActivity(
        Henson.with(this)
                .gotoDetailActivity()
                .age(27)
                .name("jason")
                .user(user)
                .build()
);
```
当你写完`Henson.with(this)`后代码提示会自动弹出`.gotoDetailActivity()`，Henson帮助你提示你DetailActivity是可以被跳转的；随后你继续写下`.gotoDetailActivity()`后，又自动弹出`.age()`方法，提示你传入一个int类型给age，写好后，又自动弹出`.name()`方法，以此类推，最后以一个`build()`收场。
注意，这里你可以不写`.user()`，因为在DetailActivity中我们指定它是nullable的，可以不传。但是`.age()`和`.name()`都是会强制弹出让你填写数据的。

至此，Henson&Dart的基本介绍结束。

### 小改进建议
Dart还有一个`@HensonNavigable`注解，它标注在将会被跳转到的Activity的classname上，说明这个activity可以被跳转，会自动生成`.gotoXXXActivity()`这样的代码，但是它不能在这个类中存在@InjectExtra注解，这里总觉得这个注解有些多余。

第二，和Dart项目的作者提了issue问了关于参数指定的顺序问题，上述例子中我们传递数据的代码是：
```
 Henson.with(this)
       .gotoDetailActivity()
       .age(Integer.parseInt(age.getText().toString()))
       .name(name.getText().toString())
       .build()
```
但是如果我偏向要对调name和age呢？发现不行~如果你写完`.gotoDetailActivity()`后发现只有`.age()`的提示，却没有`.name()`的，即强制要求你先传递age。这里作者说明后才知道他们是按照字幕顺序来排数据传入的，目前没有更好的方法。

欢迎你开心的使用Dart & Henson!
