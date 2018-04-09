---
title: Java实现二叉排序树的插入、查找和删除 
date: 2017-10-12 15:39:00  
tags: [算法,数据结构]    
categories: 数据结构与算法
---
## 二叉排序树
又叫二叉查找树。  
1.可以使一颗空树  
2.若左子树不为空，则左子树上所有节点的值均小于根节点的值  
3.若右子树不为空，则右子树上所有节点的值均大于根节点的值  
4.左右子树也分别为二叉排序树  

### 性能分析  
**查找性能**：含有n个节点的二叉排序树的平均查找长度和树的形态有关。  
（最坏情况）当先后插入的关键字有序时，构成的二叉排序树蜕变为单枝树，查找性能为O(n)  
（最好情况）二叉排序的形态和折半查找的判定树相同，平均查找长度和log2(n)成正比  

**插入、删除性能**：插入、删除操作的时间复杂度都是O(logn)级的。

```
//二叉树节点定义
public class Node{
    private int value;
    private Node left;
    private Node right;
    
    public Node(Node left, Node right, int value){
        this.left = left;
        this.right=right;
        this.value = value;
    }
    
    set,get...
}

public class BinarySortTree{
    
    private Node root=null;
    
    //查找二叉排序树中是否有key值
    public boolean searchBST(int key){
        Node current = root;
        while(current != null){
            if(key == current.getValue())
                return true;
            else(key < current.getValue())
                current=current.getLeft();
            else
                current=current.getRight();
        }
        return false;
    }
    
    //向二叉排序树中插入节点
    public void insertBST(int key){
        Node p = root;
        //记录查找节点的父节点
        Node prev = null;
        //一直查找下去，一直到满足条件的叶子节点
        while(p != null){
            prev=p;
            if(key < p.getValue())
                p = p.getLeft();
            else if(key > p.getValue())
                p = p.getRight();
            else
                return;//找到了相同节点就返回
        }
        
        //prev为要安放节点的父结点，根据key的大小，判断放入prev的左右节点
        if(root == null)
            root = new Node(key);
        else if(key < prev.getValue())
            prev.setLeft(new Node(key));
        else
            prev.setRight(new Node(key));
    }
    
    //删除二叉排序树中的节点
    分为三种情况：
    (1)要删除的节点p是叶子节点，删除置空即可
    (2)p只有左子树或右子树，直接让左子树/右子树代替p
    (3)p既有左子树又有右子树，用p左子树中最大的那个值（最右端s）代替p，删除s，重接其左子树
    public boolean delete(Node node){
        Node temp = null;
        //右子树为空，重接它的左子树；如果是叶子节点，在这也被置为空
        if(node.getRight()==null){
            temp = node;
            node = node.getLeft();
        }
        //左子树为空，重接它的右子树
        else if(node.getLeft()==null){
            temp = node;
            node = node.getRight();
        }
        //左右子树都不为空
        else{
            temp = node;
            Node s = node;
            //转向左子树，然后向右走到尽头
            s=s.getLeft();//temp是s的父结点
            while(s.getRight() != null){
                temp = s;
                s = s.getRight();
            }
            node.setValue(s.getValue());//将node的值设为左子树最右侧s的值
            if(temp != node){
                temp.setRight(s.getLeft());//见书p326
            }
            else{
                temp.setLeft(s.getLeft());
            }
        }
        return true;
    }
}
```
