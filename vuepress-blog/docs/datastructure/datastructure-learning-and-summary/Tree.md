> About Tree

树🌲，是一种用于存储**非顺序**的数据结构（跟集合和字典一致都是存储无序数据），但可用于存储相同数据（这也是跟集合和字典不一样的地方）。

一个树结构包含一系列存在父子关系的节点。每个节点都有一个父节点（当然，除了顶部的第一个节点以外）以及至少0个子节点。示例图如下：



![About-Tree](/Users/andraw-lin/Mine/Personal/FE_Images/Data-Structure/About-Tree.jpg)



树中每个元素都叫作节点，节点又分为内部节点以及外部节点，例如：内部节点就有7、5、9、15、13、20，外部节点就有3、6、8、12、14、18、25，而11则是根节点。子树有节点和它的后代构成，例如：20、18、25就构成了一个子树。

节点的深度取决于它的祖先节点的数量，简单滴说：**节点的深度 = 该节点的祖先节点数量**，例如：节点3有3个祖先节点，因此它的深度为3。

树的深度取决于所有节点深度的最大值，看图已在左侧标出，看到的是根节点是在第0层开始，到第三层叶节点。因此树的高度为3。



---

> Create BinarySearchTree

日常开发中，用的最多的无非就是二叉树。**二叉树的节点最多只能有两个子节点：一个左侧子节点和一个右侧子节点**。

**二叉搜索树（BST）是二叉树的一种，它规定在左侧节点存储比父节点小的值，而在右侧节点存储比父节点大或者相等的值**。接下来就会实现一个二叉搜索树，一个基本的二叉搜索树骨架包括如下：

```javascript
function BinarySearchTree() {
  var Node = function(key) {		// 用于创建树节点的私有类
    this.key = key
    this.left = null
    this.right = null
  }
  var root = null		// 用于存储二叉搜索树的根节点（有点类似于链表的Head）
  this.insert = function(key) { ... }		// 向树中插入一个新的键
  this.search = function(key) { ... }		// 在树中查找一个键，如果节点存在，则返回true，否则返回false
  this.inOrderTraverse = function() { ... }		// 通过中序遍历方式来遍历树中所有节点
  this.preOrderTraverse = function() { ... }		// 通过先序遍历方式来遍历树中所有节点
  this.postOrderTraverse = function() { ... }		//通过后序遍历方式来遍历树中所有节点
  this.min = function() { ... }		// 返回树中最小的值/键
  this.max = function() { ... }		// 返回树中最大的值/键
  this.remove = function(key) { ... }		// 从树中移除某个键
}
```

在实现遍历的方式时，由于每个节点的结构基本都会是至多包含两个字节点，因此结构上来说是基本一致，方便后续使用递归的形式进行。

1. insert

   在向树中插入一个元素时，需要考虑一个问题就是，如果使用遍历的形式，由于根节点会存在左右两个字节点，因此使用遍历的话会耗时间而且还会很容易出错。因此该使用递归的形式。

   ```javascript
   this.insert = function(key) {
     let elementNode = new Node(key)
     if (root === null) {
       root = elementNode
     } else {
       insertNode(root, elementNode)	// 用于递归的函数方法
     }
   }
   let insertNode = function(root, elementNode) {
     if (root.key > elementNode.key) {
       if (root.left === null) {
       	root.left = elementNode
       } else {
         insertNode(root.left, elementNode)	// 递归时，当发现左子节点比添加节点的值大时，就继续递归下去
       }
     } else {
       if (root.right === null) {
         root.right = elementNode
       } else {
         insertNode(root.right, elementNode)	// 递归时，当发现右子节点比添加节点的值小时，就继续递归下去
       }
     }
   }
   ```

2. search

   查询元素时，同样也是使用递归的形式进行查询，当查询Key比根节点的值小时，直接递归根节点的左子树即可，大时则递归根节点的右子树即可。

   ```javascript
   this.search = function(key) {
     let elementNode = new Node(key)
     if (root === null) {
       return false
     } else {
       return searchNode(root, elementNode)
     }
   }
   let searchNode = function(root, elementNode) {
     if (root.key > elementNode.key) {
       if (root.left === null) {
         return false
       } else {
        	return searchNode(root.left, elementNode)
       }
     } else if (root.key < elementNode.key) {
       if (root.right === null) {
         return false
       } else {
         return searchNode(root.left, elementNode)
       }
     } else {
       return true
     }
   }
   ```

3. inOrderTraverse**（中序遍历）**

   中序遍历是一种**从下往上从左往右进行遍历所有节点的方式，也就是从最小到最大的顺序访问所有节点**。
   
   ```javascript
   this.inOrderTraverse = function(callback) {
     inOrderTraverseNode(root, callback)
   }
   ```
   
   `inOrderTraverse`方法接收一个回调函数作为参数，回调函数是便于用来定义我们在遍历到的每一个节点时所进行的操作（也叫访问者模式）。**在BST中最常实现的算法就是递归**。因此`inOrderTraverse`方法就是用来定义的递归方法，也作为一个私有的辅助方法。
   
   ```javascript
   let inOrderTraverseNode = function(root, callback) {
     if (root !== null) {
       inOrderTraverseNode(root.left, callback)
       callback(root.key)
       inOrderTraverseNode(root.right, callback)
     }
   }
   ```
   
   代码使用递归起来很简单，从理解上，会一直向树中的左节点执行，直到找到最小的节点为止，然后执行callback，然后返回上一层执行最小节点的父节点callback，再执行其父节点的右节点，以此类推。示意图如下：
   
   ![中序遍历](/Users/andraw-lin/Library/Application Support/typora-user-images/image-20190421234600501.png)
   
