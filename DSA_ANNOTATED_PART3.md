# 200 Most-Asked DSA Interview Questions - ANNOTATED PART 3

## Linked List (Q56–Q75) with Context & Comments

---

### Q56. Reverse Linked List
**Problem Statement:**
Reverse a singly linked list in-place and return the new head.

**Example:**
```
Input: 1 -> 2 -> 3 -> 4 -> NULL
Output: 4 -> 3 -> 2 -> 1 -> NULL
```

**Approach:** Iterative reversal by changing next pointers as we traverse.
- Time: O(n), Space: O(1)

```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode(int v): val(v), next(nullptr) {}
};

ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;  // Previous node (initially null)
    
    while (head) {
        auto nx = head->next;      // Save next node before changing pointer
        head->next = prev;         // Reverse the link
        prev = head;               // Move prev forward
        head = nx;                 // Move head forward
    }
    return prev;  // New head is previous
}
```

---

### Q57. Linked List Cycle
**Problem Statement:**
Detect if a linked list contains a cycle (node points back to earlier node).

**Example:**
```
Input: 1 -> 2 -> 3 -> 4
                 ^    |
                 |____|  (4 points back to 2)
Output: true
```

**Approach:** Floyd's Cycle Detection (tortoise and hare).
- Time: O(n), Space: O(1)

```cpp
bool hasCycle(ListNode* head) {
    auto s = head, f = head;  // Slow and fast pointers
    
    while (f && f->next) {
        s = s->next;              // Move slow 1 step
        f = f->next->next;         // Move fast 2 steps
        
        if (s == f) return true;   // Cycle detected
    }
    return false;  // No cycle
}
```

---

### Q58. Merge Two Sorted Lists
**Problem Statement:**
Merge two sorted linked lists into a single sorted list.

**Example:**
```
Input: list1 = 1 -> 2 -> 4, list2 = 1 -> 3 -> 4
Output: 1 -> 1 -> 2 -> 3 -> 4 -> 4
```

**Approach:** Two-pointer merge, compare values and link smaller nodes.
- Time: O(n + m), Space: O(1)

```cpp
ListNode* mergeTwoLists(ListNode* a, ListNode* b) {
    ListNode dummy(0), *t = &dummy;  // Dummy node and tail pointer
    
    while (a && b) {
        if (a->val <= b->val) {
            t->next = a;  // Link smaller node
            a = a->next;
        } else {
            t->next = b;
            b = b->next;
        }
        t = t->next;  // Move tail forward
    }
    
    // Link remaining nodes (one list may be exhausted)
    t->next = a ? a : b;
    return dummy.next;  // Skip dummy node
}
```

---

### Q59. Remove Nth Node From End
**Problem Statement:**
Remove the nth node from the end of a list (n = 1 means last node).

**Example:**
```
Input: 1 -> 2 -> 3 -> 4 -> 5, n = 2
Output: 1 -> 2 -> 3 -> 5  (removed 4)
```

**Approach:** Use two pointers with n-step gap to find previous node.
- Time: O(n), Space: O(1)

```cpp
ListNode* removeNthFromEnd(ListNode* head, int n) {
    ListNode dummy(0);
    dummy.next = head;
    
    auto f = &dummy, s = &dummy;  // Fast and slow pointers
    
    // Move fast pointer n+1 steps ahead
    for (int i = 0; i <= n; i++) {
        f = f->next;
    }
    
    // Move both pointers until fast reaches end
    while (f) {
        f = f->next;
        s = s->next;
    }
    
    // Remove nth node by skipping it
    s->next = s->next->next;
    return dummy.next;
}
```

---

### Q60. Reorder List
**Problem Statement:**
Reorder list so it goes: first, last, second, second-last, etc.

**Example:**
```
Input: 1 -> 2 -> 3 -> 4
Output: 1 -> 4 -> 2 -> 3
```

**Approach:** Find middle, reverse second half, then merge two halves alternately.
- Time: O(n), Space: O(1)

