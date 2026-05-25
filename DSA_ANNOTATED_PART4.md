# 200 Most-Asked DSA Interview Questions - ANNOTATED PART 4

## Stack & Queue (Q76–Q90) with Context & Comments

---

### Q76. Min Stack
**Problem Statement:**
Design a stack that supports push, pop, top, and getMin in O(1) time where getMin returns the minimum element.

**Example:**
```
MinStack ms;
ms.push(-2);
ms.push(0);
ms.push(-3);
ms.getMin();  // Returns -3
ms.pop();
ms.getMin();  // Returns -2
```

**Approach:** Use two stacks - one for values, one for running minimums.
- Time: O(1) for all operations, Space: O(n)

```cpp
class MinStack {
    stack<int> s;   // Main stack
    stack<int> mn;  // Stack to track minimums
    
public:
    void push(int v) {
        s.push(v);
        // Push to min stack if empty or value <= current min
        if (mn.empty() || v <= mn.top()) {
            mn.push(v);
        }
    }
    
    void pop() {
        // If popped element is current min, pop from min stack too
        if (s.top() == mn.top()) {
            mn.pop();
        }
        s.pop();
    }
    
    int top() {
        return s.top();
    }
    
    int getMin() {
        return mn.top();
    }
};
```

---

### Q77. Evaluate Reverse Polish Notation
**Problem Statement:**
Evaluate an expression in Reverse Polish Notation (postfix notation).

**Example:**
```
Input: tokens = ["2","1","+","3","*"]
Output: 9
Explanation: ((2 + 1) * 3) = 9
```

**Approach:** Stack-based evaluation - operands are pushed, operators pop and compute.
- Time: O(n), Space: O(n)

```cpp
int evalRPN(vector<string>& v) {
    stack<int> s;
    
    for (auto& t : v) {
        // Check if token is an operator
        if (t == "+" || t == "-" || t == "*" || t == "/") {
            // Pop two operands (note: order matters for - and /)
            int b = s.top(); s.pop();
            int a = s.top(); s.pop();
            
            // Perform operation and push result
            if (t == "+") s.push(a + b);
            else if (t == "-") s.push(a - b);
            else if (t == "*") s.push(a * b);
            else s.push(a / b);  // Integer division
        } else {
            // Token is a number, push to stack
            s.push(stoi(t));
        }
    }
    
    return s.top();  // Final result
}
```

---

### Q78. Daily Temperatures
**Problem Statement:**
Given temperatures, return array where each element is days until warmer temperature.

**Example:**
```
Input: temperatures = [73, 74, 75, 71, 69, 72, 76, 73]
Output: [1, 1, 4, 2, 1, 1, 0, 0]
Explanation: Day 0: 1 day until 74. Day 1: 1 day until 75, etc.
```

**Approach:** Monotonic decreasing stack - store indices, pop when finding warmer.
- Time: O(n), Space: O(n)

```cpp
vector<int> dailyTemperatures(vector<int>& t) {
    vector<int> res(t.size(), 0);
    stack<int> st;  // Store indices in decreasing temperature order
    
    for (int i = 0; i < t.size(); i++) {
        // Pop indices of days with smaller temperatures
        while (!st.empty() && t[i] > t[st.top()]) {
            int prev = st.top();
            st.pop();
            res[prev] = i - prev;  // Days until warmer
        }
        st.push(i);  // Push current day
    }
    
    // Days not popped have no warmer day (remain 0)
    return res;
}
```

---

### Q79. Next Greater Element I
**Problem Statement:**
For each element in nums1 (subset of nums2), find next greater element in nums2.

**Example:**
```
Input: nums1 = [4, 1, 2], nums2 = [1, 3, 4, 2]
Output: [-1, 3, -1]
Explanation: 4 has no greater, 1's next greater is 3, 2 has no greater
```

**Approach:** Monotonic stack to precompute next greater for all elements in nums2.
- Time: O(n + m), Space: O(n)

```cpp
vector<int> nextGreaterElement(vector<int>& a, vector<int>& b) {
    unordered_map<int,int> mp;  // Element -> next greater
    stack<int> st;
    
    // Process nums2 from right to left
    for (int x : b) {
        // Pop smaller elements (they're less than current)
        while (!st.empty() && st.top() < x) {
            mp[st.top()] = x;  // Map smaller to current (greater)
            st.pop();
        }
        st.push(x);  // Push current element
    }
    
    // Build result for nums1
    vector<int> r;
    for (int x : a) {
        r.push_back(mp.count(x) ? mp[x] : -1);
    }
    return r;
}
```

