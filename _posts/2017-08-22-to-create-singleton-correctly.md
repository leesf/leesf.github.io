## 前言
> 对项目代码进行扫描时，发现静态扫描严重问题，发现是由于多线程环境下没有正确创建单例所导致。

## 问题原因
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