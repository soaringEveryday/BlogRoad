## 用RxJava处理复杂表单验证问题

>无论是简单的登录页面，还是复杂的订单提交页面，表单的前端验证（比如登录名和密码都符合基本要求才能点亮登录按钮）都是必不可少的步骤。本文展示了如何用RxJava来方便的处理表单提交前的验证问题，例子采用了Android上的一个简单的登录页面
### 内容提要
* 传统的验证方式
* combineLatest操作符
* 用combineLatest处理表单验证
* combineLatest和zip的区别

本文中所演示的例子sample代码位于[RxAndroidDemo](https://github.com/soaringEveryday/RxAndroidDemo)，参见[loginActivity](https://github.com/soaringEveryday/RxAndroidDemo/blob/master/app/src/main/java/com/jason/rxjavademo/activity/LoginActivity.java)这个文件

![RxAndroidDemo-LoginActivity](http://7xtefp.com1.z0.glb.clouddn.com/blog_RxAndroidDemo_LoginActivity_1.png)
### 传统的验证方式
这里我们用最简单的例子来说明，如上图，一个email输入和一个password输入，下方是一个登录的按钮。只有当email输入框内容含有@字符，password输入框内容大于4个，才点亮下方的按钮。

首先你用EditText还是继承自EditText的控件，一般来说监听它的内容，都是用addTextChangedListener。但是如何显然登录按钮的enable与否是同时要判断email和password的，两个都成立才可点亮。所以我们在email的TextWatcher中除了要判断email是否符合条件以外，还要同时判断password是否符合条件，这样以来就容易造成多重判断。
试想如果你在提交一个订单的表单，上面是十几个输入框，每个输入的内容都同时符合条件才可以点亮“提交”按钮，这是多么痛苦的事情————每一个输入框的改变都要同时再判断其他十几个输入框内容是否符合（实际上此时其他十几个输入框没变化）

### combineLatest操作符
combineLatest是RxJava本身提供的一个常用的操作符，它接受两个或以上的Observable和一个FuncX闭包。当传入的Observable中任意的一个发射数据时，combineLatest将每个Observable的最近值(Lastest)联合起来（combine）传给FuncX闭包进行处理。要点在于
1. combineLatest是会存储每个Observable的最近的值的
2. 任意一个Observable发射新值时都会触发操作->“combine all the Observable's lastest value together and send to Function”
![combineLatest操作符](http://7xtefp.com1.z0.glb.clouddn.com/blog_RxAndroidDemo_LoginActivity_2.png)

### 用combineLatest处理表单验证
首先我们写上email和password的验证方法，一个需要含有@字符，一个要求字符数超过4个：
```java
private boolean isEmailValid(String email) {
        //TODO: Replace this with your own logic
        return email.contains("@");
    }

private boolean isPasswordValid(String password) {
    //TODO: Replace this with your own logic
    return password.length() > 4;
}
```

随后，我们针对email和password分别创建Observable，发射的值即为各自edittext的变化的内容，而call回调方法的返回值是textWatcher中afterTextChanged方法的传入参数：
```java
Observable<String> ObservableEmail = Observable.create(new Observable.OnSubscribe<String>() {

            @Override
            public void call(final Subscriber<? super String> subscriber) {
                mEmailView.addTextChangedListener(new TextWatcher() {
                    @Override
                    public void beforeTextChanged(CharSequence s, int start, int count, int after) {

                    }

                    @Override
                    public void onTextChanged(CharSequence s, int start, int before, int count) {

                    }

                    @Override
                    public void afterTextChanged(Editable s) {
                        subscriber.onNext(s.toString());
                    }
                });
            }
        });

Observable<String> ObservablePassword = Observable.create(new Observable.OnSubscribe<String>() {

    @Override
    public void call(final Subscriber<? super String> subscriber) {
        mPasswordView.addTextChangedListener(new TextWatcher() {
            @Override
            public void beforeTextChanged(CharSequence s, int start, int count, int after) {

            }

            @Override
            public void onTextChanged(CharSequence s, int start, int before, int count) {

            }

            @Override
            public void afterTextChanged(Editable s) {
                subscriber.onNext(s.toString());
            }
        });
    }
});
```

最后，用combineLastest将ObservableEmail和ObservablePassword联合起来进行验证：
```java
Observable.combineLatest(ObservableEmail, ObservablePassword, new Func2<String, String, Boolean>() {
            @Override
            public Boolean call(String email, String password) {
                return isEmailValid(email) && isPasswordValid(password);
            }
        }).subscribe(new Subscriber<Boolean>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(Boolean verify) {
                if (verify) {
                    mEmailSignInButton.setEnabled(true);
                } else {
                    mEmailSignInButton.setEnabled(false);
                }
            }
        });
```
onNext中的verify就是经过combineLastest对两者验证后组合的结果。
> 参见LoginActivity的bindView()方法

这里，即使表单非常复杂，实际上你需要扩展的话就很容易了：

1. 对每个EditText封装一个Observable
2. 改写这句话，加入新的逻辑：
```java
return isEmailValid(email) && isPasswordValid(password);
```

觉得为每一个EditText封装一个Observable要写很多重复代码？放心，Jake Wharton大神早已经想到，RxBinding中的RxTextView就可以解决这个问题：
```java
Observable<CharSequence> ObservableEmail = RxTextView.textChanges(mEmailView);
Observable<CharSequence> ObservablePassword = RxTextView.textChanges(mPasswordView);

Observable.combineLatest(ObservableEmail, ObservablePassword, new Func2<CharSequence, CharSequence, Boolean>() {
    @Override
    public Boolean call(CharSequence email, CharSequence password) {
        return isEmailValid(email.toString()) && isPasswordValid(password.toString());
    }
}).subscribe(new Subscriber<Boolean>() {
    @Override
    public void onCompleted() {

    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(Boolean verify) {
        if (verify) {
            mEmailSignInButton.setEnabled(true);
        } else {
            mEmailSignInButton.setEnabled(false);
        }
    }
});
```
> 参见LoginActivity的bindViewByRxBinding()方法

### combineLatest和zip的区别
zip是和combineLatest有点像的一个操作符，接受的参数也是两个或多个Observable和一个闭包。但是区别在于：

1. zip是严格按照顺序来组合每个Observable，比如ObservableA的第一个数据和ObservableB的第一个数据组合在一起发射给FuncX来处理，两者的第N个数据组合在一起发射给FuncX来处理，以此类推
2. zip并不是任意一个Observable发射数据了就触发闭包处理，而是等待每个Observable的第N个数据都发射齐全了才触发

zip一般用于整合多方按照顺序排列的数据。
![zip](http://7xtefp.com1.z0.glb.clouddn.com/blog_RxAndroidDemo_LoginActivity_3.png)