```cpp
void reorderList(ListNode* h) {
    if (!h || !h->next) return;
    
    // Step 1: Find middle using slow/fast pointers
    auto s = h, f = h;
    while (f->next && f->next->next) {
        s = s->next;
        f = f->next->next;
    }
    
    // Step 2: Reverse second half
    ListNode* prev = nullptr;
    auto cur = s->next;
    s->next = nullptr;  // Break the list
    
    while (cur) {
        auto nx = cur->next;
        cur->next = prev;
        prev = cur;
        cur = nx;
    }
    
    // Step 3: Merge two halves alternately
    auto a = h, b = prev;
    while (b) {
        auto t1 = a->next, t2 = b->next;
        a->next = b;
        b->next = t1;
        a = t1;
        b = t2;
    }
}
```

---

### Q61. Add Two Numbers
**Problem Statement:**
Add two numbers represented as linked lists (digits stored in reverse order).

**Example:**
```
Input: l1 = 2 -> 4 -> 3, l2 = 5 -> 6 -> 4
       (represents 342 + 465)
Output: 7 -> 0 -> 8
       (represents 807)
```

**Approach:** Simulate addition with carry.
- Time: O(max(n, m)), Space: O(1)

```cpp
ListNode* addTwoNumbers(ListNode* a, ListNode* b) {
    ListNode dummy(0), *t = &dummy;
    int carry = 0;
    
    while (a || b || carry) {
        // Get values (0 if node is null)
        int s = carry + (a ? a->val : 0) + (b ? b->val : 0);
        carry = s / 10;  // Extract carry
        
        // Create new node with digit
        t->next = new ListNode(s % 10);
        t = t->next;
        
        // Move pointers forward if they exist
        if (a) a = a->next;
        if (b) b = b->next;
    }
    return dummy.next;
}
```

---

### Q62. Copy List with Random Pointer
**Problem Statement:**
Deep copy a linked list where each node has a random pointer to any node.

**Example:**
```
Input: Original list with next and random pointers
Output: Deep copy with same structure and pointers
```

**Approach:** Hash map to track original -> copy mapping.
- Time: O(n), Space: O(n)

```cpp
struct Node {
    int val;
    Node *next, *random;
};

Node* copyRandomList(Node* head) {
    unordered_map<Node*, Node*> mp;  // Original -> Copy mapping
    
    // First pass: create all nodes
    for (auto c = head; c; c = c->next) {
        mp[c] = new Node{c->val, nullptr, nullptr};
    }
    
    // Second pass: set next and random pointers
    for (auto c = head; c; c = c->next) {
        mp[c]->next = mp[c->next];      // Set next pointer
        mp[c]->random = mp[c->random];  // Set random pointer
    }
    
    return mp[head];  // Return copy of head
}
```

---

### Q63. Intersection of Two Linked Lists
**Problem Statement:**
Find the intersection node of two linked lists (if they intersect).

**Example:**
```
List1: 4 -> 1 -> 8 -> 4 -> 5
             ↓
List2: 5 -> 6 -> 1 ↑  (intersect at node with value 8)
Output: 8
```

**Approach:** Two pointers reach intersection by traversing both lists.
- Time: O(n + m), Space: O(1)

```cpp
ListNode* getIntersectionNode(ListNode* a, ListNode* b) {
    auto p1 = a, p2 = b;
    
    // When one pointer reaches end, switch to other list
    while (p1 != p2) {
        p1 = p1 ? p1->next : b;  // Switch to b when reaching end
        p2 = p2 ? p2->next : a;  // Switch to a when reaching end
    }
    return p1;  // Either both null or at intersection
}
```

---

### Q64. Remove Duplicates from Sorted List
**Problem Statement:**
Remove duplicate nodes from a sorted linked list (keep one occurrence).

**Example:**
```
Input: 1 -> 1 -> 2 -> 3 -> 3
Output: 1 -> 2 -> 3
```

**Approach:** Compare current with next, skip if duplicate.
- Time: O(n), Space: O(1)

```cpp
ListNode* deleteDuplicates(ListNode* h) {
    for (auto c = h; c && c->next;) {
        if (c->val == c->next->val) {
            c->next = c->next->next;  // Skip duplicate
        } else {
            c = c->next;  // Move to next
        }
    }
    return h;
}
```

---

### Q65. Palindrome Linked List
**Problem Statement:**
Check if a linked list is a palindrome.

