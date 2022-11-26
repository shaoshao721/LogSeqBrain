tags:: 二叉树，二叉搜索树

- 题目：
	- 给你输入一个正整数 `n` ，请你计算，存储 `{1,2,3...,n}` 这些值共有有多少种不同的 BST 结构。
	- 因为不用计算出具体的值，所以用个备忘录就可以算出来
	- ```
	  class Solution {
	      public int numTrees(int n) {
	          int[] dp = new int[n + 1];
	          dp[0] = 1;
	          dp[1] = 1;
	          for(int i = 2; i <= n; i++) {
	              for(int j = 0; j < i; j++) {
	                  dp[i] += dp[j] * dp[i - j - 1];
	              }
	          }
	          return dp[n];
	      }
	  }
	  ```
- 进阶：要把所有的子树都排列出来
	- ```
	  class Solution {
	      public List<TreeNode> generateTrees(int n) {
	          if(n == 0) {
	              return new LinkedList<>();
	          }
	          return build(1, n);
	      }
	  
	      List<TreeNode> build(int left, int right) {
	          List<TreeNode> res = new LinkedList<>();
	          if(left > right) {
	              res.add(null);
	              return res;
	          }
	          for(int i = left; i <= right; i++) {
	              List<TreeNode> leftTree = build(left, i - 1);
	              List<TreeNode> rightTree = build(i + 1, right);
	              for(TreeNode leftNode : leftTree) {
	                  for(TreeNode rightNode : rightTree) {
	                      TreeNode root = new TreeNode(i);
	                      root.left = leftNode;
	                      root.right = rightNode;
	                      res.add(root);
	                  }
	              }
	          }
	          return res;
	      }
	  }
	  ```