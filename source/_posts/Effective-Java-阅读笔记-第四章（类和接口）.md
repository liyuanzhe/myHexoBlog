layout: '[layout]'
title: Effective Java 阅读笔记-第四章 类和接口
date: 2016-08-28 16:55:10
tags: tech
---
类和接口是Java程序设计语言的核心，他们也是Java语言的基本抽象单元。   
      
   

## 第16条 复合优先于继承   
* 继承打破了封装性      

超类的实现有可能会随着发型版本的不同而变化，如果变化，子类可能遭到破坏，即使字类的代码完全没有改变。   

举例说明，假设有一个程序使用了HashSet，想要查询它被创建以来添加了多少个元素，编写一个hashSet变量，它记录下试图插入的元素数量，并有一个访问数量的方法。

```
**
 * 复合优先于继承
 *
 * @param <E>
 */
// Broken - Inappropriate use of inheritance
public class InstrumentedHashSet<E> extends HashSet<E> {
    // The number of attempted element insertions
    private int addCount = 0;
     
    public InstrumentedHashSet() {
    }
     
    public InstrumentedHashSet(int initCap, float loadFactor) {
        super(initCap, loadFactor);
    }
     
    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }
     
    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }
     
    public int getAddCount() {
        return addCount;
    }
}
```

建立上述类之后，执行代码：
```
public static void main(String[] args) {
    InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
    s.addAll(Arrays.asList("a", "b", "c"));
    System.out.println(s.getAddCount());
}
```
上述代码执行了三次add，我们可以猜想s.getAddCount()将返回3，但实际上它返回的是6。
这是因为在HashSet内部，addAll方法是基于add方法实现的。继承之后，addAll和add之间的关系不变，因此总共增加了6。

* 复合解决了上述问题

```
/**
 * 复合优先于继承
 *
 * @param <E>
 */
// Reusable forwarding class
public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;
    public ForwardingSet(Set<E> s) {
        this.s = s;
    } 
    @Override
    public int size() {
        return s.size();
    }
    @Override
    public boolean isEmpty() {
        return s.isEmpty();
    }
    @Override
    public boolean contains(Object o) {
        return s.contains(o);
    }
    @Override
    public Iterator<E> iterator() {
        return s.iterator();
    }
    @Override
    public Object[] toArray() {
        return s.toArray();
    }
    @Override
    public <T> T[] toArray(T[] a) {
        return s.toArray(a);
    }
    @Override
    public boolean add(E e) {
        return s.add(e);
    }
    @Override
    public boolean remove(Object o) {
        return s.remove(o);
    }
    @Override
    public boolean containsAll(Collection<?> c) {
        return s.containsAll(c);
    }
    @Override
    public boolean addAll(Collection<? extends E> c) {
        return s.addAll(c);
    }
    @Override
    public boolean retainAll(Collection<?> c) {
        return s.retainAll(c);
    }
    @Override
    public boolean removeAll(Collection<?> c) {
        return s.removeAll(c);
    }
    @Override
    public void clear() {
        s.clear();
    }
}
```

```
/**
 * 复合优先于继承
 *
 * @param <E>
 */
// Wrapper class - uses composition in place of inheritance
public class InstrumentedSet<E> extends ForwardingSet<E> {
    private int addCount = 0;
     
    public InstrumentedSet(Set<E> s) {
        super(s);
    }
     
    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }
     
    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount++;
        return super.addAll(c);
    }
     
    public int getAddCount() {
        return addCount;
    }
}
```
ForwardingSet实现了Set接口，新类中的每个实例方法都可以调用被包含的现有类实例中对应的方法，并返回它的结果。这被称为__转发__。此时，ForwardingSet中的各个方法没有内部依赖。   

## 第17条 要么为继承而设计，并提供文档说明，要么就禁止继承   

* 被继承的类的文档必须精确地描述覆盖每个方法所带来的影响。对于每个public或protected方法，文档必须知名该方法或构造器调用了哪些可覆盖的方法，是以什么顺序调用的，每个调用的结果又是如何影响后续的处理过程的。
* 构造器绝不能调用可被覆盖的方法。因为构造器会先调用super()，

__参考：__   
<a href=https://book.douban.com/subject/3360807/>《Effective Java》</a>   
