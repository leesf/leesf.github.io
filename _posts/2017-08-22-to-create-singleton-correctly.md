---
title: 多线程环境下正确创建单例
date: 2017-08-22 18:23:16
categories:
- technique
tags:
- 单例
- Java
---
## 前言
> 对项目代码进行扫描时，出现静态扫描严重问题，发现是由于多线程环境下没有正确创建单例所导致。

## 问题分析
> 本项目使用的`JDK 1.7+`。

项目代码如下（修改了类名，但核心没变）

		
```java 
static class Singleton {
	private static volatile Singleton cache = null;
	private static Object mutexObj = new Object();
		
	private Singleton() {
		
	}
		
	public static Singleton getInstance() {
		Singleton tmp = cache;
	    if (tmp == null) {
			synchronized (mutexObj) {
				if (tmp == null) {	                	
					tmp = new Singleton();
					cache = tmp;  
				}	              
			}
		}
		return tmp;
	}
}
```

按照项目生成单例代码，使用如下测试类进行测试

```java

public class Test {
	public static void main(String[] args) {	
		for (int i = 0; i < 3; i++) {
			Thread thread = new Thread(new Runnable() {
				public void run() {
					System.out.println(Thread.currentThread().getName() + " " + Singleton.getInstance().toString());
				}
				
			});
			thread.setName("Thread" + i);
			thread.start();
		}
		
	}

	static class Singleton {
		private static volatile Singleton cache = null;
		private static Object mutexObj = new Object();
		
		private Singleton() {
		
		}
		
		public static Singleton getInstance() {
			Singleton tmp = cache;
	    	if (tmp == null) {
				synchronized (mutexObj) {
					if (tmp == null) {	                	
						tmp = new Singleton();
						cache = tmp;  
					}	              
				}
			}
			return tmp;
		}
	}
}


```

输出结果如下：



> Thread1 com.hust.grid.leesf.mvnlearning.Test$Singleton@304e94a4  
Thread0 com.hust.grid.leesf.mvnlearning.Test$Singleton@304e94a4  
Thread2 com.hust.grid.leesf.mvnlearning.Test$Singleton@304e94a4  


从结果看，都生成了同一个实例，似乎不存在问题，多线程环境下确实不太好重现问题，现改动代码如下：

```java

static class Singleton {
	private static volatile Singleton cache = null;
	private static Object mutexObj = new Object();
		
	private Singleton() {
		
	}
		
	public static Singleton getInstance() {
		Singleton tmp = cache;
	    if (tmp == null) {
	        System.out.println(Thread.currentThread().getName() + " in outer if block");
	        synchronized (mutexObj) {
				System.out.println(Thread.currentThread().getName() + " in synchronized block");
	            if (tmp == null) {	 
	               	System.out.println(Thread.currentThread().getName() + " in inner if block");
	               	tmp = new Singleton();
	                cache = tmp;  
	            }	
	            System.out.println(Thread.currentThread().getName() + " out inner if block");
			}
	        System.out.println(Thread.currentThread().getName() + " out synchronized block");
	    }
	    System.out.println(Thread.currentThread().getName() + " out outer if block");
	    return cache;
	}
}


```

上述代码中添加了`Thread.sleep(1)`这条语句，其中，`Thread.sleep(1)`进行休眠时，线程不会释放拥有的锁，并且打印了相关的语句，便于查看线程正运行在哪里的状态。

再次测试，输出结果如下：

> Thread2 in outer if block  
Thread1 in outer if block  
Thread0 in outer if block  
Thread2 in synchronized block  
Thread2 in inner if block  
Thread2 out inner if block  
Thread2 out synchronized block  
Thread0 in synchronized block  
Thread2 out outer if block  
Thread2 com.hust.grid.leesf.mvnlearning.Test$Singleton@60b07af1  
Thread0 in inner if block  
Thread0 out inner if block  
Thread0 out synchronized block  
Thread1 in synchronized block  
Thread0 out outer if block  
Thread1 in inner if block  
Thread0 com.hust.grid.leesf.mvnlearning.Test$Singleton@625795ce  
Thread1 out inner if block  
Thread1 out synchronized block  
Thread1 out outer if block  
Thread1 com.hust.grid.leesf.mvnlearning.Test$Singleton@642c39d2


从结果看，生成了3个不同的实例，并且每个线程都执行了完整的流程，并且可知单例的创建存在问题。在分析原因前简单了解下多线程模型，多线程模型如下：

