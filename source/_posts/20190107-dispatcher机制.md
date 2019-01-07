---
title: cocos2dx Event机制
comments: true
date: 2017-01-07 18:34:51
tags:
- cocos2dx
categories:
- cocos2dx
---

#### 基本机制

事件机制类似观察者模式。

EventListener 事件监听者，封装事件处理代码，包含事件会回调。

事件包含时间信息。

EventDispatcher 事件分发者，管理分发事件。



#### 事件添加机制

EventDispatcher中以 listenerId->EventListenerVector的方式保存了事件监听者。

事件有两种添加方式一种是关联节点，一种是固定优先级的。

添加时根据是添加的方式放入不同的vector。



std::unordered_map<EventListener::ListenerID, EventListenerVector*> _listenerMap;

EventListenerVector包含。

std::vector<EventListener*>* _fixedListeners;

std::vector<EventListener*>* _sceneGraphListeners;



void EventDispatcher::addEventListenerWithSceneGraph(EventListener* listener, Node* node)

void EventDispatcher::addEventListenerWithFixedPriority(EventListener* listener, int priority)



#### 事件移除

固定优先级的需要removeEventListener（）

关联节点的节点消失会自行移除。



#### 事件分发

根据id取出事件监听者，

排序事件监听者按照绘制顺序相反顺序或者是事件的优先级排序

sortEventListeners(listenerID);

取出事件监听者id关联的事件监听者列表  将事件分发给每个事件监听者。



#### 分发顺序:

分别取出节点关联的和固定优先级的时间接听者给予分发

分发顺序是按照优先级<0     ==0(关联节点)      >0 的分发顺去