---

### Q80. Next Greater Element II (Circular Array)
**Problem Statement:**
Find next greater element in circular array (wrap around allowed).

**Example:**
```
Input: nums = [1, 2, 1]
Output: [2, -1, 2]
Explanation: 1 -> 2 (next greater), 2 has none, 1 -> 2 (circular)
```

**Approach:** Process array twice to handle circular nature.
- Time: O(n), Space: O(n)

```cpp
vector<int> nextGreaterElements(vector<int>& nums) {
    int n = nums.size();
    vector<int> res(n, -1);
    stack<int> st;  // Store indices
    
    // Process array twice (for circular effect)
    for (int i = 0; i < 2 * n; i++) {
        // Pop indices with smaller values
        while (!st.empty() && nums[st.top()] < nums[i % n]) {
            res[st.top()] = nums[i % n];  // Found next greater
            st.pop();
        }
        
        // Only push indices from first pass
        if (i < n) st.push(i);
    }
    
    return res;
}
```

---

### Q81. Largest Rectangle in Histogram
**Problem Statement:**
Find the largest rectangular area in a histogram.

**Example:**
```
Input: heights = [2, 1, 5, 6, 2, 3]
Output: 10 (rectangle with height 5 and width 2)
```

**Approach:** Monotonic increasing stack to find boundaries efficiently.
- Time: O(n), Space: O(n)

```cpp
int largestRectangleArea(vector<int>& h) {
    h.push_back(0);  // Add sentinel to pop remaining bars
    stack<int> st;
    int best = 0;
    
    for (int i = 0; i < h.size(); i++) {
        // Pop bars taller than current (process them)
        while (!st.empty() && h[st.top()] > h[i]) {
            int top = st.top();
            st.pop();
            
            // Width: from after previous bar to current bar
            int w = st.empty() ? i : i - st.top() - 1;
            best = max(best, h[top] * w);
        }
        st.push(i);
    }
    
    return best;
}
```

---

### Q82. Maximal Rectangle
**Problem Statement:**
Find the largest rectangle containing only 1s in a binary matrix.

**Example:**
```
Input: matrix = [["1","0","1","0","0"],
                 ["1","0","1","1","1"],
                 ["1","1","1","1","1"],
                 ["1","0","0","1","0"]]
Output: 6 (2x3 rectangle)
```

**Approach:** Convert each row to histogram problem and use largestRectangleArea.
- Time: O(m*n), Space: O(n)

```cpp
int maximalRectangle(vector<vector<char>>& m) {
    if (m.empty()) return 0;
    
    vector<int> h(m[0].size(), 0);  // Heights for histogram
    int best = 0;
    
    // For each row
    for (auto& row : m) {
        // Update heights: increment if '1', reset if '0'
        for (int j = 0; j < row.size(); j++) {
            h[j] = row[j] == '1' ? h[j] + 1 : 0;
        }
        
        // Apply largest rectangle in histogram
        best = max(best, largestRectangleArea(h));
    }
    
    return best;
}
```

---

### Q83. Sliding Window Maximum
**Problem Statement:**
Find maximum value in each sliding window of size k.

**Example:**
```
Input: nums = [1,3,-1,-3,5,3,6,7], k = 3
Output: [3, 3, 5, 5, 6, 7]
Explanation: Windows: [1,3,-1]->3, [3,-1,-3]->3, [-1,-3,5]->5, etc.
```

**Approach:** Deque to maintain useful indices (decreasing order).
- Time: O(n), Space: O(k)

```cpp
vector<int> maxSlidingWindow(vector<int>& nums, int k) {
    deque<int> dq;  // Store indices in decreasing value order
    vector<int> res;
    
    for (int i = 0; i < nums.size(); i++) {
        // Remove indices outside current window
        while (!dq.empty() && dq.front() <= i - k) {
            dq.pop_front();
        }
        
        // Remove smaller elements from back (they won't be max)
        while (!dq.empty() && nums[dq.back()] < nums[i]) {
            dq.pop_back();
        }
        
        dq.push_back(i);  // Add current index
        
        // From window size k onward, add max to result
        if (i >= k - 1) {
            res.push_back(nums[dq.front()]);
        }
    }
    
    return res;
}
```

---

### Q84. Valid Parentheses
**Problem Statement:**
Check if parentheses are balanced and properly closed.

