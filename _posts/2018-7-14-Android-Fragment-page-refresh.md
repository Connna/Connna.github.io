---
layout: post
title: Fragment切换页面内容不刷新的解决办法
tags: Android   
---

我使用Fragment分为这几步

#### 错误代码

### 一、初始化所有Fragment，全部add，然后全部隐藏，

~~~java
private  FragmentTransaction transaction；//将transaction设为全局变量

public void initFragment(){
     //获得fragment布局管理器
        android.app.FragmentManager fm = getFragmentManager();
       	transaction = fm.beginTransaction();
     //初始化所有Fragment；
            homeFragment=new HomeFragment();
            videoFragment=new VideoFragment();
            musicFragment= new MusicFragment();
            userFragment = new UserFragment();
	//add所有Fragment；
        transaction.add(R.id.fl_main,homeFragment);
        transaction.add(R.id.fl_main,videoFragment);
        transaction.add(R.id.fl_main,musicFragment);
        transaction.add(R.id.fl_main,userFragment);
	//隐藏全部
        hideAllFrag();
     //显示首页一个fragment;
        transaction.show(homeFragment);
     //commit（）
        transaction.commitAllowingStateLoss();
    }
~~~

##### hideAllFrag（）方法——隐藏所有Fragment

```java
private void hideAllFrag() {
    transaction.hide(homeFragment);
    transaction.hide(videoFragment);
    transaction.hide(musicFragment);
    transaction.hide(userFragment);
}
```



### 二、按钮切换Fragment，隐藏全部fragment，只显示需要显示的

初始化没有任何问题，界面显示也是正常的，然而在按钮切换Fragment确没有任何反应。

```java
hideAllFrag();		//隐藏全部fragment
switch (checkedId){
    case R.id.rb_home: //首页
        transaction.show(homeFragment);
        break;
    case R.id.rb_file: //视频
        transaction.show(videoFragment);
        break;
    case R.id.rb_music: //音乐
        transaction.show(musicFragment);
        break;
    case R.id.rb_user: //用户
        transaction.show(userFragment);
        break;
}
```



### 三、分析问题和解决办法；

##### 正确代码

~~~java
//1、transaction不能设为全局变量
//初始化Fragment
public void initFragment(){
     //获得fragment布局管理器
        android.app.FragmentManager fm = getFragmentManager();
       	FragmentTransaction transaction = fm.beginTransaction();
     //初始化所有Fragment；
            homeFragment=new HomeFragment();
            videoFragment=new VideoFragment();
            musicFragment= new MusicFragment();
            userFragment = new UserFragment();
	//add所有Fragment；
        transaction.add(R.id.fl_main,homeFragment);
        transaction.add(R.id.fl_main,videoFragment);
        transaction.add(R.id.fl_main,musicFragment);
        transaction.add(R.id.fl_main,userFragment);
	//隐藏全部
        hideAllFrag(transaction);  					//这里把当前局部变量transaction传入
     //显示首页一个fragment;
        transaction.show(homeFragment);
     //commit（）
        transaction.commitAllowingStateLoss();
    }
~~~

##### hedeAllFrag(FragmentTransaction transaction)方法 ：现在需要传入一个FragmentTransaction 对象。

~~~java
//隐藏所有Fragment
private void hideAllFrag(FragmentTransaction transaction) { //传入一个FragmentTransaction 对象
    transaction.hide(homeFragment);
    transaction.hide(videoFragment);
    transaction.hide(musicFragment);
    transaction.hide(userFragment);
}
~~~



##### 按钮监听：show()方法之后一定要跟commit()方法

~~~java
//监听事件，根据id显示fragment
FragmentTransaction transaction = getFragmentManager().beginTransaction(); //创建transaction
            hideAllFrag(transaction);							//传入transaction。
            switch (checkedId){
                case R.id.rb_home: //首页
                    transaction.show(homeFragment).commitAllowingStateLoss();
                    							//每次show后面一定要有commit();
                    break;
                case R.id.rb_file: //视频
                    transaction.show(videoFragment).commitAllowingStateLoss();
                    break;
                case R.id.rb_music: //音乐
                    transaction.show(musicFragment).commitAllowingStateLoss();
                    break;
                case R.id.rb_user: //用户
                    transaction.show(userFragment).commitAllowingStateLoss();
                    break;
            }
~~~



总结：

如果使用和我一样的思路：add全部fragment，隐藏全部Fragment，然后根据需要显示对应的页面。

那么要注意这几点：

1、不能图方便将FragmentTransaction对象设置为成员变量。哪里需要就在哪里获得然后传入。

2、每次show()方法后面都需要使用commit()方法。