**Example:**
```
Input: 1 -> 2 -> 2 -> 1
Output: true
Input: 1 -> 2 -> 3
Output: false
```

**Approach:** Find middle, reverse second half, compare both halves.
- Time: O(n), Space: O(1)

```cpp
bool isPalindrome(ListNode* h) {
    // Step 1: Find middle
    auto s = h, f = h;
    while (f && f->next) {
        s = s->next;
        f = f->next->next;
    }
    
    // Step 2: Reverse second half
    ListNode* prev = nullptr;
    while (s) {
        auto nx = s->next;
        s->next = prev;
        prev = s;
        s = nx;
    }
    
    // Step 3: Compare both halves
    while (prev) {  // prev will be shorter or equal
        if (prev->val != h->val) return false;
        prev = prev->next;
        h = h->next;
    }
    return true;
}
```

---

### Q66. Swap Nodes in Pairs
**Problem Statement:**
Swap every two adjacent nodes in a linked list.

**Example:**
```
Input: 1 -> 2 -> 3 -> 4
Output: 2 -> 1 -> 4 -> 3
```

**Approach:** Use dummy node to handle swaps iteratively.
- Time: O(n), Space: O(1)

```cpp
ListNode* swapPairs(ListNode* h) {
    ListNode dummy(0);
    dummy.next = h;
    auto p = &dummy;  // Pointer to previous node
    
    while (p->next && p->next->next) {
        auto a = p->next, b = a->next;
        
        // Perform swap
        a->next = b->next;  // a points to node after b
        b->next = a;        // b points to a
        p->next = b;        // previous points to b
        
        p = a;  // Move to next pair
    }
    return dummy.next;
}
```

---

### Q67. Flatten Multilevel Doubly Linked List
**Problem Statement:**
Flatten a multilevel doubly linked list by connecting child lists sequentially.

**Example:**
```
Input:  1 - 2 - 3 - 4
            |
            7 - 8
                |
                12
Output: 1 - 2 - 7 - 8 - 12 - 3 - 4
```

**Approach:** DFS with stack to process child lists.
- Time: O(n), Space: O(n)

```cpp
struct DNode {
    int val;
    DNode *prev, *next, *child;
};

DNode* flatten(DNode* h) {
    stack<DNode*> st;
    if (h) st.push(h);
    
    DNode* prev = nullptr;
    
    while (!st.empty()) {
        auto c = st.top();
        st.pop();
        
        // Link with previous
        if (prev) {
            prev->next = c;
            c->prev = prev;
        }
        
        // Push next to stack (if exists)
        if (c->next) st.push(c->next);
        
        // Push child to stack (if exists) and remove child pointer
        if (c->child) {
            st.push(c->child);
            c->child = nullptr;
        }
        
        prev = c;
    }
    return h;
}
```

---

### Q68. LRU Cache
**Problem Statement:**
Implement Least Recently Used cache with get and put operations in O(1).

**Example:**
```
LRUCache cache(2);
cache.put(1, 1);      // cache = {1: 1}
cache.put(2, 2);      // cache = {1: 1, 2: 2}
cache.get(1);         // returns 1, cache = {2: 2, 1: 1}
cache.put(3, 3);      // remove 2, cache = {1: 1, 3: 3}
cache.get(2);         // returns -1 (not found)
```

**Approach:** Doubly linked list + hash map for O(1) operations.
- Time: O(1) for get and put, Space: O(capacity)

```cpp
class LRUCache {
    int cap;
    list<pair<int,int>> lst;  // Doubly linked list {key, value}
    unordered_map<int, list<pair<int,int>>::iterator> mp;  // key -> iterator
    
public:
    LRUCache(int c) : cap(c) {}
    
    int get(int k) {
        if (!mp.count(k)) return -1;
        
        // Move to front (most recently used)
        lst.splice(lst.begin(), lst, mp[k]);
        return mp[k]->second;
    }
    
    void put(int k, int v) {
        if (mp.count(k)) {
            // Update value and move to front
            mp[k]->second = v;
            lst.splice(lst.begin(), lst, mp[k]);
            return;
        }
        
        // Remove LRU item if capacity exceeded
        if (lst.size() == cap) {
            mp.erase(lst.back().first);
            lst.pop_back();
        }
        
        // Add new item at front
        lst.push_front({k, v});
        mp[k] = lst.begin();
    }
};
```

