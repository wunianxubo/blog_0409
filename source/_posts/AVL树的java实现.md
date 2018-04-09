---
title: AVL树的Java实现 
date: 2017-10-23 20:55:00  
tags: [算法,数据结构]    
categories: 数据结构与算法
---
AVL树，也叫二叉平衡树，是一种二叉排序树，其中每一个节点的左子树和右子树的高度差至多等于1。  
# AVL树的Java实现
## 1、节点
### 1.1节点定义
```
public class AVLTree<T extends Comparable<T>>{
    private AVLTreeNode<T extends Comparable<T>>{
        T key;                 //键值
        int height;            //高度
        AVLTreeNode<T> left;   //左孩子
        AVLTreeNode<T> right;  //右孩子
        
        public AVLTreeNode(T key, AVLTreeNode<T> left, AVLTreeNode<T> right){
            this.key = key;
            this.left = left;
            this.right = right;
            this.height = 0;
        }
        
    }
}
```
### 1.2树的高度
```
//获取树的高度
private int height(AVLTreeNode<T> tree){
    if(tree != null)
        return tree.height;
    return 0;
}

public int height(){
    return height(mRoot);
}
```
### 1.3比较大小
```
比较两个值的大小
private int max(int a, int b){
    return a>b ? a: b;
}
```

## 2、旋转
如果在AVL树中进行插入或删除节点后，可能导致AVL树失去平衡。这种失去平衡可以概括为4种姿势：LL（左左），LR（左右），RR（右右），RL（右左）。  

