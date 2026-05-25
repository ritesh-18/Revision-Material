# 200 Most-Asked DSA Interview Questions - ANNOTATED PART 1

## Arrays (Q1–Q25) with Context & Comments

---

### Q1. Two Sum
**Problem Statement:**
Given an array of integers `nums` and an integer `target`, return the indices of the two numbers that add up to the target. You may assume each input has exactly one solution and cannot use the same element twice.

**Example:**
```
Input: nums = [2, 7, 11, 15], target = 9
Output: [0, 1]
Explanation: nums[0] + nums[1] = 2 + 7 = 9
```

**Approach:** Use a hash map to store the complement (target - current number) and its index.
- Time: O(n), Space: O(n)

```cpp
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int,int> mp;  // Store: {value, index}
    for (int i = 0; i < nums.size(); i++) {
        int comp = target - nums[i];  // Find complement
        if (mp.count(comp)) {  // If complement exists
            return {mp[comp], i};  // Return indices
        }
        mp[nums[i]] = i;  // Store current number and its index
    }
    return {};
}
```

---

### Q2. Best Time to Buy and Sell Stock
**Problem Statement:**
Given prices of a stock on different days, find the maximum profit you can make by buying on one day and selling on a later day. If no profit is possible, return 0.

**Example:**
```
Input: prices = [7, 1, 5, 3, 6, 4]
Output: 5
Explanation: Buy at 1, sell at 6, profit = 6 - 1 = 5
```

**Approach:** Track the minimum price seen so far and calculate profit at each step.
- Time: O(n), Space: O(1)

```cpp
int maxProfit(vector<int>& prices) {
    int minP = INT_MAX;  // Track the lowest price seen
    int profit = 0;      // Track maximum profit
    
    for (int p : prices) {
        minP = min(minP, p);           // Update minimum price
        profit = max(profit, p - minP); // Update profit if current price - min > profit
    }
    return profit;
}
```

---

### Q3. Contains Duplicate
**Problem Statement:**
Given an array of integers, determine if the array contains any duplicate values. Return true if duplicates exist, false otherwise.

**Example:**
```
Input: nums = [1, 2, 3, 1]
Output: true
Input: nums = [1, 2, 3, 4]
Output: false
```

**Approach:** Use a hash set to track seen elements.
- Time: O(n), Space: O(n)

```cpp
bool containsDuplicate(vector<int>& nums) {
    unordered_set<int> s;  // Set to track seen numbers
    
    for (int n : nums) {
        if (s.count(n)) return true;  // Found duplicate
        s.insert(n);                  // Add to set
    }
    return false;  // No duplicates found
}
```

---

### Q4. Product of Array Except Self
**Problem Statement:**
Given an array, return a new array where each element is the product of all other elements (without using division).

**Example:**
```
Input: nums = [1, 2, 3, 4]
Output: [24, 12, 8, 6]
Explanation: 
- res[0] = 2*3*4 = 24
- res[1] = 1*3*4 = 12
- res[2] = 1*2*4 = 8
- res[3] = 1*2*3 = 6
```

**Approach:** Use left-to-right and right-to-left passes.
- Time: O(n), Space: O(1) excluding output

```cpp
vector<int> productExceptSelf(vector<int>& nums) {
    int n = nums.size();
    vector<int> res(n, 1);  // Initialize result array
    
    // Left pass: res[i] = product of all elements to the left
    int left = 1;
    for (int i = 0; i < n; i++) {
        res[i] = left;
        left *= nums[i];
    }
    
    // Right pass: multiply by product of all elements to the right
    int right = 1;
    for (int i = n - 1; i >= 0; i--) {
        res[i] *= right;
        right *= nums[i];
    }
    return res;
}
```

---

### Q5. Maximum Subarray (Kadane's Algorithm)
**Problem Statement:**
Find the contiguous subarray with the largest sum and return that sum.

**Example:**
```
Input: nums = [-2, 1, -3, 4, -1, 2, 1, -5, 4]
Output: 6
Explanation: [4, -1, 2, 1] has the maximum sum of 6
```

**Approach:** Dynamic programming - track current sum and best sum.
- Time: O(n), Space: O(1)