---

### Q69. Sort List (Merge Sort)
**Problem Statement:**
Sort a linked list in O(n log n) time and O(1) space (merge sort on linked list).

**Example:**
```
Input: 4 -> 2 -> 1 -> 3
Output: 1 -> 2 -> 3 -> 4
```

**Approach:** Merge sort - find middle, sort halves, merge results.
- Time: O(n log n), Space: O(log n) recursion

```cpp
ListNode* sortList(ListNode* h) {
    // Base case: empty or single node
    if (!h || !h->next) return h;
    
    // Step 1: Find middle using slow/fast pointers
    auto s = h, f = h->next;
    while (f && f->next) {
        s = s->next;
        f = f->next->next;
    }
    
    // Step 2: Break list into two halves
    auto mid = s->next;
    s->next = nullptr;
    
    // Step 3: Recursively sort both halves and merge
    return mergeTwoLists(sortList(h), sortList(mid));
}
```

---

### Q70. Rotate List
**Problem Statement:**
Rotate list k positions to the right.

**Example:**
```
Input: 1 -> 2 -> 3 -> 4 -> 5, k = 2
Output: 4 -> 5 -> 1 -> 2 -> 3
```

**Approach:** Find list length, calculate effective rotation, reconnect.
- Time: O(n), Space: O(1)

```cpp
ListNode* rotateRight(ListNode* h, int k) {
    if (!h) return h;
    
    // Step 1: Find length and tail
    int len = 1;
    auto t = h;
    while (t->next) {
        t = t->next;
        len++;
    }
    
    // Step 2: Connect tail to head (circular)
    t->next = h;
    
    // Step 3: Find new head position
    k = len - k % len;  // Effective rotation
    while (k--) {
        t = t->next;
    }
    
    // Step 4: Break circle and return new head
    h = t->next;
    t->next = nullptr;
    return h;
}
```

---

### Q71. Partition List
**Problem Statement:**
Partition list around a value x such that nodes < x come before nodes >= x.

**Example:**
```
Input: 1 -> 4 -> 3 -> 2 -> 5 -> 2, x = 3
Output: 1 -> 2 -> 2 -> 4 -> 3 -> 5
```

**Approach:** Create two lists (smaller and larger), then merge.
- Time: O(n), Space: O(1)

```cpp
ListNode* partition(ListNode* h, int x) {
    ListNode a(0), b(0);  // Two dummy nodes
    auto ta = &a, tb = &b;  // Tail pointers
    
    while (h) {
        if (h->val < x) {
            ta->next = h;  // Add to smaller list
            ta = ta->next;
        } else {
            tb->next = h;  // Add to larger list
            tb = tb->next;
        }
        h = h->next;
    }
    
    tb->next = nullptr;      // Terminate larger list
    ta->next = b.next;       // Connect lists
    return a.next;
}
```

---

### Q72. Reverse Nodes in k-Group
**Problem Statement:**
Reverse every k consecutive nodes in a linked list.

**Example:**
```
Input: 1 -> 2 -> 3 -> 4 -> 5, k = 2
Output: 2 -> 1 -> 4 -> 3 -> 5
```

**Approach:** Recursive - reverse k nodes, then recursively reverse rest.
- Time: O(n), Space: O(n/k)

```cpp
ListNode* reverseKGroup(ListNode* h, int k) {
    // Check if we have k nodes to reverse
    auto c = h;
    int cnt = 0;
    while (c && cnt < k) {
        c = c->next;
        cnt++;
    }
    
    // If less than k nodes, return as is
    if (cnt < k) return h;
    
    // Recursively reverse rest starting from c
    ListNode* prev = reverseKGroup(c, k);
    
    // Reverse first k nodes
    while (cnt--) {
        auto nx = h->next;
        h->next = prev;
        prev = h;
        h = nx;
    }
    
    return prev;  // New head after reversing k nodes
}
```

---

### Q73. Merge K Sorted Lists
**Problem Statement:**
Merge k sorted lists into one sorted list.

**Example:**
```
Input: lists = [[1,4,5], [1,3,4], [2,6]]
Output: [1, 1, 2, 3, 4, 4, 5, 6]
```

