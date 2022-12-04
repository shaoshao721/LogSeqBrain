tags:: 图

- dijkstra是为了计算从一个起点到其他节点的最短距离，返回记录最短路径权重的数组。
- 算法框架
- ```
  class State{
      int id;
      
      // 从start节点到当前节点的距离
      int disFromSatrt;
      
      State(int id, int distFromStart) {
          this.id = id;
          this.disFromSatrt = distFromStart;
      }
  }
  
  class DijkstraClass{
      int weight(int from, int to);
      
      List<Integer> ads(int s);
      
      int[] dijkstra(int start, List<Integer>[] graph) {
          int V = graph.length;
          
          int[] distTo = new int[V];
  
          Arrays.fill(distTo, Integer.MAX_VALUE);
          distTo[start] = 0;
          
          Queue<State> pq = new PriorityQueue<>((a,b) ->{
              return a.disFromSatrt - b.disFromSatrt;
          });
          
          pq.offer(new State(start, 0));
          
          while (!pq.isEmpty()) {
              State curState = pq.poll();
              int curNodeId = curState.id;
              int curDistFromStart = curState.disFromSatrt;
              
              if(curDistFromStart > distTo[curNodeId]) {
                  continue;
              }
              
              for(int nextNodeId : adj(curNodeId)) {
                  int distToNextNode = distTo[curNodeId] + weight(curNodeId, nextNodeId);
                  
                  if(distTo[nextNodeId] > distToNextNode) {
                      distTo[nextNodeId]= distToNextNode;
                      pq.offer(new State(nextNodeId, distToNextNode));
                  }
              }
          }
          return distTo;
      }
   }
  ```