---
layout: post
title: Android MVP 试水
date: 2016-10-26 18:22:00 +0800
description:  # Add post description (optional)
tags: Android
---
还记得一年前，在上一家公司的时候，领导准备接一个案子，
客户那边给了一份开发规范的文档，上面明确的写着要采用MVP模式进行开发。
一开始看到这个模式时候，一脸懵逼，什么是MVP？
不懂，问一下同事，也没有人能说清楚，无奈那就百度吧。

>MVP(Model-View-Presenter) 是从经典的模式MVC演变而来，  
它们的基本思想有相通的地方：  
Controller/Presenter负责逻辑的处理，  
Model提供数据，  
View负责显示

好简单粗暴的说明啊，还是一脸懵逼。

![7B9465C2FD1827CD2DA2487906B5120E.gif](http://upload-images.jianshu.io/upload_images/2825667-99cd26030d66b069.gif?imageMogr2/auto-orient/strip)

后来，不知道为什么案子也没有接，就这样不了了之了。

最近发生了很多事，从上一家公司离职，与朋友准备搞公司，
搞了差不多2个月，到现在的从团队退出。然后准备找工作。。。。

在这期间搞项目的时候，就抽空研究了一下MVP模式，
试着用它进行开发。因为只是一个项目，涉及的还不深，
所以叫**试水**。记录一下。

网上关于MVP的介绍、讲解、示例以及开源的项目很多，
我这里就不废话了，如果现在还有人不了解什么是MVP，
那就百度去吧。
我这里参考Google的源码**[todo-mvp](https://github.com/googlesamples/android-architecture/tree/todo-mvp/)**来说。

先看一下目录结构：  
![屏幕快照 2016-10-26 12.32.32.png](http://upload-images.jianshu.io/upload_images/2825667-3d5f22724735f67c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不要问我为什么我截图的字体颜色是蓝色的，
我不会告诉你我是用的
**[octotree](https://github.com/buunguyen/octotree)**
浏览器插件。
这里有两个Base文件：BaseView、BasePresenter，
好像和VP有关，先看一下源码：

```
package com.example.android.architecture.blueprints.todoapp;

public interface BasePresenter {

void start();

}


package com.example.android.architecture.blueprints.todoapp;

public interface BaseView<T> {

void setPresenter(T presenter);

}

```

What ？这是什么鬼？两个接口？干吗用的？不知道，不明觉厉。
不管了，反正一个对应的V，一个对应的是P就是了。
好吧，你们估计再说我这不废话了。这里看不出什么东西，
那就从程序的入口看吧，从**AndroidManifest.xml**
中找到程序的入口是tasks/TasksActivity。

```
<activity
  android:name="com.example.android.architecture.blueprints.todoapp.tasks.TasksActivity"
  android:theme="@style/AppTheme.OverlapSystemBar">

  <intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
   </intent-filter>

 </activity>
```
打开tasks包，看到如下几个文件。

![屏幕快照 2016-10-26 12.51.02.png](http://upload-images.jianshu.io/upload_images/2825667-8c961017a90c0c58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的习惯是先不看里面的内容，先看文件名字，
大致了解每个文件是干嘛用的，这样有助于对整体进行把控，
所以这里就体现了命名规范的重要性，关于命名规范，
可以百度，也可以参考我的另外一篇文章：
**[Android 开发规范(个人版)](http://www.jianshu.com/p/45ffeff3d6b5)**。
哎呀，又扯远了，继续回来看代码。

第一个文件，应该是个自定义的布局，好像没什么太大的关系。  
第二个主程序的入口，没啥说的。  
第三个Contract (契约)，谁和谁的，不懂，先不管。  
第四个Filter Type(过滤器类型),应该是一些类型的定义，好像关系也不大，先不管。  
第五个Fragment,不说了  
第六个Persenter,这个有关系，而且还很大，那就先看一下它吧。  

```
public class TasksPresenter implements TasksContract.Presenter {
   ....
  private final TasksContract.View mTasksView; 
  public TasksPresenter(@NonNull TasksRepository   tasksRepository, @NonNull    
  
  TasksContract.View tasksView) {
  ...
  mTasksView = checkNotNull(tasksView, "tasksView cannot be null!"); 
  mTasksView.setPresenter(this);
  }

  @Override
  public void start() {
  ...
  }
}
```

省略了一下不必要的代码，这里可以看到几个关键点：  
1、TasksPresenter本身实现了TasksContract.Presenter；  
2、构造函数里面需要传入一个TasksContract.View；  
3、拿到这个tasksView后赋值给了mTasksView，
    并把自己通过mTasksView.setPresenter(this)方法传递出去。

到这里算是有点眉目了。知道了P和V是如何绑定在一起的了。  
P绑定V：通过实例化是传入V;  
V绑定P: 通过v.setPresenter(P);  

但如何使用V呢？继续往下，这里用到了TasksContract这个契约，跟踪一下代码看一下。
```
package com.example.android.architecture.blueprints.todoapp.tasks;
import android.support.annotation.NonNull;
import com.example.android.architecture.blueprints.todoapp.BaseView;
import com.example.android.architecture.blueprints.todoapp.data.Task;
import com.example.android.architecture.blueprints.todoapp.BasePresenter;
import java.util.List;

/**
 * 这指定 view 和 presenter 之间的 contract。
 * This specifies the contract between the view and the presenter.
 */

public interface TasksContract {
  interface View extends BaseView<Presenter> {
      void setLoadingIndicator(boolean active);
      void showTasks(List<Task> tasks);
      void showAddTask();
      void showTaskDetailsUi(String taskId);
      void showTaskMarkedComplete();
      void showTaskMarkedActive();
      void showCompletedTasksCleared();
      void showLoadingTasksError();
      void showNoTasks();
      void showActiveFilterLabel();
      void showCompletedFilterLabel();
      void showAllFilterLabel();
      void showNoActiveTasks();
      void showNoCompletedTasks();
      void showSuccessfullySavedMessage();
      boolean isActive();
      void showFilteringPopUpMenu();
  }

  interface Presenter extends BasePresenter {
      void result(int requestCode, int resultCode);
      void loadTasks(boolean forceUpdate);
      void addNewTask();
      void openTaskDetails(@NonNull Task requestedTask);
      void completeTask(@NonNull Task completedTask);
      void activateTask(@NonNull Task activeTask);
      void clearCompletedTasks();
      void setFiltering(TasksFilterType requestType);
      TasksFilterType getFiltering();
  }
}
```
看到这里就有点意思了，契约类里面主要做了两件事，
* 定义了一个继承自BaseView的接口(View)， 并声明需要实现的方法。
* 定义一个继承自BasePresenter的接口并继承(Presenter)，并声明需要实现的方法。

其实也可以说是一件事，就是声明一些接口。

哦，这下知道Contract是干吗用的了，
就是把V、P的接口写到同一个文件里面啊，好像也并么有什么高大上的东西啊？
那我把这个文件分成两个文件写，应该也可以吧？我认为是可以的。
但是又基于Contract的含义即：契约，就是把View和Presenter绑定到一起：

```
interface Presenter extends BasePresenter {}
interface View extends BaseView<Presenter> {}
```

这样还是按照官方的来，用一个文件来写好了。

Presenter找到了，Contract也知道是干吗用的了。那么View呢，
从文件名已经找不到了，那就看继续看代码吧。
从TasksActivity看起，
首先我们知道TasksPresenter构造函数里面有一个TasksContract.View的参数，
那么就找这个参数传的什么。

```
public class TasksActivity extends AppCompatActivity { 
    ....
    private TasksPresenter mTasksPresenter;
 
   @Override 
   protected void onCreate(Bundle savedInstanceState) {
     super.onCreate(savedInstanceState);
     setContentView(R.layout.tasks_act);
     ....
 
     TasksFragment tasksFragment =
     (TasksFragment) getSupportFragmentManager().findFragmentById(R.id.contentFrame);
     if (tasksFragment == null) {
     // Create the fragment 
     tasksFragment = TasksFragment.newInstance();
    }
    .....
 
    // Create the presenter 
    //这个传入的是 tasksFragment
    mTasksPresenter = new TasksPresenter(
    Injection.provideTasksRepository(getApplicationContext()), tasksFragment);
 
   } 
  .......
} 
```

由上面的代码可知，TasksPresenter 传入了一个TasksFragment的对象，
那这样的TasksFragment就应该是所谓的View了，跟踪进入TasksFragment。


```
public class TasksFragment extends Fragment implements TasksContract.View {
private TasksContract.Presenter mPresenter;

......
public TasksFragment() {
// Requires empty public constructor
}

public static TasksFragment newInstance() {
return new TasksFragment();
}

......

@Override
public void onResume() {
super.onResume();
mPresenter.start();
}

@Override
public void setPresenter(@NonNull TasksContract.Presenter presenter) {
mPresenter = checkNotNull(presenter);
}
....

@Override
public View onCreateView(LayoutInflater inflater, ViewGroup container,

Bundle savedInstanceState) {
.....
}
.......

}
```

果然，TasksFragment 实现了TasksContract.View，
就是所谓的View。他的核心点在于：  
1、实现了TasksContract.View；  
2、重写setPresenter方法，接收传递过来的presenter。

这样之后，就可以通过presenter.xxxxx()的方式来调用presenter里面定义的一些方法，
而presenter里面定义的方法主要执行耗时操作或者一些数据处理等等，
等到presenter里面的函数执行完毕之后，
在通过mTasksView.xxx()的方式回调给TasksFragment，
TasksFragment再进行页面的改变。

官方给的Demo就看到这里吧，因为关于MVP核心的东西差不多就看完了，
或许还有更多的东西我没有发掘。

根据官方Demo，我这里总结了一下实现MVP模式的步骤：  
1、定义BaseView、BasePresenter。可以参考官方示例。  
2、定义契约类，在里面定义两个接口，举个登录的例子：  

```
public interface LoginContract {
   interface Presenter extends BasePresenter {
      /**
       * 登录
       */
      void login();
   }
   interface View extends BaseView<Presenter> {
      /** 
        * 返回登录成功
       */
      void loginSuccess();
      void loginFailed(String errorMessage);
   }
}
```

3、定义一个实现契约类中Presenter接口的类，用于实现逻辑代码，并把处理结果返回。

例如：

```
public class LoginPresenter implements LoginContract.Presenter {
   private LoginContract.View view;
   public LoginPresenter(LoginContract.View view) {
      this.view = view;
      view.setPresenter(this);
   }
   /**
    * 登录
    */
   @Override
   public void login() {
      String useName = view.getUserName();
      String pwd = view.getPwd();
      Map<String, String> params = new HashMap<String, String>();
      params.put("phone", useName);
      params.put("password", pwd);
      AuthRequestUtil.doLogin(params, User.class, new ResponseCallBack<User>() {
         @Override
         public void onSuccess(User data) {
            super.onSuccess(data);
            saveLoginInfo(data);
            //返回登录成功
            view.loginSuccess();
         }
         @Override
         public void onFailure(ServiceException e) {
            super.onFailure(e);
            //返回登录失败
            view.loginFailed(e.getMessage());
         }
      });
   }
}
```

4、在Activity 或者Fragment中实现契约类中的View接口。

要实现简单的MVP,差不多就这4步。接触的时间也不长，
中间有可能会出现一些纰漏或者错误，如果有这方面的牛人在看到这篇文章的时候，
希望能给出宝贵意见。这里先说声谢谢。

关于MVP，还有很多东西，我看到还有关Presenter生命周期的相关文章，
还没有仔细研究。这里先记一下。等有时间在仔细研究一下。