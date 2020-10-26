我们在阅读源码的时候经常会看到有mask flag的与或运算，往往出现在判断的语句中，另我十分的困惑。

# 一、认识Mask flag

```java
    /**
     * Returns the enabled status for this view. The interpretation of the
     * enabled state varies by subclass.
     *
     * @return True if this view is enabled, false otherwise.
     */
    @ViewDebug.ExportedProperty
    public boolean isEnabled() {
        return (mViewFlags & ENABLED_MASK) == ENABLED;
    }
```
代码位于Android View.java文件，isEnable方法表示视图控件是否可以响应点击状态等。其中ENABLED_MASK的定义如下：

```java
    /**
     * Mask for use with setFlags indicating bits used for indicating whether
     * this view is enabled
     * {@hide}
     */
    static final int ENABLED_MASK = 0x00000020;
```
ENABLED_MASK为16进制的数，可以用于检验mViewFlags的enabla状态。ENABLED = 0x00000000,DISABLED = 0x00000020.

16进制的一位对应二进制4位，ENABLED_MASK去除高位0（0与运算均是0），低二位二进制位表示为0010 0000。

假设mViewFlags = 0x00000020,低二位为 0010 0000

```java
mViewFlags & ENABLED_MASK => 0010 0000 & 0010 0000 = 0010 0000 = 0x20 DISABLED 
```

可以看出只有低2位为0x20的情况，与mask的运算才能得到DISABLED的结果，此mask就是用于检验第二位是否为0x20的.

> 如果mViewFlags低2位为0x10,0x00等计算结果都是ENABLED,大家可以用上述类似的方式演算下。

通过上面判断状态的案例初步认识了mask flag是怎么工作的之后，问题来了

<b>通过一个true/false变量就能完成的事搞那么复杂干吗?</b>


# 二、 深入MASK

在只表示是否的时候，使用掩码和状态变量基本没有什么区别，但是当变量变得多起来的时候使用mask就变得更加清楚了。

这里列举一个更为复杂的场景,HUNGRY=0x0000 0010表示饿了，THIRST=0x0000 0100表示渴了，TIRDED=0x0000 1000表示困了，根据小明的上班的状态，决定小明做什么事情。


比如判断小明是不是饿了

```java
void isHungry(int flag){
    return flag & HUNGRY != 0;
}
```

此时如果小明只是渴了那么flag=0x0000 0100=> 0x0000 0100 & 0x0000 0001 = 0x0000 0000，所以没有饿。

当flag=0x0000 0010=> 0x0000 0010 & 0x0000 0010 = 0x0000 0010 = HUNGRY,说明是饿了。


但是如果小明又饿又困,很显然，这个没有办法用一个变量来表示当前的状态，用mask来玩就变得简单了。

```java
boolean isHungryAndThired(int flag){
    int HAT = (HUNGRY | TIRDED);//又饿又困没救了
    return (flag & HAT) == HAT;
}

```

如果flag此时是<b>渴了</b>即flag = THIRST

```java
flag & (HUNGRY | TIRDED) = 0000 0100 & (0000 0010 | 0000 1000) = 0000 0100 & 0000 1010 = 0000 00000 
```

如果是<b>既饿又渴</b>flag = \(HUNGRY \| THIRST\) = 0x0000 0110                                                                    
```java
flag & (HUNGRY | TIRDED) = 0000 0110 & 0000 1010 = 0000 0010
```                    

只有当flag = \(HUNGRY \| TIRDED\)

```java
flag & (HUNGRY | TIRDED) = 0000 1010 & 0000 1010 = 0000 1010 
```


以上只用定义的三个变量玩出很多花样，变量的与或可以灵活组合出一个新的状态，如又饿又困:(HUNGRY | TIRDED)，即不饿又不渴
:

~\(HUNGRY\|THIRST\)，然后又可以方便的去检测这些状态。

这里拓展下上面的HUNGRY,THIRST 和TIRDED使用二级制也可以表示为:

```java
   public static final int HUNGRY = 1 << 1; // 0010
   public static final int THIRST = 1 << 2; // 0100
   public static final int TIRDED = 1 << 3; // 1000
```


# 三、复位功能
<b>不仅可以完成复杂控制，还可以简便的完成复位清除等。</b>

继续拿上面的去举例，小明被判断为饿了之后去吃饭了，吃完应该就不饿了。

MASK = 0x0000 0010 

```java
if(flag & HUNGRY != 0){    
    toLunch();//饿了，去吃午饭了
    Thread.sleep(30*1000);//吃了三十分钟
    lunchFinish(flag);//吃完了
}
```

lunchFinish()

```java
void lunchFinish(boolean flag){
    flag &= ~HUNGRY;//不饿了
}
```

这里留给大家去推算。

常见的位运算操作：

- 测试第 k 位: s & (1 << k)
- 设置第 k 位: s \|= (1 << k)
- 第 k 位置零: s &= ~(1 << k)
- 切换第 k 位值: s ^= ~(1 << k)
- 乘以 2: s << n
- 除以 2: s >> n
- 交集: s & t
- 并集: s \| t
- 减法: s & ~t
- 交换 x = x ^ y ^ (y = x)
- 取出最小非 0 位（Extract lowest set bit）: s & (-s)
- 取出最小 0 位（Extract lowest unset bit）: ~s & (s + 1)
- 交换值:
       ```
          x ^= y;
          y ^= x;
          x ^= y;
       ```