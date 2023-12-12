### GarbageCollection

- 哪些内存需要回收？
- 什么时候回收？
- 如何回收？

#### 对象已死？
- 引用计数法 
  - 在对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加一；当引用失效时，计数器值就减一；任何时刻计数器为零的对象就是不可能再被使用的。(Reference Counting，Java并不使用这个方法判断对象是否已死)
- 可达性分析
  - Reachability Analysis， GC Roots到这个对象不可达时，则证明此对象是不可能再被使用的。（Java的做法）
  - 四种引用   Strongly reference（不被回收）, soft reference（除非内存溢出，否则不回收）, weak reference（对象存活到下一次GC开始时）, phantom reference(虚引用，仅仅通知对象被回收)

#### 垃圾回收算法
##### 分代收集理论
    - 弱分代假说（Weak Generational Hypothesis）：绝大多数对象都是朝生夕灭的
    - 强分代假说（Strong Generational Hypothesis）：熬过越多次垃圾收集过程的对象就越难以消亡。
    - 跨代引用假说（Intergenerational Reference Hypothesis）：跨代引用相对于同代引用来说仅占极少数。

##### 部分垃圾回收策略
    - 新生代收集（M inor GC/Young GC）：指目标只是新生代的垃圾收集
    - 老年代收集（M ajor GC/Old GC）：指目标只是老年代的垃圾收集。目前只有CMS收集器会有单独收集老年代的行为。

- 标记-清除算法
- 复制算法
- 标记-整理算法
- 分代收集算法

#### 垃圾回收器

- Serial收集器
- ParNew收集器
- Parallel Scavenge收集器
- Serial Old收集器
- Parallel Old收集器
- CMS收集器
- G1收集器

#### 内存分配策略

- 对象优先在Eden分配
- 大对象直接进入老年代
- 长期存活的对象进入老年代
- 动态对象年龄判定
- 空间分配担保