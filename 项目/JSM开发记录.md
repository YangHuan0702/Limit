


#为什么java的class要克隆时不是直接new 而是需要通过`super.clone()` ?
这是一个触及 Java 语言底层设计哲学的经典问题。
在重写 `clone()` 方法时，如果直接使用 `new` 关键字（比如 `return new MyClass();`）而不是调用 `super.clone()`，在当前类自身的上下文中似乎没问题，但这会**彻底破坏类的继承体系**。
1. 继承体系下的“类型截断”灾难（最致命的原因）,`clone()` 方法的契约（Contract）隐含了一个极其重要的前提：**克隆出来的对象，必须和被克隆对象的运行时真实类型（Runtime Type）完全一致。** 如果父类中的`clone()`中直接`new`，那么子类调用`super.clone()`的时候返回的类就是父类的类型，而我们如果这时在子类中直接强转为子类类型的话，就会收到一个**`ClassCastException`**。因为父类返回的是确确实实的 `Base` 对象，它在内存里根本没有子类 `Derived` 的空间（比如字段 `y`），无法向下转型。
2. 当你使用 `new` 关键字时，JVM 的标准流程是：分配内存 -> **调用构造函数** -> 初始化各个字段。而 `Object.clone()` 走的是完全不同的路线,它是一个 C++ 实现的本地方法（Native Method）它在分配完正确大小的内存后，会直接进行底层内存的**按位复制（Bitwise Copy，类似于 C 语言里的 `memcpy`）**。**它会完全绕过构造函数（Constructor）。**
3. `super.clone()` 是如何解决这个问题的？由于所有的 `super.clone()` 最终都会向上追溯调用到 `java.lang.Object` 的 `clone()` 方法。`Object.clone()` 是一个 `native` 方法，它在 JVM 底层拥有上帝视角。**它会读取当前对象对象头（Object Header）里的类型指针（Klass Pointer），获知这个对象的真实运行时类型，然后直接在堆内存中开辟一块与该真实类型大小完全相同的内存空间。** 这样就保证了 `Derived.clone()` 出来的永远是 `Derived` 实例。

#System拷贝数组的注意点
```
private synchronized void checkAdjustTableLevel(int level) {  
    int oldLen = Objects.nonNull(table) ? table.length : 0;  
  
    if (oldLen <= level) {  
        SkipTableElement[] newTable = new SkipTableElement[level + 1];  
  
        ReadWriteLock[] newLocks = new ReadWriteLock[level + 1];  
  
        if (oldLen > 0) {  
            System.arraycopy(table, 0, newTable, 0, oldLen);  
            System.arraycopy(levelLocks, 0, newLocks, 0, oldLen);  
        }  
  
        for (int i = oldLen; i <= level; i++) {  
            newTable[i] = new SkipTableElement(null, null, null);  
            newTable[i].setDown(i == 0 ? null : newTable[i - 1]);  
  
            newLocks[i] = new ReentrantReadWriteLock();  
        }  
  
        table = newTable;  
        levelLocks = newLocks;  
    }  
    currentMaxLevel = Math.max(currentMaxLevel, level);  
}
```
上面的代码是判断是否需要重新构建跳表，其中会通过`System.arraycopy`的方式来进行数组拷贝。