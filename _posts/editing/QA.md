1. ThreadLocal 传递参数  
	
``` java 
	/**
	 *	提供一个 ThreadLocalCache 单例实例，线程在运行期都能访问到它
	 * 每个线程访问到的都是它自己设定的值
	 * 使用完成后注意使用 remove 方法清楚变量，避免内存泄漏
	 */
	public class ThreadLocalCache {
		private ThreadLocal<Integer> params;
		private ThreadLocalCache(){
			params = new ThreadLocal<>();
		}
		private static ThreadLocalCache threadLocalCache = new ThreadLocalCache();
		
		public Integer getParam(){
			return params.get();
		}
		public void setParam(Integer param){
			params.set(param);
		}
		
		public static ThreadLocalCache getInstance(){
			return threadLocalCache;		
		}
	}
	
	/**
	 *	线程中设定参数 
	 */
	 public class SomeController{
	 	public void someMethod(Integer param){
	 		//...
	 		ThreadLocalCache.getInstance().setParam(param);
	 		//...
	 	}
	 }
	 
	 /**
	 *	同一线程中使用参数 
	 */
	 public class SomeService{
	 	public void someMethod(){
	 		//...
	 		Integer param = ThreadLocalCache.getInstance().getParam();
	 		//...
	 	}
	 }

``` 