**Approach:** Priority queue (min-heap) to always get smallest node.
- Time: O(n log k), Space: O(k)

```cpp
ListNode* mergeKLists(vector<ListNode*>& v) {
    // Min-heap comparator
    auto cmp = [](ListNode* a, ListNode* b) {
        return a->val > b->val;
    };
    
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);
    
    // Push first node from each list
    for (auto x : v) {
        if (x) pq.push(x);
    }
    
    ListNode dummy(0), *t = &dummy;
    
    while (!pq.empty()) {
        auto n = pq.top();
        pq.pop();
        
        t->next = n;
        t = t->next;
        
        // Push next node if exists
        if (n->next) pq.push(n->next);
    }
    
    return dummy.next;
}
```

---

### Q74. Middle of the Linked List
**Problem Statement:**
Find the middle node of a linked list (if even length, return second middle).

**Example:**
```
Input: 1 -> 2 -> 3 -> 4 -> 5
Output: 3 (the middle node)
Input: 1 -> 2 -> 3 -> 4
Output: 3 (second of two middle nodes)
```

**Approach:** Slow and fast pointers.
- Time: O(n), Space: O(1)

```cpp
ListNode* middleNode(ListNode* h) {
    auto s = h, f = h;  // Slow and fast pointers
    
    // Fast moves 2, slow moves 1
    while (f && f->next) {
        s = s->next;
        f = f->next->next;
    }
    
    return s;  // Slow is at middle
}
```

---

### Q75. Odd Even Linked List
**Problem Statement:**
Reorder list so odd-indexed nodes come first, then even-indexed nodes (1-indexed).

**Example:**
```
Input: 1 -> 2 -> 3 -> 4 -> 5
Output: 1 -> 3 -> 5 -> 2 -> 4
```

**Approach:** Separate odd and even nodes, then merge.
- Time: O(n), Space: O(1)

```cpp
ListNode* oddEvenList(ListNode* h) {
    if (!h) return h;
    
    auto o = h;          // Odd nodes head
    auto e = h->next;    // Even nodes head
    auto eh = e;         // Save even head for merging
    
    while (e && e->next) {
        o->next = e->next;  // Link next odd
        o = o->next;        // Move to next odd
        
        e->next = o->next;  // Link next even
        e = e->next;        // Move to next even
    }
    
    o->next = eh;  // Connect odd and even lists
    return h;
}
```

---

## Summary Table for Q56-Q75

| Question | Topic | Approach | Time | Space |
|----------|-------|----------|------|-------|
| Q56 | Reverse List | Iterative | O(n) | O(1) |
| Q57 | Cycle Detection | Floyd's | O(n) | O(1) |
| Q58 | Merge Sorted | Two Pointer | O(n+m) | O(1) |
| Q59 | Remove Nth | Two Pointer | O(n) | O(1) |
| Q60 | Reorder List | Reverse + Merge | O(n) | O(1) |
| Q61 | Add Two Numbers | Simulation | O(max(n,m)) | O(1) |
| Q62 | Copy Random List | Hash Map | O(n) | O(n) |
| Q63 | Intersection | Two Pointer | O(n+m) | O(1) |
| Q64 | Remove Duplicates | Iteration | O(n) | O(1) |
| Q65 | Palindrome List | Reverse Half | O(n) | O(1) |
| Q66 | Swap Pairs | Iteration | O(n) | O(1) |
| Q67 | Flatten Multilevel | DFS | O(n) | O(n) |
| Q68 | LRU Cache | List + Hash | O(1) | O(capacity) |
| Q69 | Sort List | Merge Sort | O(n log n) | O(log n) |
| Q70 | Rotate List | Circular | O(n) | O(1) |
| Q71 | Partition List | Two Lists | O(n) | O(1) |
| Q72 | Reverse K-Group | Recursion | O(n) | O(n/k) |
| Q73 | Merge K Lists | Min-Heap | O(n log k) | O(k) |
| Q74 | Middle Node | Two Pointer | O(n) | O(1) |
| Q75 | Odd Even List | Separation | O(n) | O(1) |

---

**END OF PART 3 (Q56-Q75)**

Next: Part 4 will cover Stack & Queue (Q76-Q90)

