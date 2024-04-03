# JAVA多线程

## 1. 线程的创建

```mermaid
flowchart LR
创建Runnable的实现类-->实现run函数-->创建实现类的对象,调用start方法.
```

## 2.线程创建的具体方式

1. 通过**继承`Thread`**类来实现多线程创建

   class TestThread extends Thread{
       @Override
       public void run(){
           System.out.println("hello");
       }
   }
   
   public class App{
       public static void main(String[] args){
           new TestThread().start();//创建并执行了一个线程
       }
   }
   ```

   

2. 通过实现`Runnable`类和`Thread`代理来实现多线程创建

   

   ```java
   class TestThread implements Runnable{
       @Override
       public void run(){
           System.out.println("hello");
       }
   }
   
   public class App{
       public static void main(String[] args){
           new Thread(new TestThread).start();//创建并执行了一个线程
       }
   }
   ```

   

## 3. 比较两种方法

由于java单继承的局限性，一般不建议使用第一种方式（直接继承Thread类）来创建多线程，而且使用第二种方式创建多线程时，可以使用lambda表达式或者匿名类，来实现具体多线程中要执行的内容。

## 线程的优先级

在Java中的线程执行都是有优先级的，但是优先级高并不代表一定先执行，只是表示先执行的概率更大。

在java中有个`Thread`定义了一些基本的优先级，其中就包括最大优先级，最小优先级和正常优先级。

| 优先级     | 数指 | 说明                                                       |
| ---------- | ---- | ---------------------------------------------------------- |
| 最大优先级 | 10   | 超出这个会报错                                             |
| 正常优先级 | 5    | 一般的线程（没有设置优先级的线程）就是这个值，包括main线程 |
| 最小优先级 | 1    | 低于这个会报错                                             |

## 线程的常用函数

1. **start()**：启动线程

2. **sleep()**：休眠线程

3. **setPriority()**：设置线程优先级

4. **setDaemon()**：设置为守护线程

5. **join()**：插队线程，也就是前行进入执行状态，直到执行完，其他的线程才能执行

6. ***yield()***：*静态方法*，当前线程礼让，也就是强制当前正在执行的线程进入挂起状态。但是没有指明接下来谁执行。

7. **getState()**：获取线程状态。在Thread中有定义枚举类型来表示，一般有五种状态：


> - *NEW*：新建的线程，还没有唤醒
> - *RUNNABLE*：可执行状态，这就包括正在执行或者等待CPU选中两种情况。
> - *BLOCKED*：阻塞状态，阻塞状态是线程阻塞在进入synchronized关键字修饰的方法或代码块(获取锁)时的状态。
  > - *WAITING*： 等待，处于这种状态的线程不会被分配 CPU 执行时间，它们要等待被显式地唤醒，否则会处于无限期等待的状态。
> - *TIMED_WAITING*：超时等待，处于这种状态的线程不会被分配 CPU 执行时间，不过无须无限期等待被其他线程显示地唤醒，在达到一定时间后它们会自动唤醒。
> - *TERMINATED*：终止状态，当线程的 run() 方法完成时，或者主线程的 main() 方法完成时，我们就认为它终止了。这个线程对象也许是活的，但是，它已经不是一个单独执行的线程。线程一旦终止了，就==不能复生==。

## 守护线程

守护线程相当于是一个服务一样，不一定要等到他结束整个程序才能结束，他往往只是提供某种服务，当我们的主要逻辑业务结束时，他也就没有存在的意义，也就会自动结束，如我们常谈的GCC（垃圾回收），他一直保持在后台运行，但是当我们的主方法结束时，他也便会自动结束。

> 并不是说守护线程一定要写成一个死循环等待JVM将其结束，他也可以自己结束，只是当没有用户线程正在运行时，JVM会强制结束守护线程。

## synchronized

这是一个java内置的锁，可以直接当做关键字使用。

- **用法**

  > synchronized 可以用来修饰方法或者代码块，但是他锁住的不是代码，而是数据，具体所哪个数据可以自定义。如果写在成员方法上面，那么他锁住的就是this，如果学士代码块，就需要指定对象上锁。

  ```java
  public class LockTest{
      private int num;
      private Date date;
      //方法上锁，默认将方法所属对象上锁
      public synchronized int subNum(int sub){
          if(sub<=0){
              this.num-=sub;
          }
          return num;
      }
      
      public String diffDate(Date d){
          //指定对象枷锁
          synchronized(data){
              return date.compareTo(d);
          }
      }
  }
  ```

- **锁的工作流程**

  > 首先要知道JMM（java memory module，java内存模型）,每个线程拥有自己的一个工作空间，然后共享一块主内存，主内存一般放的就是共享数据（对象），在每个线程启动后，都会从主内存中复制一份数据（自己需要的）到自己的工作空间中。