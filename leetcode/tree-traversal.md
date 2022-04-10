---
layout: page
title: 树的遍历
description: >
  树的遍历
hide_description: false
sitemap: true
---

* toc
{:toc .large-only}

## 二叉树

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

```

## 新建二叉树
传入一个或多个遍历，新建一颗二叉树。
输入的遍历字符串如:

```
[1,null,2,3]
```

### 从前序和中序遍历新建二叉树

比较难想出来的是下面这行代码：

```
  root_node.right = new_tree_from_array_ldr_and_pre(ldr[root_index_on_ldr+1:], pre[root_index_on_ldr+1:])
```
在获取右子树的父节点时，刚好是前序遍历的第`root_index_on_ldr`个元素。

```python
# ldr_str and pre_str are strings like "[1,null,2,3]" with ldr/中序 and pre/先序
def new_tree_from_str_ldr_and_pre(ldr_str,pre_str):
    str_len = len(ldr_str)
    ldr_str = ldr_str[1:str_len-1]
    pre_str = pre_str[1:str_len-1]
    return new_tree_from_array_ldr_and_pre(ldr_str.split(','),pre_str.split(','))

# ldr_str and pre_str are strings array like ["1","null","2","3"] with ldr/中序 and pre/先序
def new_tree_from_array_ldr_and_pre(ldr,pre):
    # traceback exit condition
    if len(pre)==0 or len(ldr) == 0:
        return
    # print("pre: %v \n",pre)
    # print("ldr: %v \n",ldr)
    root_val,root_node =pre[0],TreeNode(pre[0])
    root_index_on_ldr = ldr.index(root_val)

    # build
    root_node.left = new_tree_from_array_ldr_and_pre(ldr[0:root_index_on_ldr], pre[1:])
    root_node.right = new_tree_from_array_ldr_and_pre(ldr[root_index_on_ldr+1:], pre[root_index_on_ldr+1:])

    return root_node
```

### 从层序遍历新建二叉树

```python

# build a tree from level str likes '[1,null,2,3]'
def new_tree_from_level_str(level_str):
    level_array = level_str[1:len(level_str)-1].split(',')
    pre_node_q = []
    pre_node_q.append(TreeNode(level_array.pop(0)))
    return_root = pre_node_q[0]

    while len(level_array) != 0:
        # left
        elem = level_array.pop(0)
        if elem != 'null':
            t = TreeNode(elem)
            pre_node_q.append(t)
            # print("l is ",elem)
            pre_node_q[0].left = t
         
        if len(level_array)==0:
            break
        
        # right 
        elem = level_array.pop(0)
        if elem != 'null':
            t = TreeNode(elem)
            pre_node_q.append(t)
            # print("r is ",elem)

            pre_node_q[0].right = t

        # next pre_node
        pre_node_q.pop(0)   

    return return_root    

```

## 前/中/后序遍历

leetcode #94 #144 #145
使用递归很简单。

```python

class Solution:
    def Traversal(self, root: Optional[TreeNode]) -> List[str]:
        if root is None:
            return []
        result = []
        self.Trace(root,result)
        return result

    def Trace(self,root: Optional[TreeNode],result):
        if root is None:
            return
        result.append(root.val) #前序
        self.preTrace(root.left,result)
        # result.append(root.val) #中序
        self.preTrace(root.right,result)
        # result.append(root.val) #后序

```


### 层序遍历
leetcode #102
使用两个队列，存储上一层和当前层的节点，然后遍历上一层节点，取出`val`并`append`其左右子节点。

```
class Solution:
    def levelOrder(self, root: TreeNode) -> List[List[str]]:
        if root is None:
            return []
        res, segRes = [],[]
        preNodeq,curNodeq = [],[]

        preNodeq.append(root)
        res.append([root.val])
        while len(preNodeq) != 0:
            t = preNodeq.pop(0)

            if t.left is not None:
                curNodeq.append(t.left)
                segRes.append(t.left.val)
            if t.right is not None:
                curNodeq.append(t.right)
                segRes.append(t.right.val)

#                 next level
            if len(preNodeq) is 0 and len(segRes) is not 0:
                res.append(segRes)
                segRes = []
                preNodeq,curNodeq = curNodeq,[]

        return res
```