![image](http://osrmzp0jr.bkt.clouddn.com/%E6%97%8B%E8%BD%AC1.jpg)  
### 2.1 LL的旋转
![image](http://osrmzp0jr.bkt.clouddn.com/LL.jpg)  

```
//LL：左左对应的情况
//返回值：旋转后的根节点
private AVLTreeNode<T> leftLeftRotation(AVLTreeNode<T> k2){
    AVLTreeNode<T> k1;
    k1 = k2.left;
    k2.left = k1.right;
    k1.right = k2;
    
    k2.height = max(height(k2.left),height(k2.right)) + 1;
    k1.height = max(height(k1.left),k2.height) + 1;
    
    return k1;
}
```
### 2.2 RR的旋转
![image](http://osrmzp0jr.bkt.clouddn.com/RR.jpg)  

```
//RR：右右对应的情况
//返回值：旋转后的根节点
private AVLTreeNode<T> rightRightRotation(AVLTreeNode<T> k1){
    AVLTreeNode<T> k2;
    k2 = k1.right;
    k1.right = k2.left;
    k2.left = k1;
    
    k1.height = max(height(k1.left),height(k1.right)) + 1;
    k2.height = max(height(k2.left),k1.height) + 1;
    
    return k2;
}
```
### 2.3 LR的旋转
LR失去平衡的情况，需要经过两次旋转才能让AVL树恢复平衡。  
![image](http://osrmzp0jr.bkt.clouddn.com/LR.jpg)  

```
//LR：左右对应的情况
//返回值：旋转后的根节点
private AVLTreeNode<T> leftRightRotation(AVLTreeNode<T> k3){
    k3.left = rightRightRotation(k3.left);
    
    return leftLeftRotation(k3);
}
```
### 2.4 RL的旋转
![image](http://osrmzp0jr.bkt.clouddn.com/RL.jpg)  

```
//RL：右左对应的情况
//返回值：旋转后的根节点
private AVLTreeNode<T> rightLeftRotation(AVLTreeNode<T> k1){
    k1.right = leftLeftRotation(k1.right);
    
    return rightRightRotation(k1);
}
```

## 3、插入
```
//将结点插入到AVL树中，并返回根节点
private AVLTreeNode<T> insert(AVLTreeNode<T> tree, T key){//tree为根节点，key为待插入的值
    if(tree == null){
        //新建节点
        tree = new AVLTreeNode<T>(key,null,null);
        if(tree==null){
            System.out.println("创建节点失败！");
            return null;
        }
    }else{
        int cmp = key.compareTo(tree.key);
        
        //将key插入到Tree的左子树
        if(cmp < 0){
            tree.left = insert(tree.left, key);
            //插入节点后，若AVL树失去平衡，做相应的调节
            if(height(tree.left) - height(tree.right) == 2){
                if(key.compareTo(tree.left.key) < 0)
                    tree = leftLeftRotation(tree);
                else
                    tree = leftRightRotation(tree);
            }
        }
        //将key插入到Tree的右子树
        else if(cmp > 0){
            tree.right = insert(tree.right, key);
            //插入节点后，若AVL树失去平衡，做相应的调节
            if(height(tree.right) - height(tree.left== 2){
                if(key.compareTo(tree.left.key) > 0)
                    tree = rightRightRotation(tree);
                else
                    tree = rightLeftRotation(tree);
            }
        }else{
            System.out.println("添加节点失败！");
        }
    }
    
    tree.height = max(height(tree.left), height(tree.right)) + 1;
    
    return tree;
    
}
```

## 4、删除
```
private AVLTreeNode<T> remove(AVLTreeNode<T> tree, AVLTreeNode<T> z) {//tree为根节点，z为待删除节点
    // 根为空 或者 没有要删除的节点，直接返回null。
    if (tree==null || z==null)
        return null;

    int cmp = z.key.compareTo(tree.key);
    if (cmp < 0) {        // 待删除的节点在"tree的左子树"中
        tree.left = remove(tree.left, z);
        // 删除节点后，若AVL树失去平衡，则进行相应的调节。
        if (height(tree.right) - height(tree.left) == 2) {
            AVLTreeNode<T> r =  tree.right;
            if (height(r.left) > height(r.right))
                tree = rightLeftRotation(tree);
            else
                tree = rightRightRotation(tree);
        }
    } else if (cmp > 0) {    // 待删除的节点在"tree的右子树"中
        tree.right = remove(tree.right, z);
        // 删除节点后，若AVL树失去平衡，则进行相应的调节。
        if (height(tree.left) - height(tree.right) == 2) {
            AVLTreeNode<T> l =  tree.left;
            if (height(l.right) > height(l.left))
                tree = leftRightRotation(tree);
            else
                tree = leftLeftRotation(tree);
        }
    } else {    // tree是对应要删除的节点。
        // tree的左右孩子都非空
        if ((tree.left!=null) && (tree.right!=null)) {
            if (height(tree.left) > height(tree.right)) {
                // 如果tree的左子树比右子树高；
                // 则(01)找出tree的左子树中的最大节点
                //   (02)将该最大节点的值赋值给tree。
                //   (03)删除该最大节点。
                // 这类似于用"tree的左子树中最大节点"做"tree"的替身；
                // 采用这种方式的好处是：删除"tree的左子树中最大节点"之后，AVL树仍然是平衡的。
                AVLTreeNode<T> max = maximum(tree.left);
                tree.key = max.key;
                tree.left = remove(tree.left, max);
            } else {
                // 如果tree的左子树不比右子树高(即它们相等，或右子树比左子树高1)
                // 则(01)找出tree的右子树中的最小节点
                //   (02)将该最小节点的值赋值给tree。
                //   (03)删除该最小节点。
                // 这类似于用"tree的右子树中最小节点"做"tree"的替身；
                // 采用这种方式的好处是：删除"tree的右子树中最小节点"之后，AVL树仍然是平衡的。
                AVLTreeNode<T> min = minimum(tree.right);
                tree.key = min.key;
                tree.right = remove(tree.right, min);
            }
        } else {
            AVLTreeNode<T> tmp = tree;
            tree = (tree.left!=null) ? tree.left : tree.right;
            tmp = null;
        }
    }

    return tree;
}
```


