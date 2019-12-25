---
layout:     post
title:      "数据结构(三)栈、队列"
subtitle:   ""
date:       2019-08-22 02:30:00
author:     "YaPi"
header-img: ""
tags:
    - 数据结构
---

### 栈

#### 基础特点

- 栈也是一种线性结构
- 相比数组，栈对应的操作是数组的子集
- 只能从一段添加元素，也只能从一段取出元素
- 添加(入栈)或取出(出栈)的那一段称之为栈顶

后进先出 (Last In First Out) LIFO

#### 栈的应用

- 无处不在的Undo操作(撤销操作)

操作系统或程序会将你的操作放进一个栈中。点击撤销的时候，将最后添加的操作出栈。

- 程序调用的系统栈

在程序的运行过程中，去调用另外的程序。若有三个函数A、B、C。在A中运行B，B中运行C。执行过程开始，运行A，当运行到执行B的代码时,将A中此位置进行入栈。开始运行B，在B中运行到执行C的代码，将B中此位置进行入栈。开始运行C，当C运行完了过后，通过栈的数据结构，将B中运行C的位置的元素进行出栈，那么又回到了B，同理，继续向下执行，而后，又回到了A。这样，通过栈，就能让函数运行过后，跳转到上级调用函数。

- 括号匹配


### 队列 (Queue)

#### 特点

- 队列也是一种线性结构
- 相比数组，队列对应的操作是数组的子集
- 只能从一端(队尾)添加元素，只能从另一端(对首)取出元素

先进先出(First In First Out) FIFO


#### 实现方式

##### 数组队列

缺点: 出队的时间复杂度是O(n)的，这是因为删除队首，会将所有其他的元素都向前移动一个单位。


##### 循环队列

同样可以使用数组结构构成，但是需要维护两个变量，front和tail，分别代表对首和队尾的指针。并且，指针的位置是可以循环使用的，就比如这样一种情况:

一个长度为8的数组，首先新建生成的时候，队首和队尾的指针都指向0,此时，front == tail,队列为空。当后续继续添加元素后，tail指向7，即此时数组的最后一个元素，此时，若队列已出队两次，则位置为 0 、1的空间是可以继续被使用的，那么，tail的指针就应该指向0


这样就有两个特殊点：

- front == tail 队列为空
- (tail + 1) % c == front队列已满

针对队列已满的情况，当循环到tail + 1等于front的时候，说明数组已经在循环使用了，此时，队列已满。第二种情况，当队首是0，队尾是末尾，此时队列还是已满，所有不是tail + 1等于front,而是 (tail + 1) % c == front代表队列已满。当满足(tail + 1) % c == front这种情况时，数组就需要扩容了。

#### 队列实现-使用自定义数组

```
/*
@date : 2019/09/06
@author : YaPi
@desc : 使用自定义数组实现队列
*/
package queue

import "dtSt/array"

type arrayQueue struct {
	array *array.SArray
}

func (s *arrayQueue) String() string {
	return s.array.String()
}

func NewArrayQueue() *arrayQueue {
	return &arrayQueue{array: array.NewSArray()}
}

func (s *arrayQueue) Size() int {
	return s.array.Size()
}

func (s *arrayQueue) IsEmpty() bool {
	return s.array.Size() == 0
}

func (s *arrayQueue) Enqueue(q array.E) {
	s.array.AddLast(q)
}

func (s *arrayQueue) Dequeue() array.E {
	return s.array.RemoveFirst()
}

func (s *arrayQueue) GetFront()array.E {
	return s.array.Get(0)
}

```

测试

