# UUID的性能

## 背景知识简单说明
UUID含义是通用唯一识别码 (Universally Unique Identifier)，常用于分布式系统，比如作为消息唯一标识使用。
JDK中也提供了获取UUID的API方法：java.util.UUID.randomUUID()。本文基于JDK7，不同JDK版本的UUID实现不太一样。

## JDK中的UUID性能
先上测试代码：
```java
import java.util.UUID;

public class UUIDPerformanceTest {
	
	private static final int RUN_TIMES = 2000000;   // 执行次数
	
	public static void jdkUUIDTest() {
		System.out.println("jdkUUIDTest: ");
		long start = System.currentTimeMillis();
		for(int i = 0; i < RUN_TIMES;i++) {
			UUID.randomUUID();
		}
		long totalMillisElapsed = System.currentTimeMillis() - start;
		System.out.println("Total time: " + (totalMillisElapsed) 
				+ "ms, average: " + (double) totalMillisElapsed / RUN_TIMES + "ms");
		System.out.println();
	}
	
	public static void main(String[] args) {
		jdkUUIDTest();
	}

}

```

测试机器配置为CPU 2.9 GHz，内存8GB，得到的结果如下：

```html
jdkUUIDTest: 
Total time: 4464ms, average: 0.002232ms

```
