---
title: HashMap的四种遍历方式  
date: 2017-12-01 20:39:00  
tags: [java,java基础]    
categories: java基础  
---
# 一、Map的四种遍历方式
## 1、foreach map.entrySet()
```
Map<String,String> map = new HashMap<String,String>();
for(Entry<String,String> entry : map.entrySet()){
    entry.getKey();
    entry.getValue();
    
    //在遍历过程中删除元素，会抛出异常java.util.ConcurrentModificationException
    //entry.remove(entry.getKey());
}
```
## 2、调用map.entrySet()的集合迭代器
```
Iterator<Map.Entry<String,String>> it = map.entrySet().iterator();
while(it.hasNext()){
    Map.Entry<String,String> entry = it.next();
    entry.getKey();
    entry.getValue();
}
```
## 3、foreach map.keySet()
```
Map<String,String> map = new HashMap<String,String>();
for(String key : map.keySet()){
    map.get(key);
}
```
## 4、foreach map.entrySet()，用临时变量保存map.entrySet()
```
Set<Entry<String,String>> entrySet = map.entrySet();
for(Entry<String,String> entry : entrySet()){
    entry.getKey();
    entry.getValue();
}
```
# 二、结论
1、在foreach map.entrySet()这种遍历方法的遍历过程中，不能删除元素，由于在遍历HashMap中删除了当前元素，下一个待访问的元素的指针也丢失了，所以会抛出java.util.ConcurrentModificationException。  
2、如果需要在遍历过程中进行删除元素，可以使用调用map.entrySet()的集合迭代器来进行遍历。  
3、如果只需要遍历key而不需要value的话，可以使用foreach map.keySet()的方式进行遍历。