**Example:**
```
Input: s = "()[]{}"
Output: true
Input: s = "([)]"
Output: false
```

**Approach:** Stack - push opening brackets, pop and match closing brackets.
- Time: O(n), Space: O(n)

```cpp
bool isValid(string s) {
    stack<char> st;
    
    for (char c : s) {
        if (c == '(' || c == '[' || c == '{') {
            st.push(c);  // Push opening brackets
        } else {
            // Process closing bracket
            if (st.empty()) return false;  // No matching opening
            
            char t = st.top();
            st.pop();
            
            // Check if they match
            if ((c == ')' && t != '(') || 
                (c == ']' && t != '[') || 
                (c == '}' && t != '{')) {
                return false;
            }
        }
    }
    
    return st.empty();  // All brackets matched
}
```

---

### Q85. Decode String
**Problem Statement:**
Decode string with pattern k[string], where k is a digit.

**Example:**
```
Input: s = "3[a]2[bc]"
Output: "aaabcbc"
Explanation: 3 copies of "a", then 2 copies of "bc"
```

**Approach:** Two stacks (one for numbers, one for strings).
- Time: O(n), Space: O(n)

```cpp
string decodeString(string s) {
    stack<int> ns;      // Number stack
    stack<string> ss;   // String stack
    string cur = "";    // Current string being built
    int k = 0;          // Current number
    
    for (char c : s) {
        if (isdigit(c)) {
            // Build multi-digit numbers
            k = k * 10 + (c - '0');
        } else if (c == '[') {
            // Push state and reset for new nested level
            ns.push(k);
            ss.push(cur);
            k = 0;
            cur = "";
        } else if (c == ']') {
            // Pop and repeat string
            string t = cur;
            cur = ss.top();
            ss.pop();
            
            int rep = ns.top();
            ns.pop();
            
            // Repeat current string rep times
            while (rep--) {
                cur += t;
            }
        } else {
            // Regular character
            cur += c;
        }
    }
    
    return cur;
}
```

---

### Q86. Implement Queue using Stacks
**Problem Statement:**
Implement a queue using only two stacks.

**Example:**
```
MyQueue q;
q.push(1);
q.push(2);
q.pop();    // Returns 1 (FIFO)
q.peek();   // Returns 2
```

**Approach:** Use two stacks - input and output. Reverse when needed.
- Time: O(1) amortized, Space: O(n)

```cpp
class MyQueue {
    stack<int> in, out;  // Input and output stacks
    
public:
    void push(int x) {
        in.push(x);  // All pushes go to input stack
    }
    
    int pop() {
        peek();  // Ensure out stack is ready
        int v = out.top();
        out.pop();
        return v;
    }
    
    int peek() {
        // If output stack empty, reverse input stack
        if (out.empty()) {
            while (!in.empty()) {
                out.push(in.top());
                in.pop();
            }
        }
        return out.top();
    }
    
    bool empty() {
        return in.empty() && out.empty();
    }
};
```

---

### Q87. Implement Stack using Queues
**Problem Statement:**
Implement a stack using only one or two queues.

**Example:**
```
MyStack s;
s.push(1);
s.push(2);
s.pop();    // Returns 2 (LIFO)
s.top();    // Returns 1
```

**Approach:** Use one queue, reverse order by re-queuing on each push.
- Time: O(n) push, O(1) others, Space: O(n)

```cpp
class MyStack {
    queue<int> q;
    
public:
    void push(int x) {
        q.push(x);
        
        // Move elements to make x the front (like top of stack)
        for (int i = 1; i < q.size(); i++) {
            q.push(q.front());
            q.pop();
        }
    }
    
    int pop() {
        int v = q.front();
        q.pop();
        return v;
    }
    
    int top() {
        return q.front();
    }
    
    bool empty() {
        return q.empty();
    }
};
```

---

### Q88. Number of Visible People in a Queue
**Problem Statement:**
Count how many people each person can see to their right (taller blocks visibility).

**Example:**
```
Input: heights = [10, 6, 8, 5, 11, 9]
Output: [3, 1, 2, 1, 1, 0]
Explanation: 
- 10 can see 6, 8, 5 (blocked by 11)
- 6 can see 8 (blocked)
- 8 can see 5 (blocked by 11)
- etc.
```

**Approach:** Monotonic decreasing stack from right to left.
- Time: O(n), Space: O(n)

