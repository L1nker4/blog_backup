---
title: ThreadLocalRondom原理剖析
date: 2020-04-28 17:31:07
categories:
- Java
- Java并发
tags:
- Java
- 并发
---





ThreadLocalRondom是JDK 7在并发包中新增的随机数生成器，该类弥补了Random类在并发环境下的缺陷。

<!--more-->



## Random的局限性

Random生成随机数的方法如下：

```java
public int nextInt(int bound) {
    	//参数校验
        if (bound <= 0)
            throw new IllegalArgumentException(BadBound);
		//根据老的种子生成新的种子
        int r = next(31);
        int m = bound - 1;
    	//根据新种子计算随机数
        if ((bound & m) == 0)
            r = (int)((bound * (long)r) >> 31);
        else {
            for (int u = r;
                 u - (r = u % bound) + m < 0;
                 u = next(31))
                ;
        }
        return r;
    }
```

next方法通过计算生成新的种子。用原子变量来存放种子，多线程的情况下，CAS操作会保证只有一个线程可以更新老的种子为新种子，更新失败的线程进行自旋，这降低了并发性能，所以产生了ThreadLocalRandom。

```java
protected int next(int bits) {
        long oldseed, nextseed;
        AtomicLong seed = this.seed;
        do {
            oldseed = seed.get();
            //seed计算公式，通过CAS操作进行更新
            nextseed = (oldseed * multiplier + addend) & mask;
        } while (!seed.compareAndSet(oldseed, nextseed));
    	//将得到的值进行逻辑右移
        return (int)(nextseed >>> (48 - bits));
    }
```





## ThreadLocalRandom简介

ThreadLocalRandom和ThreadLocal的原理相似，ThreadLocalRandom使得每个线程都维护自己独有的种子变量，这样就不存在竞争问题，大大提高并发性能。





## current方法

在current方法中，获得ThreadLocalRandom实例并初始化。seed不再是一个AtomicLong变量，在Thread类中有三个变量。

- threadLocalRandomSeed：使用它来控制随机数种子。
- threadLocalRandomProbe：使用它来控制初始化。
- threadLocalRandomSecondarySeed：二级种子。

这三个变量都加了`sun.misc.Contended`注解，用来避免伪共享问题。

```java
    public static ThreadLocalRandom current() {
        //判断是否初始化
        if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
            //进行初始化
            localInit();
        return instance;
    }

	static final void localInit() {
        int p = probeGenerator.addAndGet(PROBE_INCREMENT);
        int probe = (p == 0) ? 1 : p; // skip 0
        long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
        Thread t = Thread.currentThread();
        UNSAFE.putLong(t, SEED, seed);
        UNSAFE.putInt(t, PROBE, probe);
    }


	/** The current seed for a ThreadLocalRandom */
    @sun.misc.Contended("tlr")
    long threadLocalRandomSeed;

    /** Probe hash value; nonzero if threadLocalRandomSeed initialized */
    @sun.misc.Contended("tlr")
    int threadLocalRandomProbe;

    /** Secondary seed isolated from public ThreadLocalRandom sequence */
    @sun.misc.Contended("tlr")
    int threadLocalRandomSecondarySeed;
```



## Unsafe机制

```java
private static final sun.misc.Unsafe UNSAFE;
private static final long SEED;
private static final long PROBE;
private static final long SECONDARY;
static {
    try {
        //获取Unsafe实例
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> tk = Thread.class;
        //获取Thread类里面threadLocalRandomSeed变量在Thread实例的偏移量
        SEED = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSeed"));
        //获取Thread类里面threadLocalRandomProbe变量在Thread实例的偏移量
        PROBE = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomProbe"));
        //获取Thread类里面threadLocalRandomSecondarySeed变量在Thread实例的偏移量
        SECONDARY = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
```



## nextInt方法

nextInt方法用于获取下一个随机数。

```java
	public int nextInt(int bound) {
        //参数校验
        if (bound <= 0)
            throw new IllegalArgumentException(BadBound);
        //根据当前线程中的种子计算新种子
        int r = mix32(nextSeed());
        //根据新种子计算随机数
        int m = bound - 1;
        if ((bound & m) == 0) // power of two
            r &= m;
        else {
            for (int u = r >>> 1;
                 u + m - (r = u % bound) < 0;
                 u = mix32(nextSeed()) >>> 1)
                ;
        }
        return r;
    }
```



## nextSeed方法

获取当前线程的threadLocalRandomSeed变量值，然后加上GAMMA值作为新种子。可参照上文Unsafe机制。

```java
final long nextSeed() {
    Thread t; long r;
    UNSAFE.putLong(t = Thread.currentThread(), SEED,
                   r = UNSAFE.getLong(t, SEED) + GAMMA);
    return r;
}
```

## 参考

《Java并发编程之美》