```cpp
int maxSubArray(vector<int>& nums) {
    int cur = nums[0];   // Current subarray sum
    int best = nums[0];  // Maximum sum found
    
    for (int i = 1; i < nums.size(); i++) {
        // Either extend current subarray or start new one
        cur = max(nums[i], cur + nums[i]);
        // Update best if current is better
        best = max(best, cur);
    }
    return best;
}
```

---

### Q6. Maximum Product Subarray
**Problem Statement:**
Find the contiguous subarray with the largest product and return that product.

**Example:**
```
Input: nums = [2, 3, -2, 4]
Output: 6
Explanation: [2, 3] has maximum product 6
```

**Approach:** Track both maximum and minimum products (negative * negative = positive).
- Time: O(n), Space: O(1)

```cpp
int maxProduct(vector<int>& nums) {
    int mx = nums[0];  // Max product ending at current position
    int mn = nums[0];  // Min product ending at current position
    int ans = nums[0]; // Overall maximum
    
    for (int i = 1; i < nums.size(); i++) {
        // If negative, swap max and min (negative flips sign)
        if (nums[i] < 0) swap(mx, mn);
        
        // Update max: either current number or extend previous max
        mx = max(nums[i], mx * nums[i]);
        // Update min: either current number or extend previous min
        mn = min(nums[i], mn * nums[i]);
        
        // Update answer
        ans = max(ans, mx);
    }
    return ans;
}
```

---

### Q7. Find Minimum in Rotated Sorted Array
**Problem Statement:**
A sorted array is rotated at an unknown pivot. Find the minimum element. Array has no duplicates.

**Example:**
```
Input: nums = [3, 4, 5, 1, 2]
Output: 1
Explanation: The array was [1, 2, 3, 4, 5] rotated at index 3
```

**Approach:** Binary search - compare mid with right to decide which half contains minimum.
- Time: O(log n), Space: O(1)

```cpp
int findMin(vector<int>& nums) {
    int l = 0, r = nums.size() - 1;
    
    while (l < r) {
        int m = l + (r - l) / 2;
        
        // If mid > right, minimum is in right half
        if (nums[m] > nums[r]) {
            l = m + 1;
        }
        // Otherwise minimum is in left half (including mid)
        else {
            r = m;
        }
    }
    return nums[l];
}
```

---

### Q8. Search in Rotated Sorted Array
**Problem Statement:**
Search for a target in a rotated sorted array with no duplicates.

**Example:**
```
Input: nums = [4, 5, 6, 7, 0, 1, 2], target = 0
Output: 4
```

**Approach:** Binary search with logic to identify which half is sorted.
- Time: O(log n), Space: O(1)

```cpp
int search(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    
    while (l <= r) {
        int m = (l + r) / 2;
        if (nums[m] == target) return m;
        
        // Check which half is sorted
        if (nums[l] <= nums[m]) {  // Left half is sorted
            // If target is in sorted left half
            if (nums[l] <= target && target < nums[m]) {
                r = m - 1;  // Search left
            } else {
                l = m + 1;  // Search right
            }
        } else {  // Right half is sorted
            // If target is in sorted right half
            if (nums[m] < target && target <= nums[r]) {
                l = m + 1;  // Search right
            } else {
                r = m - 1;  // Search left
            }
        }
    }
    return -1;  // Not found
}
```

---

### Q9. 3Sum
**Problem Statement:**
Find all unique triplets in an array that sum to a target (0 by default). Return no duplicate triplets.

**Example:**
```
Input: nums = [-1, 0, 1, 2, -1, -4]
Output: [[-1, -1, 2], [-1, 0, 1]]
```

**Approach:** Sort array, then for each element use two-pointer approach on remaining array.
- Time: O(n²), Space: O(1) excluding output

```cpp
vector<vector<int>> threeSum(vector<int>& nums) {
    sort(nums.begin(), nums.end());  // Sort first
    vector<vector<int>> res;
    
    for (int i = 0; i < (int)nums.size() - 2; i++) {
        // Skip duplicate values
        if (i && nums[i] == nums[i-1]) continue;
        
        // Two-pointer approach for remaining elements
        int l = i + 1, r = nums.size() - 1;
        
        while (l < r) {
            int s = nums[i] + nums[l] + nums[r];
            
            if (s == 0) {
                res.push_back({nums[i], nums[l], nums[r]});
                
                // Skip duplicates
                while (l < r && nums[l] == nums[l+1]) l++;
                while (l < r && nums[r] == nums[r-1]) r--;
                
                l++; r--;
            } else if (s < 0) {
                l++;
            } else {
                r--;
            }
        }
    }
    return res;
}
```

