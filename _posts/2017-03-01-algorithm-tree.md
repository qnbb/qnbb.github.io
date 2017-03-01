---
layout: post
title: 重建二叉树
categories: [Algorithm]
description: 重建二叉树
keywords: 二叉树
---
### 重建二叉树

题目描述
输入某二叉树的前序遍历和中序遍历的结果，请重建出该二叉树。假设输入的前序遍历和中序遍历的结果中都不含重复的数字。例如输入前序遍历序列{1,2,4,7,3,5,6,8}和中序遍历序列{4,7,2,1,5,3,8,6}，则重建二叉树并返回。

```
/**
 * Definition for binary tree
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
import java.util.HashMap;

public class Solution {
    public TreeNode reConstructBinaryTree(int [] pre,int [] in) {
        
        if (pre == null || in == null)
            return null;
        
        //当条件有不含重复的数字时，必须要考虑下HashMap
        //将中序数组，放入HashMap，方便取 和前序同值的位置
        java.util.HashMap<Integer,Integer> hs = new java.util.HashMap<Integer,Integer>();
        for (int i = 0;i < in.length;i++) {
            hs.put(in[i],i);	//值当key，位置i当value，方便取值
        }
        
		
        //参数含义：
        //pre （前序数组），0（前序数组起始位置），pre.length - 1（前序数组末尾位置）
        //in（中序数组），0（中序数组起始位置），in.length - 1（中序数组末尾位置）
        TreeNode head = treeInPreIn(pre,0,pre.length - 1,in,0,in.length - 1,hs);
        
        //最终返回的头节点
        return head;
        
    }
    
     public TreeNode treeInPreIn(int[] pre,int p1,int p2,int[] in,int in1 ,int in2,java.util.HashMap<Integer,Integer> hs) {
         
        //递归退出条件，当只有一个节点的时候，p1 = p1 + 1,p2 == p1,所以 p1 > p2
        if (p1 > p2)
            return null;
         //局部变量，包含在栈里面
         TreeNode tempHead = new TreeNode(pre[p1]);
         
         //找出与pre[p1]相等的中序位置
         int m = hs.get(pre[p1]);
         //p1 + 1:头节点 下一个
         //p1 + (m - in1):不是直接 m,当 p1 != in1时
         tempHead.left = treeInPreIn(pre,p1 + 1,p1 + (m - in1),in,in1,m - 1,hs);
         
         //p1 + (m - in1) + 1，其实就是 前序最后一个节点，的下一个
         tempHead.right = treeInPreIn(pre,p1 + m - in1 + 1,p2 ,in,m + 1,in2,hs); //p1 + m - in1 + 1 !!!
         
         return tempHead;
    }
}
```
