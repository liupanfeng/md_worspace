### 链表实现LRU算法

#### 1.缓存

CPU缓存：位于CPU与内存之间的临时存储器。解决CPU速度和内存速度的速度差异问题。

软件缓存：内存缓存、本地缓存、网络缓存

#### 2.内存缓存

预先将数据写到了容器(list,map,set)等数据存储单元中，就是软件内存缓存。

### 3.内存缓存淘汰机制

* FIFO （First In, First Out） 先进的元素，先被淘汰，队列
* LFU     (Least Frequently Used)  使用频率最低 先被淘汰  记录元素的使用频率
* LRU    (Least Recently Used) 最近最少使用

#### 4.LRU 算法

* 新数据插入到链表头部
* 当缓存命中(即缓存数据被访问)，数据要移到表头
* 当链表满的时候，将链表尾部的数据丢弃

#### 5.实现单链表以及LRU

```java
package com.enjoyedu.struct;

public class MSLinkedList<T> {
    //首节点
    Node list;
    //大小
    int size;
    /**
     * 链表头部添加元素
     * @param data
     */
    public void put(T data) {
        Node head=list;
        Node currentNode=new Node(list,data);
        list=currentNode;
        size++;
    }
    /**
     * index 位置添加元素
     * @param index
     * @param data
     */
    public void put(int index, T data) {
        checkPositionIndex(index);
        //记录头节点
        Node head=list;
        //记录当前的位置
        Node cur=list;

        //链表的表里，，找位置的通用方法  ，声明两个节点 一个节点=另外一个节点  另外一个节点进行next遍历

        for (int i = 0; i < index; i++) {
            head=cur;
            cur=cur.next;
        }
        //node 节点 next 指向cur节点
        Node node= new Node(cur,data);
        //head节点的next 指向node
        head.next=node;
        size++;

    }
    /**
     * 删除头节点
     * @return
     */
    public T remove() {
        if (list!=null){
            Node node=list;
            list=list.next;
            node.next=null;   //让first节点指向空 GC回收
            size--;
            return node.data;
        }
        return null;
    }
    /**
     * 删除指定位置的元素
     * @param index
     * @return
     */
    public T remove(int index) {
        checkPositionIndex(index);
        Node head=list;
        Node cur=list;
        for (int i = 0; i < index; i++) {
            head=cur;
            cur=cur.next;
        }
        head.next=cur.next;
        cur.next=null;
        size--;
        return cur.data;
    }
    /**
     * 删除链表最后一个元素
     * @return
     */
    public T removeLast() {
        Node head=list;
        Node cur=list;
        for (int i = 0; i < size; i++) {
            head=cur;
            cur=cur.next;
        }
        head.next=null;
        size--;
        return cur.data;
    }
    /**
     * 更改index位置的元素
     * @param index
     * @param newData
     */
    public void set(int index, T newData) {
        checkPositionIndex(index);
        Node cur=list;
        for (int i = 0; i < index; i++) {
            cur=cur.next;
        }
        cur.data=newData;
    }
    //获取头节点数据
    public T get() {
        Node node=list;
        if (node!=null){
            return node.data;
        }else{
            return null;
        }
    }

    public T get(int index) {
        checkPositionIndex(index);
        Node cur=list;
        for (int i = 0; i < index; i++) {
            cur=cur.next;
        }
        return cur.data;
    }

    /**
     * 位置检查
     * @param index
     */
    public void checkPositionIndex(int index) {
        if (!(index>=0&&index<=size)){
            throw new IndexOutOfBoundsException("index "+index+"is out of size="+size);
        }
    }

    @Override
    public String toString() {
        Node cur=list;
        for (int i = 0; i < size; i++) {
            System.out.println(" "+cur.data);
            cur=cur.next;
        }
        System.out.println("----------");
        return super.toString();
    }

    //先申明一个node节点
    class Node {
        Node next;
        T data;

        public Node(Node next, T data) {
            this.next = next;
            this.data = data;
        }
    }
}

public class MSLruLinkedList<T> extends MSLinkedList<T> {

    //缓存大小
    int memSize;
    static final int DEFAULT_CAP = 5;

    public MSLruLinkedList() {
        this(DEFAULT_CAP);
    }

    public MSLruLinkedList(int memSize) {
        this.memSize = memSize;
    }

    //向缓存池放置元素   在链表的头部添加元素  注意缓存池的容量  如果超了删除尾部元素 然后再添加
    public void lruPut(T data) {
        if (size >= memSize) {
            put(data);
            removeLast();
        } else {
            put(data);
        }
    }

    //删除缓存元素  删除链表尾部的元素
    public T lruRemove() {
        return removeLast();
    }

    //获取缓存元素  拿到指定位置的元素  并将这个元素的位置放置到链表的头部
    public T lruGet(int index) {
        checkPositionIndex(index);
        Node head = list;
        Node pre = list;
        Node cur = list;
        for (int i = 0; i < index; i++) {
            pre = cur;
            cur = cur.next;
        }

        cur.next = head;
        pre.next = cur.next;
        list = cur;  //更新表头的位置

        return cur.data;
    }

}

```