---

### Q10. Container With Most Water
**Problem Statement:**
Given heights of containers, find two lines that form a container holding the most water.

**Example:**
```
Input: height = [1, 8, 6, 2, 5, 4, 8, 3, 7]
Output: 49
Explanation: Lines at index 1 and 8, area = min(8,7) * (8-1) = 49
```

**Approach:** Two-pointer from both ends, move pointer with smaller height inward.
- Time: O(n), Space: O(1)

```cpp
int maxArea(vector<int>& h) {
    int l = 0, r = h.size() - 1;
    int best = 0;
    
    while (l < r) {
        // Area = height * width
        best = max(best, min(h[l], h[r]) * (r - l));
        
        // Move the pointer with smaller height
        if (h[l] < h[r]) {
            l++;
        } else {
            r--;
        }
    }
    return best;
}
```

---

### Q11. Move Zeroes
**Problem Statement:**
Move all zeros to the end of array while maintaining relative order of non-zero elements (in-place).

**Example:**
```
Input: nums = [0, 1, 0, 3, 12]
Output: [1, 3, 12, 0, 0]
```

**Approach:** Use pointer to track position for next non-zero element.
- Time: O(n), Space: O(1)

```cpp
void moveZeroes(vector<int>& nums) {
    int j = 0;  // Position to place next non-zero element
    
    for (int i = 0; i < nums.size(); i++) {
        // If current element is non-zero
        if (nums[i]) {
            swap(nums[i], nums[j++]);  // Swap to position j
        }
    }
}
```

---

### Q12. Rotate Array
**Problem Statement:**
Rotate array k steps to the right in-place.

**Example:**
```
Input: nums = [1, 2, 3, 4, 5], k = 2
Output: [4, 5, 1, 2, 3]
```

**Approach:** Use reverse operation three times.
- Time: O(n), Space: O(1)

```cpp
void rotate(vector<int>& nums, int k) {
    k %= nums.size();  // Handle k > size
    
    // Reverse entire array
    reverse(nums.begin(), nums.end());
    
    // Reverse first k elements
    reverse(nums.begin(), nums.begin() + k);
    
    // Reverse remaining elements
    reverse(nums.begin() + k, nums.end());
}
```

---

### Q13. Find the Duplicate Number (Floyd's Cycle Detection)
**Problem Statement:**
Given an array with n+1 integers where each is between 1 and n, find the duplicate number. There's always one duplicate but may be multiple occurrences.

**Example:**
```
Input: nums = [1, 3, 4, 2, 2]
Output: 2
```

**Approach:** Treat array as linked list (value = next index), use cycle detection.
- Time: O(n), Space: O(1)

```cpp
int findDuplicate(vector<int>& nums) {
    // Phase 1: Find intersection point in cycle
    int slow = nums[0], fast = nums[0];
    
    do {
        slow = nums[slow];              // Move 1 step
        fast = nums[nums[fast]];        // Move 2 steps
    } while (slow != fast);  // Until they meet
    
    // Phase 2: Find cycle start (duplicate number)
    slow = nums[0];
    while (slow != fast) {
        slow = nums[slow];
        fast = nums[fast];
    }
    return slow;
}
```

---

### Q14. Merge Intervals
**Problem Statement:**
Given overlapping intervals, merge overlapping intervals into a single interval.

**Example:**
```
Input: intervals = [[1,3], [2,6], [8,10], [15,18]]
Output: [[1,6], [8,10], [15,18]]
```

**Approach:** Sort by start, then merge overlapping intervals.
- Time: O(n log n), Space: O(n)

```cpp
vector<vector<int>> merge(vector<vector<int>>& iv) {
    sort(iv.begin(), iv.end());  // Sort by start position
    vector<vector<int>> res;
    
    for (auto& cur : iv) {
        // If no intervals yet or current doesn't overlap with last
        if (!res.empty() && cur[0] <= res.back()[1]) {
            // Merge: extend the end of last interval
            res.back()[1] = max(res.back()[1], cur[1]);
        } else {
            // No overlap, add new interval
            res.push_back(cur);
        }
    }
    return res;
}
```