```
/*
@date : 2019/09/06
@author : YaPi
@desc :
*/
package queue

import (
	"dtSt/array"
	"fmt"
	"math/rand"
	"strconv"
	"testing"
)

type NewInt int

func (ee NewInt) String() string {
	return strconv.Itoa(int(ee))
}

func (ee NewInt) CompareTo(e array.E) int {

	eeInt := int(ee)

	eInt := int(e.(NewInt))

	if eeInt > eInt {
		return 1
	}else if eeInt == eInt{
		return 0
	}else {
		return -1
	}
	return 0
}


var q arrayQueue

func init()  {
	a := array.NewSArray()
	for i:=0;i<5;i++{
		newInt := NewInt(rand.Intn(100))
		a.Add(newInt)
	}
	q := NewArrayQueue()
	q.array = a
}

func TestArrayQueue(t *testing.T) {
	qq := NewArrayQueue()
	qq.Enqueue(NewInt(1))
	qq.Enqueue(NewInt(2))
	qq.Enqueue(NewInt(3))
	qq.Enqueue(NewInt(4))
	fmt.Println(qq)
	qq.Dequeue()
	fmt.Println(qq)
	qq.Dequeue()
	fmt.Println(qq)
	fmt.Println(qq.Size())
	fmt.Println(qq.IsEmpty())
	fmt.Println(qq.GetFront())
}


输出：
数组长度 : 4 【 1  2  3  4   】
数组长度 : 3 【 2  3  4   】
数组长度 : 2 【 3  4   】
2
false
3
```


#### 数组队列实现(JAVA)

###### 定义通用队列结构

```
public interface Queue<E> {

    int getSize();
    boolean isEmpty();
    void enqueue(E e);
    E dequeue();
    E getFront();
}
```

###### 定义自定义动态数组实现

```

public class Array<E> {

    private E[] data;
    private int size;

    // 构造函数，传入数组的容量capacity构造Array
    public Array(int capacity){
        data = (E[])new Object[capacity];
        size = 0;
    }

    // 无参数的构造函数，默认数组的容量capacity=10
    public Array(){
        this(10);
    }

    // 获取数组的容量
    public int getCapacity(){
        return data.length;
    }

    // 获取数组中的元素个数
    public int getSize(){
        return size;
    }

    // 返回数组是否为空
    public boolean isEmpty(){
        return size == 0;
    }

    // 在index索引的位置插入一个新元素e
    public void add(int index, E e){

        if(index < 0 || index > size)
            throw new IllegalArgumentException("Add failed. Require index >= 0 and index <= size.");

        if(size == data.length)
            resize(2 * data.length);

        for(int i = size - 1; i >= index ; i --)
            data[i + 1] = data[i];

        data[index] = e;

        size ++;
    }

    // 向所有元素后添加一个新元素
    public void addLast(E e){
        add(size, e);
    }

    // 在所有元素前添加一个新元素
    public void addFirst(E e){
        add(0, e);
    }

    // 获取index索引位置的元素
    public E get(int index){
        if(index < 0 || index >= size)
            throw new IllegalArgumentException("Get failed. Index is illegal.");
        return data[index];
    }

    public E getLast(){
        return get(size - 1);
    }

    public E getFirst(){
        return get(0);
    }

    // 修改index索引位置的元素为e
    public void set(int index, E e){
        if(index < 0 || index >= size)
            throw new IllegalArgumentException("Set failed. Index is illegal.");
        data[index] = e;
    }

    // 查找数组中是否有元素e
    public boolean contains(E e){
        for(int i = 0 ; i < size ; i ++){
            if(data[i].equals(e))
                return true;
        }
        return false;
    }

    // 查找数组中元素e所在的索引，如果不存在元素e，则返回-1
    public int find(E e){
        for(int i = 0 ; i < size ; i ++){
            if(data[i].equals(e))
                return i;
        }
        return -1;
    }

    // 从数组中删除index位置的元素, 返回删除的元素
    public E remove(int index){
        if(index < 0 || index >= size)
            throw new IllegalArgumentException("Remove failed. Index is illegal.");

        E ret = data[index];
        for(int i = index + 1 ; i < size ; i ++)
            data[i - 1] = data[i];
        size --;
        data[size] = null; // loitering objects != memory leak

        if(size == data.length / 4 && data.length / 2 != 0)
            resize(data.length / 2);
        return ret;
    }

    // 从数组中删除第一个元素, 返回删除的元素
    public E removeFirst(){
        return remove(0);
    }

    // 从数组中删除最后一个元素, 返回删除的元素
    public E removeLast(){
        return remove(size - 1);
    }

    // 从数组中删除元素e
    public void removeElement(E e){
        int index = find(e);
        if(index != -1)
            remove(index);
    }

    @Override
    public String toString(){

        StringBuilder res = new StringBuilder();
        res.append(String.format("Array: size = %d , capacity = %d\n", size, data.length));
        res.append('[');
        for(int i = 0 ; i < size ; i ++){
            res.append(data[i]);
            if(i != size - 1)
                res.append(", ");
        }
        res.append(']');
        return res.toString();
    }

    // 将数组空间的容量变成newCapacity大小
    private void resize(int newCapacity){

        E[] newData = (E[])new Object[newCapacity];
        for(int i = 0 ; i < size ; i ++)
            newData[i] = data[i];
        data = newData;
    }
}

```