```cpp
vector<int> canSeePersonsCount(vector<int>& h) {
    int n = h.size();
    vector<int> res(n, 0);
    stack<int> st;  // Stack of heights in decreasing order
    
    // Process from right to left
    for (int i = n - 1; i >= 0; i--) {
        // Pop shorter people (person i can see them)
        while (!st.empty() && st.top() < h[i]) {
            res[i]++;
            st.pop();
        }
        
        // If stack not empty, person i can see top (taller person)
        if (!st.empty()) {
            res[i]++;
        }
        
        st.push(h[i]);
    }
    
    return res;
}
```

---

### Q89. Remove K Digits
**Problem Statement:**
Remove k digits to get the smallest possible number.

**Example:**
```
Input: num = "1432219", k = 3
Output: "1219"
Explanation: Remove 4, 3, 2 to get 1219
```

**Approach:** Greedy with stack - maintain increasing sequence.
- Time: O(n), Space: O(n)

```cpp
string removeKdigits(string num, int k) {
    string s;
    
    for (char c : num) {
        // Remove larger digits if we have removals left
        while (k && !s.empty() && s.back() > c) {
            s.pop_back();
            k--;
        }
        s += c;
    }
    
    // Remove remaining k digits from end if needed
    while (k--) {
        s.pop_back();
    }
    
    // Remove leading zeros
    int i = 0;
    while (i < s.size() && s[i] == '0') {
        i++;
    }
    s = s.substr(i);
    
    return s.empty() ? "0" : s;
}
```

---

### Q90. Asteroid Collision
**Problem Statement:**
Simulate asteroid collisions where positive values move right, negative move left.

**Example:**
```
Input: asteroids = [5, 10, -5]
Output: [5, 10]
Explanation: -5 collides with 10, 10 wins. 5 survives.
Input: asteroids = [8, -8]
Output: []
Explanation: They collide and destroy each other
```

**Approach:** Stack-based simulation with collision logic.
- Time: O(n), Space: O(n)

```cpp
vector<int> asteroidCollision(vector<int>& a) {
    vector<int> st;
    
    for (int x : a) {
        bool alive = true;
        
        // Left-moving asteroid might collide with right-moving ones
        while (alive && x < 0 && !st.empty() && st.back() > 0) {
            int right = st.back();
            
            if (right < -x) {
                // Right asteroid explodes, left continues
                st.pop_back();
            } else if (right == -x) {
                // Both explode
                st.pop_back();
                alive = false;
            } else {
                // Left asteroid explodes, right survives
                alive = false;
            }
        }
        
        if (alive) {
            st.push_back(x);
        }
    }
    
    return st;
}
```

---

## Summary Table for Q76-Q90

| Question | Topic | Approach | Time | Space |
|----------|-------|----------|------|-------|
| Q76 | Min Stack | Two Stacks | O(1) | O(n) |
| Q77 | Eval RPN | Stack | O(n) | O(n) |
| Q78 | Daily Temps | Monotonic Stack | O(n) | O(n) |
| Q79 | Next Greater I | Monotonic Stack | O(n+m) | O(n) |
| Q80 | Next Greater II | Monotonic Stack | O(n) | O(n) |
| Q81 | Largest Rectangle | Monotonic Stack | O(n) | O(n) |
| Q82 | Maximal Rectangle | Histogram | O(m*n) | O(n) |
| Q83 | Sliding Max | Deque | O(n) | O(k) |
| Q84 | Valid Parens | Stack | O(n) | O(n) |
| Q85 | Decode String | Two Stacks | O(n) | O(n) |
| Q86 | Queue Stack | Stack × 2 | O(1) amortized | O(n) |
| Q87 | Stack Queue | Queue | O(n) push | O(n) |
| Q88 | Visible People | Monotonic Stack | O(n) | O(n) |
| Q89 | Remove K Digits | Greedy Stack | O(n) | O(n) |
| Q90 | Asteroid Collision | Stack Simulation | O(n) | O(n) |

---

## Key Concepts Covered

### Monotonic Stack
- Maintains elements in increasing/decreasing order
- Efficiently finds next greater/smaller element
- Used in histogram problems

### Stack Applications
- Expression evaluation (RPN)
- Bracket matching
- Function call stack simulation
- Undo/Redo functionality

### Queue Applications
- BFS traversal
- Task scheduling
- Load balancing

### Sliding Window with Deque
- Maintains max/min in window
- Two-ended queue for efficient operations

---

**END OF PART 4 (Q76-Q90)**

Next: Part 5 will cover Binary Trees & BST (Q91-Q115)