---

### Q15. Insert Interval
**Problem Statement:**
Insert a new interval into a list of non-overlapping intervals and merge if necessary.

**Example:**
```
Input: intervals = [[1,2],[3,5],[6,9]], newInterval = [4,8]
Output: [[1,2],[3,8],[6,9]]
```

**Approach:** Add all non-overlapping intervals before and after, merge overlapping ones.
- Time: O(n), Space: O(n)

```cpp
vector<vector<int>> insert(vector<vector<int>>& iv, vector<int> ni) {
    vector<vector<int>> res;
    int i = 0, n = iv.size();
    
    // Add all intervals that end before new interval starts
    while (i < n && iv[i][1] < ni[0]) {
        res.push_back(iv[i++]);
    }
    
    // Merge overlapping intervals
    while (i < n && iv[i][0] <= ni[1]) {
        ni[0] = min(ni[0], iv[i][0]);  // Extend start
        ni[1] = max(ni[1], iv[i][1]);  // Extend end
        i++;
    }
    res.push_back(ni);
    
    // Add remaining intervals
    while (i < n) {
        res.push_back(iv[i++]);
    }
    return res;
}
```

---

### Q16. Jump Game
**Problem Statement:**
Determine if you can reach the last index starting from the first index. At each position, the value represents max jump length.

**Example:**
```
Input: nums = [2, 3, 1, 1, 4]
Output: true
Explanation: Jump 1 step from index 0 to 1, then 3 steps to last index
```

**Approach:** Track the farthest position reachable.
- Time: O(n), Space: O(1)

```cpp
bool canJump(vector<int>& nums) {
    int reach = 0;  // Farthest index we can reach
    
    for (int i = 0; i < nums.size(); i++) {
        // If current index is beyond our reach, cannot proceed
        if (i > reach) return false;
        
        // Update how far we can reach
        reach = max(reach, i + nums[i]);
    }
    return true;
}
```

---

### Q17. Jump Game II
**Problem Statement:**
Find minimum number of jumps needed to reach the last index.

**Example:**
```
Input: nums = [2, 3, 1, 1, 4]
Output: 2
Explanation: Minimum 2 jumps: index 0 -> 1 -> last index
```

**Approach:** Greedy - jump to position that allows furthest reach.
- Time: O(n), Space: O(1)

```cpp
int jump(vector<int>& nums) {
    int jumps = 0;  // Number of jumps
    int end = 0;    // End of current jump range
    int far = 0;    // Farthest position reachable
    
    for (int i = 0; i < nums.size() - 1; i++) {
        // Update farthest position we can reach
        far = max(far, i + nums[i]);
        
        // If we've reached the end of current jump range
        if (i == end) {
            jumps++;        // Make another jump
            end = far;      // Update the end to new far position
        }
    }
    return jumps;
}
```

---

### Q18. Gas Station
**Problem Statement:**
Determine if you can complete a circuit starting from any gas station, where you consume gas while traveling.

**Example:**
```
Input: gas = [1,2,3,4,5], cost = [3,4,5,1,2]
Output: 3
Explanation: Start at station 3 and you have enough gas to return to station 3
```

**Approach:** Greedy - if total gas < total cost, impossible. Otherwise, find valid start.
- Time: O(n), Space: O(1)

```cpp
int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
    int total = 0;  // Track if circuit is possible
    int tank = 0;   // Current tank level
    int start = 0;  // Starting position
    
    for (int i = 0; i < gas.size(); i++) {
        int d = gas[i] - cost[i];  // Net gas at current station
        total += d;
        tank += d;
        
        // If tank goes negative, current start is invalid
        if (tank < 0) {
            start = i + 1;  // Try next station
            tank = 0;       // Reset tank
        }
    }
    // If total is negative, impossible. Otherwise start is valid
    return total < 0 ? -1 : start;
}
```

---

### Q19. First Missing Positive
**Problem Statement:**
Find the smallest missing positive integer in an unsorted array (in-place preferred).

**Example:**
```
Input: nums = [3, 4, -1, 1]
Output: 2
Explanation: 1 is present, 2 is missing
```

