---
layout: post
title:  "Java 多线程"
date:   2018-10-15 
categories: jekyll update
---

#Java 多线程

### 线程状态的切换
![](https://image.ibb.co/bGRAmp/Thread_State.jpg)

## 双重锁判定的坑

```java
public class DoubleCheckedLocking { //1
    private static Instance instance; //2

    public static Instance getInstance() { //3
        if (instance == null) { //4:第一次检查
            synchronized (DoubleCheckedLocking.class) { //5:加锁
                if (instance == null) //6:第二次检查
                    instance = new Instance(); //7:问题的根源出在这里
            } //8
        } //9
        return instance; //10
    } //11

    static class Instance {
    }
}
```

![](https://image.ibb.co/diSVK9/1537965103782.jpg)
![](https://image.ibb.co/eFrt6p/1537965194392.jpg)

很有可能在B线程拿到的对象其实尚未完全初始化完成。解决方法就是将 `instance` 使用 `volatile` 修饰防止进行重排序

```java
public class DoubleCheckedLocking { //1
    private static volatile Instance instance; //2

    public static Instance getInstance() { //3
        if (instance == null) { //4:第一次检查
            synchronized (DoubleCheckedLocking.class) { //5:加锁
                if (instance == null) //6:第二次检查
                    instance = new Instance(); //7:问题的根源出在这里
            } //8
        } //9
        return instance; //10
    } //11

    static class Instance {
    }
}
```

## 同步关键字

当线程对 synchronized 保护的对象进行访问时才会去尝试获得锁，进行阻塞等待

```java
public class Synchronized {
    public static void main(String[] args)throws Exception {
        Runnable r1 = () ->{
            synchronized (s){
                while (true){
                    try {
                        Thread.sleep(2*1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };

        Runnable r2 = () ->{
//            synchronized (s){
//                System.out.println(s);
//            }
            System.out.println(s);
        };

        new Thread(r1).start();
        Thread.sleep(1000);
        new Thread(r2).start();
    }

    public static Synchronized s = new Synchronized();
}	
```

上述代码输出内容为 chapter04.Synchronized@2482ac6c 说明 r2 线程未阻塞

将注释替换代码如下

```java
public class Synchronized {
    public static void main(String[] args)throws Exception {
        Runnable r1 = () ->{
            synchronized (s){
                while (true){
                    try {
                        Thread.sleep(2*1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };

        Runnable r2 = () ->{
           synchronized (s){
                System.out.println(s);
           }
           //System.out.println(s);
        };

        new Thread(r1).start();
        Thread.sleep(1000);
        new Thread(r2).start();
    }

    public static Synchronized s = new Synchronized();
}	
```
结果为没有输出 

结果 一个线程对某个对象进行同步关键字保护，当该对象处于上锁时，另外的线程以无同步关键字方式访问是可以进行访问的。


### 等待方，通知方模型的编程范式
对上述代码进行提炼出等待/通知的经典范式，范式分为两个部分，等待方（消费者）和通知方（生产者）。

```java
等待方：
synchronized(对象){
	while(条件不满足){
		对象.wait();
	}
	对应的处理逻辑
}	

通知方:
synchronized(对象){
	改变条件
	对象.notifyAll();	
}	
```
添加等待超时限制，伪代码如下

```java
//当前对象加锁
public synchronized object get(long mills) throws InterruptedException{
	long future = System.currentTimeMills() + mills;
	long remaining = mills;
	// 当超时大于0并且result返回值不满足要求
	while((result == null) && remaining > 0){
		wait(remaining);
		remaining = future - System.currentTimeMills();
	}
	return result;
}
```

这样即使调用方式时间过长，也不会“永久”阻塞调用者。