4. preOrderTraverse（先序遍历）

   先序遍历是一种**先访问节点本身，然后访问其左节点，最后再访问其右节点**。

   ```javascript
   this.preOrderTraverse = function(callback) {
     preOrderTraverse(root, callback)
   }
   let preOrderTraverse = function(root, callback) {
     callback(root.key)
     preOrderTraverse(root.left, callback)
     preOrderTraverse(root.right, callback)
   }
   ```

   示意图如下：

   ![](http://ppu8vcpyg.bkt.clouddn.com/preOrderTraverse.png)

   ![preOrderTraverse](/Users/andraw-lin/Mine/Personal/FE_Images/Data-Structure/preOrderTraverse.png)

   

5. postOrderTraverse（后序遍历）

   后序遍历的思想就是**从节点本身开始，先执行其左节点，再执行其右节点，最后才执行其节点本身**（即优先执行子节点，再执行节点本身）。后续遍历的一种应用就是计算一个目录和它的子目录中所有文件所占空间的大小。

   ```javascript
   this.postOrderTraverse = function(callback) {
     postOrderTraverseNode(root, callback)
   }
   let postOrderTraverseNode = function(root, callback) {
     if (root !== null) {
       postOrderTraverseNode(root.left, callback)
       postOrderTraverseNode(root.right, callback)
       callback(root.key)
     } 
   }
   ```

   示意图如下：

   ![后序遍历](http://ppu8vcpyg.bkt.clouddn.com/postOrderTraverse.png)

   ![postOrderTraverse](/Users/andraw-lin/Mine/Personal/FE_Images/Data-Structure/postOrderTraverse.png)

   

6. min

   搜索二叉树中的最小节点，我们知道，在二叉搜索树中，小的值都会放到节点的左子节点中，大的值都会放到节点的右节点中。因此搜索树中的最小值，只需要一直向左搜即可。

   ```javascript
   this.min = function() {
     return minNode(root)
   }
   let minNode = function(node) {
     if (node) {
       while(node.left) {
       	node = node.left
       }
       return node.key
     }
     return null
   }
   ```

7. max

   与min方法一致，只需一直往树中的右节点搜索即可。

   ```javascript
   this.max = function() {
     return maxNode(root)
   }
   let maxNode = function(node) {
     if (node) {
       while(node.right) {
         node = node.right
       }
       return node.key
     }
     return null
   }
   ```

8. search

   针对搜索树中一个特定的Key值时，依然需要使用递归形式，判断值比节点的值小时，就递归搜索左子树，值比节点的值大时，就递归搜索右子树。

   ```javascript
   this.search = function(key) {
     return searchNode(root, key)
   }
   let searchNode = function(node, key) {
     if (node === null) {
       return false
     }
     if (node.key > key) {
       return searchNode(node.left, key)
     } else if (node.key < key) {
       return searchNode(node.right, key)
     } else {
       return true
     }
   }
   ```

9. remove

   移除树中的某个节点，需判断三种情况：第一种就是无子节点的节点，第二种就是只有一个子节点的节点，第三种就是有两个子节点的节点。最复杂的就是第三种，在处理上，移除时需找到其节点右子树中最小的节点与其进行替换。

   ```javascript
   this.remove = function(key) {
   	return removeNode(root, key)
   }
   let removeNode = function(node, key) {
     if (node === null) {
       return null
     }
     if (node.key > key) {
       return removeNode(node.left, key)
     } else if (node.key < key) {
       return removeNode(node.right, key)
     } else {
       // 第一种情况：当无子节点时
       if (node.left === null && node.right === null) {
         node = null
         return node
       }
       // 第二种情况：当只有一个子节点时
       if (node.right === null) {
         node = node.left
         return node
       }
       if (node.left === null) {
        	node = node.right
         return node
       }
       // 第三种情况：当有两个子节点时
       let minNode = minNode(node.right)
       node.key = minNode.key
       node.right = removeNode(node.right, minNode.key)
       return node
     }
   }
   ```

   需要注意的是，移除的递归并不是返回最终移除的节点，而是返回更改完后的树结构（包括子树结构）。特别是对于第三种情况，需找到对应节点的右子树中最小的节点，然后与该节点进行替换。以下是各种情况的示意图：

   - 第一种情况：当无子节点时

     ![](http://ppu8vcpyg.bkt.clouddn.com/Tree-Remove-one.png)

   - 第二种情况：当只有一个子节点时

     ![](http://ppu8vcpyg.bkt.clouddn.com/Tree-Remove-two.png)

   - 第三种情况：当有两个子节点时

     ![](http://ppu8vcpyg.bkt.clouddn.com/Tree-Remove-Three.png)



需要注意的是，BST有一个很明显的的缺陷：当向树中添加节点数时，树的一边可能会非常深。当树的一边很深时，而其他边却很浅，会导致树在搜索移除添加元素时耗费性能。因此AVL（平衡二叉搜索树）出来了，即任何一个节点的左右两侧子树的高度之差最多为1。