**Approach:** Place each positive integer in its correct position (value k at index k-1).
- Time: O(n), Space: O(1)

```cpp
int firstMissingPositive(vector<int>& nums) {
    int n = nums.size();
    
    // Place each number in correct position if possible
    for (int i = 0; i < n; i++) {
        // If number is between 1 and n and not in correct position
        while (nums[i] > 0 && nums[i] <= n && nums[nums[i]-1] != nums[i]) {
            // Swap to correct position
            swap(nums[i], nums[nums[i]-1]);
        }
    }
    
    // Find first missing positive
    for (int i = 0; i < n; i++) {
        if (nums[i] != i+1) return i+1;
    }
    return n + 1;  // All 1 to n present
}
```

---

### Q20. Trapping Rain Water
**Problem Statement:**
Calculate the amount of water that can be trapped after raining given elevation heights.

**Example:**
```
Input: height = [0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1]
Output: 6
Explanation: 6 units of water can be trapped
```

**Approach:** Two-pointer - track max height from left and right.
- Time: O(n), Space: O(1)

```cpp
int trap(vector<int>& h) {
    int l = 0, r = h.size() - 1;  // Two pointers
    int lMax = 0, rMax = 0;       // Max height from each side
    int water = 0;                // Total water trapped
    
    while (l < r) {
        if (h[l] < h[r]) {
            // Process left side
            lMax = max(lMax, h[l]);
            // Water trapped at position l
            water += lMax - h[l];
            l++;
        } else {
            // Process right side
            rMax = max(rMax, h[r]);
            // Water trapped at position r
            water += rMax - h[r];
            r--;
        }
    }
    return water;
}
```

---

### Q21. Median of Two Sorted Arrays
**Problem Statement:**
Find the median of two sorted arrays.

**Example:**
```
Input: nums1 = [1, 3], nums2 = [2]
Output: 2.0
Explanation: Merged array would be [1, 2, 3], median is 2
```

**Approach:** Binary search - partition arrays so left and right halves are balanced.
- Time: O(log(min(m,n))), Space: O(1)

```cpp
double findMedianSortedArrays(vector<int>& a, vector<int>& b) {
    // Ensure a is smaller array for binary search on smaller array
    if (a.size() > b.size()) return findMedianSortedArrays(b, a);
    
    int m = a.size(), n = b.size();
    int lo = 0, hi = m;
    
    while (lo <= hi) {
        int i = (lo + hi) / 2;  // Partition point in a
        int j = (m + n + 1) / 2 - i;  // Partition point in b
        
        // Get values at partition points
        int aL = i == 0 ? INT_MIN : a[i-1];
        int aR = i == m ? INT_MAX : a[i];
        int bL = j == 0 ? INT_MIN : b[j-1];
        int bR = j == n ? INT_MAX : b[j];
        
        // Check if partition is correct
        if (aL <= bR && bL <= aR) {
            // If odd total length, return larger of left elements
            if ((m+n) % 2) return max(aL, bL);
            // If even, return average of max(left) and min(right)
            return (max(aL,bL) + min(aR,bR)) / 2.0;
        } else if (aL > bR) {
            // a's partition is too far right
            hi = i - 1;
        } else {
            // a's partition is too far left
            lo = i + 1;
        }
    }
    return 0;
}
```

---

### Q22. Majority Element (Boyer-Moore Voting Algorithm)
**Problem Statement:**
Find element appearing more than n/2 times in array.

**Example:**
```
Input: nums = [3, 2, 3]
Output: 3
```

**Approach:** Boyer-Moore voting - track candidate and count.
- Time: O(n), Space: O(1)

```cpp
int majorityElement(vector<int>& nums) {
    int cand = 0;  // Current candidate
    int cnt = 0;   // Count for current candidate
    
    for (int n : nums) {
        // If count is 0, start new candidate
        if (cnt == 0) cand = n;
        
        // Increment if matches candidate, decrement otherwise
        cnt += (n == cand) ? 1 : -1;
    }
    return cand;
}
```

---

### Q23. Single Number
**Problem Statement:**
Every element appears twice except one. Find that single element.

**Example:**
```
Input: nums = [4, 1, 2, 1, 2]
Output: 4
```

