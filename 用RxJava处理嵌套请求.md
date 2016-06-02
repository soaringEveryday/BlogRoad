## 用RxJava处理嵌套请求

>互联网应用开发中由于请求网络数据频繁，往往后面一个请求的参数是前面一个请求的结果，于是经常需要在前面一个请求的响应中去发送第二个请求，从而造成“请求嵌套”的问题。如果层次比较多，代码可读性和效率都是问题。本文首先从感性上介绍下RxJava，然后讲解如何通过RxJava中的flatMap操作符来处理“嵌套请求”的问题
### 内容提要
* RxJava简单介绍
* 嵌套请求举例
* 运用flatMap
* map和flatMap
* RxJava与Retrofit配合解决嵌套请求

### RxJava简单介绍
这里并不打算详细介绍RxJava的用法和原理，这方面的文章已经很多了。这里仅仅阐述本人对于RxJava的感性上的理解。先上一个图：
![Rxjava overview](http://7xtefp.com1.z0.glb.clouddn.com/rxjava_overview.jpg)
我们都知道RxJava是基于观察者模式的，但是和传统的观察者模式又有很大的区别。传统的观察者模式更注重订阅和发布这个动作，而RxJava的重点在于数据的“流动”。
如果我们把RxJava中的Observable看做一个盒子，那么Observable就是把数据或者事件给装进了这个易于拿取的盒子里面，让订阅者（或者下一级别的盒子）可以拿到而处理。这样一来，原来静态的数据/事件就被流动起来了。

我们知道人类会在河流中建设大坝，其实我们可以把RxJava中的filter/map/merge等Oberservable操作符看做成数据流中的大坝，经过这个操作符的操作后，大坝数据流被过滤被合并被处理，从而灵活的对数据的流动进行管制，让最终的使用者灵活的拿到。

以上就是我对RxJava的理解，深入的用法和原理大家请自行看网上的文章。

### 嵌套请求举例
这里开始进入正题，开始举一个嵌套请求的例子。
比如我们下列接口：
1. api/students/getAll (传入班级的id获得班级的学生数组，返回值是list<Student>)
2. api/courses/getAll (传入Student的id获得这个学生所上的课程，返回值是List<Course>)

我们最终的目的是要打印班上所有同学分别所上的课程（大学，同班级每个学生选上的课不一样），按照传统Volley的做法，代码大概是这样子（Volley已经被封装过）
```java
private void getAllStudents(String id) {
        BaseRequest baseRequest = new BaseRequest();
        baseRequest.setClassId(id);
        String url = AppConfig.SERVER_URL + "api/students/getAll";

        final GsonRequest request = new GsonRequest<>(url, baseRequest, Response.class, new Response.Listener<Response>() {
            @Override
            public void onResponse(Response response) {
                if (response.getStatus() > 0) {
                    List<Student> studentList = response.getData();
                    for (Student student : studentList) {

                    }
                } else {
                    //error
                }
            }
        }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                //error
            }
        });
        MyVolley.startRequest(request);
    }

    private void getAllCourses(String id) {
        BaseRequest baseRequest = new BaseRequest();
        baseRequest.setStudentId(id);
        String url = AppConfig.SERVER_URL + "api/courses/getAll";

        final GsonRequest request = new GsonRequest<>(url, baseRequest, Response.class, new Response.Listener<Response>() {
            @Override
            public void onResponse(Response response) {
                if (response.getStatus() > 0) {
                    List<Course> courseList = response.getData();
                    for (Course course : courseList) {
                        //use
                    }
                } else {
                    //error
                }
            }
        }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                //error
            }
        });
        MyVolley.startRequest(request);
    }
```
显然第一个请求的响应中获得的数据是一个List，正对每一个List中的item要再次发送第二个请求，在第二个请求中获得最终的结果。这就是一个嵌套请求。这会有两个问题：
* 目前来看并不复杂，如果嵌套层次多了，会造成代码越来越混乱
* 写出来的重复代码太多

### 运用flatMap
现在我们可以放出RxJava大法了，flatMap是一个Observable的操作符，**接受一个Func1闭包**，这个闭包的第一个函数是待操作的上一个数据流中的数据类型，第二个是这个flatMap操作完成后返回的数据类型的被封装的Observable。说白了就是讲一个多级数列“拍扁”成了一个一级数列。
按照上面的列子，flatMap将接受student后然后获取course的这个二维过程给线性化了，变成了一个可观测的连续的数据流。
于是代码是：
```java
ConnectionBase.getApiService2()
                .getStudents(101)
                .flatMap(new Func1<Student, Observable<Course>>() {
                    @Override
                    public Observable<Course> call(Student student) {
                        return ConnectionBase.getApiService2().getAllCourse(student.getId());
                    }
                })
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<Course>() {
                    @Override
                    public void call(Course course) {
                        //use the Course
                    }
                });
```
是不是代码简洁的让你看不懂了？别急，这里面的getStutent和getAllCourse是ConnectionBase.getApiService2()的两个方法，他集成了Retrofit2用来将请求的网络数据转化成Observable，最后一节将介绍，这里先不关注。
我们所要关注的是以上代码的流程。
首先getStudent传入了班级id(101)返回了Observable<Student>，然后链式调用flatMap操作符对这个Observable<Student>进行变换处理，针对每一个发射出来的Student进行再次请求 ConnectionBase.getApiService2().getAllCourse从而返回Observable<Course>，最后对这个 ConnectionBase.getApiService2().getAllCourse进行订阅，即subscribe方法，再Action1这个闭包的回调中使用course。

flatMap的作用就是对传入的对象进行处理，返回下一级所要的对象的Observable包装。

FuncX和ActionX的区别。FuncX包装的是有返回值的方法，用于Observable的变换、组合等等；ActionX用于包装无返回值的方法，用于subscribe方法的闭包参数。Func1有两个入参，前者是原始的参数类型，后者是返回值类型；而Action1只有一个入参，就是传入的被消费的数据类型。

subscribeOn(Schedulers.io()).observeOn(AndroidScheduler.mainThread())是最常用的方式，后台读取数据，主线程更新界面。subScribeOn指在哪个线程发射数据，observeOn是指在哪里消费数据。由于最终的Course要刷新界面，必须要在主线程上调用更新view的方法，所以observeOn(AndroidScheduler.mainThread())是至关重要的。

### map和flatMap
运用flatMap的地方也是可以用map的，但是是有区别的。先看下map操作符的用法：
```java
ConnectionBase.getApiService2()
                .getStudents(101)
                .map(new Func1<Student>, Course>() {
                    @Override
                    public Course call(Student student) {
                        return conventStudentToCourse();// has problem
                    }
                })
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<Course>() {
                    @Override
                    public void call(Course course) {
                        //use the Course
                    }
                });
```
可以看到map也是**接受一个Func1闭包**，但是这个闭包的第二个参数即返回值参数类型并不是一个被包装的Observable，而是实际的原始类型，由于call的返回值是Course，所以conventStudentToCourse这里就不能用Retrofit2的方式返回一个Observable了。

所以这里是有一个问题的，对于这种嵌套的网络请求，由于接到上端数据流到处理后将结果数据放入下端数据流是一个异步的过程，而conventStudentToCourse这种直接将Student转化为Course是没法做到异步的，因为没有回调方法。那么这种情况，最好还是用flatMap并通过retrofit的方式来获取Observable。要知道，Rxjava的一个精髓就是“异步”。

那么到底map和flatMap有什么区别，或者说什么时候使用map什么时候使用flatMap呢？
flatMap() 和 map() 有一个相同点：它也是把传入的参数转化之后返回另一个对象。但需要注意，和 map() 不同的是， flatMap() 中返回的是个 Observable 对象，并且这个 Observable 对象并不是被直接发送到了 Subscriber 的回调方法中。

首先，如果你需要将一个类型的对象经过处理（非异步）直接转化成下一个类型，推荐用map，否则的话就用flatMap。
其次，如果你需要在处理中加入容错的机制（特别是你自己封装基于RxJava的网络请求框架），推荐用flatMap。
比如将一个File[] jsonFile中每个File转换成String，用map的话代码为：
```java
Observable.from(jsonFile).map(new Func1<File, String>() {
    @Override public String call(File file) {
        try {
            return new Gson().toJson(new FileReader(file), Object.class);
        } catch (FileNotFoundException e) {
            // So Exception. What to do ?
        }
        return null; // Not good :(
    }
});
```
可以看到这里在出现错误的时候直接抛出异常，这样的处理其实并不好，特别如果你自己封装框架，这个异常不大好去抓取。

如果用flatMap，由于flatMap的闭包返回值是一个Observable，所以我们可以在这个闭包的call中通过Observable.create的方式来创建Observable，而要知道create方法是可以控制数据流下端的Subscriber的，即可以调用onNext/onCompete/onError方法。如果出现异常，我们直接调用subscribe.onError即可，封装框架也很好感知。代码大致如下：
```java
Observable.from(jsonFile).flatMap(new Func1<File, Observable<String>>() {
    @Override public Observable<String> call(final File file) {
        return Observable.create(new Observable.OnSubscribe<String>() {
            @Override public void call(Subscriber<? super String> subscriber) {
                try {
                    String json = new Gson().toJson(new FileReader(file), Object.class);

                    subscriber.onNext(json);
                    subscriber.onCompleted();
                } catch (FileNotFoundException e) {
                    subscriber.onError(e);
                }
            }
        });
    }
});
```

**map操作符通常也用于处理结构化的服务端响应数据**，比如下列返回的JSON数据就是一段典型的响应数据
```json
{
    "message":"操作成功",
    "status":1,
    "data":
    {
        "noVisitCount":0,
        "planCount":0,
        "visitedCount":0
    }
}
```
在map的闭包中，我们可以先判断status进行统一的出错或者正确（返回data的内容）处理，一般来说，data的内容都是处理成一个泛型

### RxJava与Retrofit配合解决嵌套请求
这里该讨论Retrofit了。可以说Retrofit就是为了RxJava而生的。如果你的项目之前在网络请求框架用的是Volley或者自己封装Http请求和TCP/IP，而现在你看到了Retrofit这个框架后想使用起来，我可以负责任的跟你说，如果你的项目中没有使用RxJava的话，使用Retrofit和Volley是没有区别的！要用Retrofit的话，就最好或者说强烈建议也使用RxJava进行编程。

Retrofit有callback和Observable两种模式，前者就像传统的Volley一样，有successs和fail的回调方法，我们在success回调方法中处理结果；而Observable模式是将请求回来的数据由Retrofit框架自动的帮你加了一个盒子，即自动帮你装配成了含有这个数据的Observable，供你使用RxJava的操作符随意灵活的进行变换。

callback模式的Retrofit是这样建立的：
```java
retrofit = new Retrofit.Builder()
                .baseUrl(SERVER_URL)
                .addConverterFactory(GsonConverterFactory.create(gson))
                .build();
```

Observable模式是这样子建立的：
```java
retrofit2 = new Retrofit.Builder()
                .baseUrl(SERVER_URL)
                .addConverterFactory(GsonConverterFactory.create(gson))
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .build();
```
即addCallAdapterFactory这个方法在起作用，在RxJavaCallAdapterFactory的源码注释中可以看到这么一句话：
>Response wrapped body (e.g., {@code Observable<Response<User>>}) calls {@code onNext} with a {@link Response} object for all HTTP responses and calls {@code onError} with {@link IOException} for network errors

即它将返回值body为包裹上了一层**“Observable”**
