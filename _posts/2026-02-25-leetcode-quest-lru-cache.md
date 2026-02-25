---
layout: post
title:  LeetCode Quest: LRU Cache
date:   2026-02-25 18:08:00 +0800
categories: document
tag: leetcode
topping: false
---

* content
{:toc}
Q1. LRU Cache

### Problem
Design a data structure that follows the constraints of a Least Recently Used (LRU) cache.

#### Constraints
- 1 <= capacity <= 3000 
- 0 <= key <= 104 
- 0 <= value <= 105 
- At most 2 * 105 calls will be made to get and put.

0(1) implementation using:
- Hash map (unordered_map) → O(1) access by key
- Doubly linked list → O(1) insert, remove, move-to-front

### Code
```c
class LRUCache {
private:
    struct Node {
        int key, value;
        Node* prev;
        Node* next;
        Node(int k, int v) : key(k), value(v), prev(nullptr), next(nullptr) {}
    };
    
    int cap;
    unordered_map<int, Node*> mp;
    Node* head;
    Node* tail;
    
    void remove(Node* node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
    }
    
    void insertToFront(Node* node) {
        node->next = head->next;
        node->prev = head;
        head->next->prev = node;
        head->next = node;
    }

public:
    LRUCache(int capacity) {
        cap = capacity;
        head = new Node(0, 0);
        tail = new Node(0, 0);
        head->next = tail;
        tail->prev = head;
    }
    
    int get(int key) {
        if (mp.find(key) == mp.end())
            return -1;
        
        Node* node = mp[key];
        remove(node);
        insertToFront(node);
        return node->value;
    }
    
    void put(int key, int value) {
        if (mp.find(key) != mp.end()) {
            Node* node = mp[key];
            node->value = value;
            remove(node);
            insertToFront(node);
        } else {
            if (mp.size() == cap) {
                Node* lru = tail->prev;
                remove(lru);
                mp.erase(lru->key);
                delete lru;
            }
            Node* node = new Node(key, value);
            mp[key] = node;
            insertToFront(node);
        }
    }
};
```

### Why this is 0(1)?
- `unordered_map` → O(1) lookup
- Doubly linked list → O(1) remove & insert
- No traversal anywhere

Mental model:
`HEAD <-> Most Recent <-> ... <-> Least Recent <-> TAIL`

- `get()` → move node to front

- `put()`
: If exists → update + move to front
: If full → remove from back
: Insert new at front

