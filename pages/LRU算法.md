tags:: 数据结构，算法秘籍，LRU

- TODO 146LRU算法
  SCHEDULED: <2022-11-14 Mon>
-
- LRU定义
	- 一种缓存淘汰算法，least recently used，删除最近最久未使用的元素
- 要求
	- 接收一个capacity的正整数作为容量
	- put和get的方法都必须是o(1)的时间复杂度
	- put的时候就直接插入到队列的最后面
	- get的时候，返回的时候还需要把这个元素拿出来再放到最后面去。
	- 数据结构需要快速找到key的val值，且要支持在任意位置快速进行插入和删除元素
	- 哈希链表！
- 步骤
	- 首先要建立一个结构体，Node，需要包括key，val，以及前后节点。作为双向链表的底层结构，为啥要双向链表呢？因为方便删除和插入
		- ```
		  class Node {
		      public int key, value;
		      public Node prev, next;
		  
		      public Node(int key, int value) {
		          this.key = key;
		          this.value = value;
		      }
		  }
		  ```
- 实现这个双向链表的基本方法
	- 初始化，需要定义head，tail并将他俩关联上
	- 在队列尾部插入一个node
	- 删除一个node
	- 在队首删除一个node
	- 返回这个链表的size
	- 实现
		- ```
		  class DoubleList {
		      private int size;
		      private Node head;
		      private Node tail;
		  
		      public DoubleList() {
		          size = 0;
		          head = new Node(0, 0);
		          tail = new Node(0, 0);
		          head.next = tail;
		          tail.prev = head;
		      }
		  
		      public void addLast(Node x) {
		          x.prev = tail.prev;
		          x.next = tail;
		          tail.prev.next = x;
		          tail.prev = x;
		          size++;
		      }
		  
		      public void remove(Node x) {
		          x.prev.next = x.next;
		          x.next.prev = x.prev;
		          size--;
		      }
		  
		      public Node removeFist() {
		          if(head.next == tail) {
		              return null;
		          }
		          Node first = head.next;
		          remove(first);
		          return first;
		      }
		  
		      public int size() {
		          return size;
		      }
		  }
		  ```
- 整体LRU的实现
	- ```
	  class LRUCache {
	      int capacity;
	      Map<Integer, Node> map;
	      DoubleList queue;
	  
	      public LRUCache(int capacity) {
	          this.capacity = capacity;
	          map = new HashMap<>();
	          queue = new DoubleList();
	      }
	      
	      public int get(int key) {
	          // 如果map里包含这个key，将这个key放到队列的最后，并返回这个key
	          // 如果不存在这个key，就直接返回-1
	          if(!map.containsKey(key)) {
	              return -1;
	          }
	          makeRecently(key);
	          return map.get(key).value;
	      }
	      
	      public void put(int key, int value) {
	          // 如果队列里已经有这个key，那把这个元素删除掉，并将元素放到队列最后
	          if(map.containsKey(key)) {
	              deleteKey(key);
	              addRecently(key, value);
	              return;
	          }
	          // 如果没有这个key，那么要看队列满了吗？
	          // 如果队列满了，从队首删除一个元素
	          // 统一从队列最后插入一个元素
	          if(capacity == queue.size()) {
	              removeLeastRecently();
	          }
	          
	          addRecently(key, value);
	      }
	  
	      private void makeRecently(int key) {
	          Node x = map.get(key);
	  
	          queue.remove(x);
	          queue.addLast(x);
	      }
	  
	      private void addRecently(int key, int val) {
	          Node x = new Node(key, val);
	          queue.addLast(x);
	          map.put(key, x);
	      }
	  
	      private void deleteKey(int key) {
	          Node x = map.get(key);
	          queue.remove(x);
	          map.remove(key);
	      }
	  
	      private void removeLeastRecently() {
	          Node deleteNode = queue.removeFist();
	          map.remove(deleteNode.key);
	      }
	  }
	  ```
-