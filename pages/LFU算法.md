tags:: LFU，算法秘籍，数据结构

- TODO 重做460 .
  SCHEDULED: <2022-11-14 Mon>
- LFU定义
	- 一种缓存淘汰算法，删除最近使用频次最低的元素。当有多个元素都有着相同的频次的时候，就删除最早的那个数据
- 基本思路
	- 当get的时候，如果有，则将频次+1并返回value。当没有这个key的时候，直接返回-1
	- 当put的时候，如果已经有这个key，重新put覆盖新的value，并把key的频次+1，return。如果没有这个key，需要插入一个新的key，这时候需要判断是否满了，需要执行缓存淘汰算法，如果满了，将频率最小的key淘汰，并将这个key插入进去。
	- 需要o(1)的时间复杂度来实现
		- 通过key需要直接找到value，map keyToVal
		- 通过key直接找到频次，map keyToFreq
		- 当执行淘汰算法的时候，首先要知道频率最小的是啥？其次，频率最小的可能不止一个，若有多个的情况下，需要排除掉最早的数据。而且还需要快速删除掉key列表里的任何一个key，因为当freq+1的时候，要从这个链中把这个key删除掉
			- 频率最小的，要维持一个minFreq
			- 频率对应的key，应该是个list，且要有时序性，LinkedHashSet
- 代码
	- ```
	  import java.util.HashMap;
	  import java.util.LinkedHashSet;
	  import java.util.Map;
	  
	  class LFUCache {
	      int capacity;
	      int minFreq;
	      Map<Integer, Integer> keyToVal;
	      Map<Integer, Integer> keyToFreq;
	      Map<Integer, LinkedHashSet<Integer>> freqToKeys;
	  
	      public LFUCache(int capacity) {
	          this.capacity = capacity;
	          minFreq = 0;
	          keyToVal = new HashMap<>();
	          keyToFreq = new HashMap<>();
	          freqToKeys = new HashMap<>();
	      }
	      
	      public int get(int key) {
	          if(!keyToVal.containsKey(key)) {
	              return -1;
	          }
	          increaseFreqCnt(key);
	          return keyToVal.get(key);
	      }
	      
	      public void put(int key, int value) {
	          if(capacity <= 0) return;
	          if(keyToVal.containsKey(key)) {
	              keyToVal.put(key, value);
	              increaseFreqCnt(key);
	              return;
	          }
	          if (capacity <= keyToVal.size()) {
	              removeMinFreqKey();
	          }
	          //新增数据
	          keyToVal.put(key, value);
	          keyToFreq.put(key, 1);
	          freqToKeys.putIfAbsent(1, new LinkedHashSet<>());
	          freqToKeys.get(1).add(key);
	          minFreq = 1;
	      }
	      
	      void removeMinFreqKey() {
	          LinkedHashSet<Integer> set = freqToKeys.get(minFreq);
	          int deleteKey = set.iterator().next();
	          set.remove(deleteKey);
	          if(set.isEmpty()) {
	              freqToKeys.remove(minFreq);
	          }
	          keyToVal.remove(deleteKey);
	          keyToFreq.remove(deleteKey);
	      }
	      
	      void increaseFreqCnt(int key) {
	          int freq = keyToFreq.get(key);
	          freqToKeys.get(freq).remove(key);
	          if(freqToKeys.get(freq).isEmpty()) {
	              freqToKeys.remove(freq);
	              if(minFreq == freq) {
	                  minFreq++;
	              }
	          }
	          freqToKeys.putIfAbsent(freq + 1, new LinkedHashSet<>());
	          freqToKeys.get(freq + 1).add(key);
	          
	          keyToFreq.put(key, freq + 1);
	      }
	  }
	  
	  /**
	   * Your LFUCache object will be instantiated and called as such:
	   * LFUCache obj = new LFUCache(capacity);
	   * int param_1 = obj.get(key);
	   * obj.put(key,value);
	   */
	  ```