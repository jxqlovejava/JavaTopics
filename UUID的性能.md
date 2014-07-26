## UUID的性能

### 背景知识简单说明
近两周在为开放平台项目提供一个OAuth2服务端，生成appKey、appSecret时使用了JDK提供的UUID实现。
UUID含义是通用唯一识别码 (Universally Unique Identifier)，常用于分布式系统，比如作为消息唯一标识使用。
JDK中也提供了获取UUID的API方法：java.util.UUID.randomUUID()。本文基于JDK7，不同JDK版本的UUID实现不太一样。

### JDK中的UUID性能
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
根据Google工程师Jeff Dean提供的那张性能数据图：

![Performance Numbers][./imgs/Performance_Numbers.png]

平均每次大概2232纳秒，性能并不是很理想。

为何这个操作会慢？网上有人说是因为UUID底层基于SecureRandom，而构造SecureRandom对象很慢，SecureRandom使用了操作系统的随机设备（Linux系统上是/dev/random）。我认可构造SecureRandom很慢，测试后发现构造一个SecureRandom对象大概388ns。我们继续看下UUID的源码：
```java
    /*
     * The random number generator used by this class to create random
     * based UUIDs. In a holder class to defer initialization until needed.
     */
    private static class Holder {
        static final SecureRandom numberGenerator = new SecureRandom();
    }

    public static UUID randomUUID() {
        SecureRandom ng = Holder.numberGenerator;

        byte[] randomBytes = new byte[16];
        ng.nextBytes(randomBytes);
        randomBytes[6]  &= 0x0f;  /* clear version        */
        randomBytes[6]  |= 0x40;  /* set to version 4     */
        randomBytes[8]  &= 0x3f;  /* clear variant        */
        randomBytes[8]  |= 0x80;  /* set to IETF variant  */
        return new UUID(randomBytes);
    }
```
可以看到只有第一次调用randomUUID()时才会new一个SecureRandom对象，之后所有调用都直接使用这个构造好的SecureRandom对象。所以SecureRandom对象构造不是导致UUID性能的原因。退一步，就光从时间消耗数字上看，每一次randomUUID()调用的平均时间是2232ns，而构造一个SecureRandom对象只需要388ns，只是randomUUID的一个零头而已。

既然SecureRandom不是导致UUID性能差的罪魁祸首，那么我们继续看ng.nextBytes方法源码：
```
    /**
     * Generates a user-specified number of random bytes.
     *
     * <p> If a call to <code>setSeed</code> had not occurred previously,
     * the first call to this method forces this SecureRandom object
     * to seed itself.  This self-seeding will not occur if
     * <code>setSeed</code> was previously called.
     *
     * @param bytes the array to be filled in with random bytes.
     */

    synchronized public void nextBytes(byte[] bytes) {
        secureRandomSpi.engineNextBytes(bytes);
    }
```
一个synchronzied方法，可以推测在多线程并发访问环境下，UUID的性能更差。但测试代码是单线程，所以UUID性能差跟这个应该关系不大（可能会有一点关系，毕竟synchronized加了对象内置锁，每次执行这段代码前可能都要检查下）。