**Approach:** XOR cancels out pairs (a ^ a = 0, a ^ 0 = a).
- Time: O(n), Space: O(1)

```cpp
int singleNumber(vector<int>& nums) {
    int x = 0;  // Running XOR result
    
    for (int n : nums) {
        x ^= n;  // XOR with each number
    }
    return x;  // All pairs cancel, only single remains
}
```

---

### Q24. Subarray Sum Equals K
**Problem Statement:**
Find the number of subarrays whose sum equals a target value k.

**Example:**
```
Input: nums = [1, 1, 1], k = 2
Output: 2
Explanation: [1,1] and [1,1] sum to 2
```

**Approach:** Prefix sum with hash map - count how many times (current_sum - k) occurred.
- Time: O(n), Space: O(n)

```cpp
int subarraySum(vector<int>& nums, int k) {
    unordered_map<int,int> mp{{0,1}};  // {prefix_sum, count}
    int sum = 0, cnt = 0;
    
    for (int n : nums) {
        sum += n;  // Update prefix sum
        
        // If (sum - k) exists, add its count
        if (mp.count(sum - k)) {
            cnt += mp[sum - k];
        }
        
        // Add current sum to map
        mp[sum]++;
    }
    return cnt;
}
```

---

### Q25. Longest Consecutive Sequence
**Problem Statement:**
Find the length of the longest consecutive element sequence in an unsorted array.

**Example:**
```
Input: nums = [100, 4, 200, 1, 3, 2]
Output: 4
Explanation: [1, 2, 3, 4] is the longest
```

**Approach:** Use hash set, only start counting from sequence start (num-1 not present).
- Time: O(n), Space: O(n)

```cpp
int longestConsecutive(vector<int>& nums) {
    unordered_set<int> s(nums.begin(), nums.end());
    int best = 0;
    
    for (int n : s) {
        // Only start counting from sequence start
        if (!s.count(n - 1)) {
            int cur = n, len = 1;
            
            // Count consecutive numbers
            while (s.count(cur + 1)) {
                cur++;
                len++;
            }
            best = max(best, len);
        }
    }
    return best;
}
```

---

## Summary for Q1-Q25

| Question | Topic | Approach | Time | Space |
|----------|-------|----------|------|-------|
| Q1 | Two Sum | Hash Map | O(n) | O(n) |
| Q2 | Max Profit | Greedy/DP | O(n) | O(1) |
| Q3 | Contains Duplicate | Hash Set | O(n) | O(n) |
| Q4 | Product Except Self | Prefix Product | O(n) | O(1) |
| Q5 | Max Subarray | Kadane's | O(n) | O(1) |
| Q6 | Max Product Subarray | DP with Min/Max | O(n) | O(1) |
| Q7 | Find Min in Rotated | Binary Search | O(log n) | O(1) |
| Q8 | Search Rotated Array | Binary Search | O(log n) | O(1) |
| Q9 | 3Sum | Sorting + Two Pointer | O(n²) | O(1) |
| Q10 | Container Water | Two Pointer | O(n) | O(1) |
| Q11 | Move Zeroes | Two Pointer | O(n) | O(1) |
| Q12 | Rotate Array | Reverse | O(n) | O(1) |
| Q13 | Find Duplicate | Floyd's Cycle | O(n) | O(1) |
| Q14 | Merge Intervals | Sorting | O(n log n) | O(n) |
| Q15 | Insert Interval | Two Pointers | O(n) | O(n) |
| Q16 | Jump Game | Greedy | O(n) | O(1) |
| Q17 | Jump Game II | Greedy | O(n) | O(1) |
| Q18 | Gas Station | Greedy | O(n) | O(1) |
| Q19 | First Missing Positive | In-place Sorting | O(n) | O(1) |
| Q20 | Trapping Rain Water | Two Pointer | O(n) | O(1) |
| Q21 | Median 2 Arrays | Binary Search | O(log(min(m,n))) | O(1) |
| Q22 | Majority Element | Boyer-Moore | O(n) | O(1) |
| Q23 | Single Number | XOR | O(n) | O(1) |
| Q24 | Subarray Sum K | Prefix Sum + Hash | O(n) | O(n) |
| Q25 | Longest Consecutive | Hash Set | O(n) | O(n) |