![多线程内存模型](https://raw.githubusercontent.com/leesf/blogPhotos/master/thread-model.png)

> 每个线程有自己独立的工作空间，线程间进行通信是通过主内存完成的，想了解详细内容可参见如下链接:[内存模型](http://www.cnblogs.com/leesf456/p/5291484.html)或[深入理解java内存模型](http://files.cnblogs.com/skywang12345/%E6%B7%B1%E5%85%A5Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.pdf)。


知道每个线程会有一份tmp拷贝后，配合打印输出，就不难分析出原因。

## 问题解决
> 按照《Effective Java》一书中创建单例的推荐，可使用如下两种解决方法

### 双重锁检查机制

> 需要配合`volatile`关键字使用，并且需要`JDK`版本在`1.5`以上，核心代码如下

```java

static class Singleton {
	private static volatile Singleton cache = null;
	private static Object mutexObj = new Object();
		
	private Singleton() {
		
	}
		
	public static Singleton getInstance() {
		Singleton tmp = cache;
	    if (tmp == null) {
			tmp = cache;
			synchronized (mutexObj) {
				if (tmp == null) {	                	
					tmp = new Singleton();
					cache = tmp;  
				}	              
			}
		}
		return tmp;
	}
}

```

进行如下测试（添加打印语句方便分析）：

```java

public class Test {
	public static void main(String[] args) {	
		for (int i = 0; i < 3; i++) {
			Thread thread = new Thread(new Runnable() {
				public void run() {
					System.out.println(Thread.currentThread().getName() + " " + Singleton.getInstance().toString());
				}
				
			});
			thread.setName("Thread" + i);
			thread.start();
		}
		
	}
	
	static class Singleton {
		private static volatile Singleton cache = null;
		private static Object mutexObj = new Object();
		
		private Singleton() {
		
		}
		
		public static Singleton getInstance() {
			Singleton tmp = cache;
	        if (tmp == null) {
	        	System.out.println(Thread.currentThread().getName() + " in outer if block");
	            synchronized (mutexObj) {
	            	System.out.println(Thread.currentThread().getName() + " in synchronized block");
	            	tmp = cache;
	                if (tmp == null) {	 
	                	System.out.println(Thread.currentThread().getName() + " in inner if block");
	                	tmp = new Singleton();
	                	try {
	                		Thread.sleep(1);
	                	} catch (InterruptedException e) {
	                		e.printStackTrace();
	                	}
	                    cache = tmp;  
	                }	
	                System.out.println(Thread.currentThread().getName() + " out inner if block");
	            }
	            System.out.println(Thread.currentThread().getName() + " out synchronized block");
	        }
	        System.out.println(Thread.currentThread().getName() + " out outer if block");
	        return tmp;
		}
	}
}


```

输出结果如下：


> Thread0 in outer if block  
Thread0 in synchronized block  
Thread0 in inner if block  
Thread2 in outer if block  
Thread1 in outer if block  
Thread0 out inner if block  
Thread0 out synchronized block  
Thread1 in synchronized block  
Thread1 out inner if block  
Thread1 out synchronized block  
Thread1 out outer if block  
Thread0 out outer if block  
Thread1 com.hust.grid.leesf.mvnlearning.Test$Singleton@13883d5f  
Thread0 com.hust.grid.leesf.mvnlearning.Test$Singleton@13883d5f  
Thread2 in synchronized block  
Thread2 out inner if block  
Thread2 out synchronized block  
Thread2 out outer if block  
Thread2 com.hust.grid.leesf.mvnlearning.Test$Singleton@13883d5f


**从结果中和线程运行步骤可以看到三个线程并发的情况下，只生成了唯一实例。**

### 静态内部类

> 无`JDK`版本限制，也不需要使用`volatile`关键字即可完成单例模式，核心代码如下：

```java

static class Singleton {

	private Singleton() {
		
	}
		
	private static class InstanceHolder {
        public static Singleton instance = new Singleton();
    }
	
    public static Singleton getInstance() {
	    return InstanceHolder.instance;  
	}
}

```

进行如下测试：

```java

public class Test {
	public static void main(String[] args) {	
		for (int i = 0; i < 3; i++) {
			Thread thread = new Thread(new Runnable() {
				public void run() {
					System.out.println(Thread.currentThread().getName() + " " + Singleton.getInstance().toString());
				}
				
			});
			thread.setName("Thread" + i);
			thread.start();
		}
		
	}
	
	static class Singleton {
		
		private Singleton() {
		
		}
		
		private static class InstanceHolder {
	        public static Singleton instance = new Singleton();
	    }

	    public static Singleton getInstance() {
	        return InstanceHolder.instance;  
	    }
	}
}

```

运行结果如下：


> Thread2 com.hust.grid.leesf.mvnlearning.Test$Singleton@71f801f7  
Thread1 com.hust.grid.leesf.mvnlearning.Test$Singleton@71f801f7  
Thread0 com.hust.grid.leesf.mvnlearning.Test$Singleton@71f801f7  


**该模式可保证使用时才会初始化变量，达到延迟初始化目的。**

## 总结

> 单例模式在多线程环境下不太好编写，并且不容易重现异常，编写时需要谨慎，在项目中遇到问题也需要多总结和记录。


## 参考文档

* [双重检查锁定与延迟初始化](http://www.infoq.com/cn/articles/double-checked-locking-with-delay-initialization)  
* Effective Java
* 深入理解Java内存模型
