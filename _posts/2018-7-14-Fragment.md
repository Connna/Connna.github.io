---
layout: page
title: Fragment详解
date:
tags: Android   
---

- Fragment有自己的生命周期
- Fragment依赖于Activity
- Fragment通过getActivity()可以获取所在的Activity；Activityt通过FragmntManager的findFragmentById()或findFragmentByTag()获取Fragment。
- Fragment和Activity是多对多的关系

### 使用方法一：每个fragment创建一个类 ，它继承自fragment。

> **onCreateView()** ：fragment中最重要的是onCreateView()方法，重写它可以设置显示的内容；
>
> **onViewCreated()**：当view创建完成以后调用这个方法。

#### Fragment与布局文件关联

在onCreateView中，

~~~java
 public View onCreateView(LayoutInflater inflater, ViewGroup container,Bundle savedInstanceState)
 {
VIew view = inflater.inflate(R.layout.XXX ,container ,false);
//通过反射获得xml布局文件，第一个参数是Id，第二个是container、第三个是false。
return view;
 }
~~~



#### Fragment中操作控件

方法有很多，这里是在onViewCreated()中操作；（它在页面创建完成的时候被调用）

~~~java
 public void onViewCreated(@NonNull View view, Bundle savedInstanceState) {
                super.onViewCreated(view, savedInstanceState); //这个不动它
     TextView textview = View.findViewById(R.id.XXX);
         textview.setText("我是Fragment");
            }
~~~



### Fragment与Activity关联

你可以用上面的方法创建很多个Fragment，但是我们还没有把他们显示出来

#### Activity中需要做的事情

我们首先要在xml布局文件中将FragLayout放到你觉得正确的位置。Fraglayout是一个容器，你所有的Fragment都可以在里面显示。

一个fraglayout可以并且只能用于显示Fragment，但是要记住

>Fraglayout可以显示不同的Fragment，Fragment也可以显示在不同的Fraglayout上

~~~java
getFragmentManager().				//获得Fragment管理器
				beginTransaction()	//开启事务
					.add(R.id.xxxx	//使用add方法添加Fragment，它有两个参数
						，Fragment) //1、放到哪里。2、放什么东西
						  	.commitAllowingStateLoss();
									//commit一定要调用
~~~

>add方法 : 参数containerViewId一般会传Activity中某个视图容器的id。如果containerViewId传0，则这个Fragment不会被放置在一个容器中（不要认为Fragment没添加进来，只是我们添加了一个没有视图的Fragment，这个Fragment可以用来做一些类似于service的后台工作）。
>
>remove方法 : Fragment被remove后，Fragment的生命周期会一直执行完onDetach，之后Fragment的实例也会从FragmentManager中移除。



#### Fragment中getActivity()为Null的问题

先看Fragment中的几个方法：

>**onAttach(Context context)** : Fragment重建或者和Activity保持关系的时候调用这个方法；
>
>**onDetach()** :Fragment和Activity断开关系的时候运行这个方法。
>
>**onDestroy()** :Fragment被销毁的时候调用这个方法，在这里可以取消异步之类的操作

getActivity()为null的解决方法：

~~1、在onAttach(Context context)方法里面把Activity保存起来，用保存的Activity代替getActivity()。但是这样会有很多问题，~~**一般不建议这样做**。

2、在getActivity()之前判断一下是否为Null，不为Null才执行。

3、其次，在onDestroy()中把异步进程全部销毁，避免出现回调为null的问题



#### 向Fragment传递参数

Fragment不能使用有参构造，传递参数需要另辟蹊径，好在并不麻烦

~~~java
//这是一个静态方法，直接用类调用，他会返回一个Fragment的对象（带参数的）
public static Fragment newInstance(String sitle){
    Fragment fragment = new Fragment();	//创建对象
    Bundle bundle = new Bundle();		//构造Bundle对象
    bundle.putString("title",title);	//为bundle对象添加参数
    fragment.setArguments(bundle);		//为bundle对象绑定fragment
    return fragment; 					//返回Fragment对象
}

//取出得到参数的方法 (在onCreateView()或者onViewCreated()中)
if(getArguments()!=null){
    mTextView.setText(getArguments.getString("title"));
}

//初始化Fragment
Fragment fm = Fragment.newInstance("我是参数");
~~~



#### Fragment回退栈

如果Fragment—A跳转到Fragment—B，这个时候如果点击返回会直接关闭Fragment，这不是我们想看到的。

~~~java
//在Fragment中有一个按钮，点击它会跳到另一个Fragment——B.
if(bFragment==null){
    bFragment = new BFragment();
}
getFragmentManager().beginTransaction().replace(R.id.fl_main,bFragment)
    .addToBackStack(null)	//在commit之前加上这句话
    	.commitAllowinStateloss();
~~~

> 如果不想按返回就关闭Fragment，在commit之前加上addToBackStack(null)。

​	现在看上去似乎是完美的，但是仍然有个小问题：如果我们改变过Fragment里面的内容，Fragment跳转到另一个Fragment，然后又返回到原来的页面，我们发现页面又被重新初始化了——我们改变的内容又被恢复了

为什么会重新初始化呢？ 这是因为我们是调用replace()方法更换Fragment发生的问题

>replace()方法相当于remove()方法（删除） 和add()方法（添加）两个方法同时使用。

#### Fragment和Activity之间的通信

假如在Activity中有个TextView，在Fragment中有个Button，点击Button改变TextView的内容。

~~~java
//实现两者之间的通信其实很简单，getActivity可以直接获得Activity对象，然后直接调用就好
//Activity中,暴露一个方法
public void setTextView（String str）{
    mTextView.setText(str);
}

//Fragment中，通过getActivity()获得对象（记得强转类型）
((MainActivity)getActivity()).setTextView("我是从Fragment传来的数据");
~~~

_(:з)∠)_但是人家不推荐这么用啊。see below

~~~java
//推荐用的另一种方法
//在Fragment中写一个接口，然后交给Activity去实现
public interface IOnMessageClick{
    void onClick(String text)
}//接口写好了
private IonMessageClick listener; //在Fragment中创建一个接口对象
//Fragment中有个方法，与activity建立连接时调用
   @Override
   public void onAttach(Context context) {
        super.onAttach(context);
       //将context强转为listener
       listener = (IOnMessageClick)context;
       //还可以加入try/catch处理异常，如果context没有实现接口就抛出异常
   }
//点击按钮改变文字，所以要在button的点击事件里面写
listener.onClick("我是从Fragment传过来的参数")

//在Activity中，实现接口
    class MainActivity extends Activity implements Fragment.IOnMessageClick{
        @Override
        public void onClick(String text){
            mTextView.setText(text);
        }
    }

~~~