###### 自定义数组队列实现

```
public class ArrayQueue<E> implements Queue<E> {

    private Array<E> array;

    public ArrayQueue(int capacity){
        array = new Array<>(capacity);
    }

    public ArrayQueue(){
        array = new Array<>();
    }

    @Override
    public int getSize(){
        return array.getSize();
    }

    @Override
    public boolean isEmpty(){
        return array.isEmpty();
    }

    public int getCapacity(){
        return array.getCapacity();
    }

    @Override
    public void enqueue(E e){
        array.addLast(e);
    }

    @Override
    public E dequeue(){
        return array.removeFirst();
    }

    @Override
    public E getFront(){
        return array.getFirst();
    }

    @Override
    public String toString(){

        StringBuilder res = new StringBuilder();
        res.append("Queue: ");
        res.append("front [");
        for(int i = 0 ; i < array.getSize() ; i ++){
            res.append(array.get(i));
            if(i != array.getSize() - 1)
                res.append(", ");
        }
        res.append("] tail");
        return res.toString();
    }

    public static void main(String[] args){

        ArrayQueue<Integer> queue = new ArrayQueue<>();
        for(int i = 0 ; i < 10 ; i ++){
            queue.enqueue(i);
            System.out.println(queue);

            if(i % 3 == 2){
                queue.dequeue();
                System.out.println(queue);
            }
        }
    }
}
```

##### 循环实现

```
public class LoopQueue<E> implements Queue<E> {

    private E[] data;
    private int front, tail;
    private int size;

    public LoopQueue(int capacity){
        // 当容量达到 tail + 1 等于front的时候就需要扩容了，所以需要浪费一个空间，所以这儿就加 1
        data = (E[])new Object[capacity + 1];
        front = 0;
        tail = 0;
        size = 0;
    }

    public LoopQueue(){
        this(10);
    }

    public int getCapacity(){
        // 当容量达到 tail + 1 等于front的时候就需要扩容了，所以需要浪费一个空间，所以这儿容积需要减1
        return data.length - 1;
    }

    @Override
    public boolean isEmpty(){
        return front == tail;
    }

    @Override
    public int getSize(){
        return size;
    }

    @Override
    public void enqueue(E e){

        if((tail + 1) % data.length == front)
            resize(getCapacity() * 2);

        data[tail] = e;
        tail = (tail + 1) % data.length;
        size ++;
    }

    @Override
    public E dequeue(){

        if(isEmpty())
            throw new IllegalArgumentException("Cannot dequeue from an empty queue.");

        E ret = data[front];
        data[front] = null;
        front = (front + 1) % data.length;
        size --;
        if(size == getCapacity() / 4 && getCapacity() / 2 != 0)
            resize(getCapacity() / 2);
        return ret;
    }

    @Override
    public E getFront(){
        if(isEmpty())
            throw new IllegalArgumentException("Queue is empty.");
        return data[front];
    }

    private void resize(int newCapacity){

        E[] newData = (E[])new Object[newCapacity + 1];
        for(int i = 0 ; i < size ; i ++)
            newData[i] = data[(i + front) % data.length];

        data = newData;
        front = 0;
        tail = size;
    }

    @Override
    public String toString(){

        StringBuilder res = new StringBuilder();
        res.append(String.format("Queue: size = %d , capacity = %d\n", size, getCapacity()));
        res.append("front [");
        for(int i = front ; i != tail ; i = (i + 1) % data.length){
            res.append(data[i]);
            if((i + 1) % data.length != tail)
                res.append(", ");
        }
        res.append("] tail");
        return res.toString();
    }

    public static void main(String[] args){

        LoopQueue<Integer> queue = new LoopQueue<>();
        for(int i = 0 ; i < 10 ; i ++){
            queue.enqueue(i);
            System.out.println(queue);

            if(i % 3 == 2){
                queue.dequeue();
                System.out.println(queue);
            }
        }
    }
}

```


