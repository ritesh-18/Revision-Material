# 200 Most-Asked DSA Interview Questions (C++ Solutions)

> Comprehensive coverage of company-favorite DSA problems. Solutions in modern C++.

## Table of Contents
1. [Arrays (Q1–Q30)](#arrays-q1q30)
2. [Strings (Q31–Q55)](#strings-q31q55)
3. [Linked List (Q56–Q75)](#linked-list-q56q75)
4. [Stack & Queue (Q76–Q90)](#stack--queue-q76q90)
5. [Binary Tree / BST (Q91–Q115)](#binary-tree--bst-q91q115)
6. [Graphs (Q116–Q135)](#graphs-q116q135)
7. [Dynamic Programming (Q136–Q165)](#dynamic-programming-q136q165)
8. [Backtracking (Q166–Q175)](#backtracking-q166q175)
9. [Heaps / Priority Queue (Q176–Q183)](#heaps--priority-queue-q176q183)
10. [Binary Search (Q184–Q190)](#binary-search-q184q190)
11. [Bit Manipulation (Q191–Q195)](#bit-manipulation-q191q195)
12. [Math (Q196–Q200)](#math-q196q200)

---

## Arrays (Q1–Q30)

### Q1. Two Sum
```cpp
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int,int> mp;
    for (int i = 0; i < nums.size(); i++) {
        int comp = target - nums[i];
        if (mp.count(comp)) return {mp[comp], i};
        mp[nums[i]] = i;
    }
    return {};
}
```

### Q2. Best Time to Buy and Sell Stock
```cpp
int maxProfit(vector<int>& prices) {
    int minP = INT_MAX, profit = 0;
    for (int p : prices) {
        minP = min(minP, p);
        profit = max(profit, p - minP);
    }
    return profit;
}
```

### Q3. Contains Duplicate
```cpp
bool containsDuplicate(vector<int>& nums) {
    unordered_set<int> s;
    for (int n : nums) {
        if (s.count(n)) return true;
        s.insert(n);
    }
    return false;
}
```

### Q4. Product of Array Except Self
```cpp
vector<int> productExceptSelf(vector<int>& nums) {
    int n = nums.size();
    vector<int> res(n, 1);
    int left = 1;
    for (int i = 0; i < n; i++) { res[i] = left; left *= nums[i]; }
    int right = 1;
    for (int i = n - 1; i >= 0; i--) { res[i] *= right; right *= nums[i]; }
    return res;
}
```

### Q5. Maximum Subarray (Kadane's)
```cpp
int maxSubArray(vector<int>& nums) {
    int cur = nums[0], best = nums[0];
    for (int i = 1; i < nums.size(); i++) {
        cur = max(nums[i], cur + nums[i]);
        best = max(best, cur);
    }
    return best;
}
```

### Q6. Maximum Product Subarray
```cpp
int maxProduct(vector<int>& nums) {
    int mx = nums[0], mn = nums[0], ans = nums[0];
    for (int i = 1; i < nums.size(); i++) {
        if (nums[i] < 0) swap(mx, mn);
        mx = max(nums[i], mx * nums[i]);
        mn = min(nums[i], mn * nums[i]);
        ans = max(ans, mx);
    }
    return ans;
}
```

### Q7. Find Minimum in Rotated Sorted Array
```cpp
int findMin(vector<int>& nums) {
    int l = 0, r = nums.size() - 1;
    while (l < r) {
        int m = l + (r - l) / 2;
        if (nums[m] > nums[r]) l = m + 1;
        else r = m;
    }
    return nums[l];
}
```

### Q8. Search in Rotated Sorted Array
```cpp
int search(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
        int m = (l + r) / 2;
        if (nums[m] == target) return m;
        if (nums[l] <= nums[m]) {
            if (nums[l] <= target && target < nums[m]) r = m - 1;
            else l = m + 1;
        } else {
            if (nums[m] < target && target <= nums[r]) l = m + 1;
            else r = m - 1;
        }
    }
    return -1;
}
```

### Q9. 3Sum
```cpp
vector<vector<int>> threeSum(vector<int>& nums) {
    sort(nums.begin(), nums.end());
    vector<vector<int>> res;
    for (int i = 0; i < (int)nums.size() - 2; i++) {
        if (i && nums[i] == nums[i-1]) continue;
        int l = i + 1, r = nums.size() - 1;
        while (l < r) {
            int s = nums[i] + nums[l] + nums[r];
            if (s == 0) {
                res.push_back({nums[i], nums[l], nums[r]});
                while (l < r && nums[l] == nums[l+1]) l++;
                while (l < r && nums[r] == nums[r-1]) r--;
                l++; r--;
            } else if (s < 0) l++;
            else r--;
        }
    }
    return res;
}
```

### Q10. Container With Most Water
```cpp
int maxArea(vector<int>& h) {
    int l = 0, r = h.size() - 1, best = 0;
    while (l < r) {
        best = max(best, min(h[l], h[r]) * (r - l));
        if (h[l] < h[r]) l++; else r--;
    }
    return best;
}
```

### Q11. Move Zeroes
```cpp
void moveZeroes(vector<int>& nums) {
    int j = 0;
    for (int i = 0; i < nums.size(); i++)
        if (nums[i]) swap(nums[i], nums[j++]);
}
```

### Q12. Rotate Array
```cpp
void rotate(vector<int>& nums, int k) {
    k %= nums.size();
    reverse(nums.begin(), nums.end());
    reverse(nums.begin(), nums.begin() + k);
    reverse(nums.begin() + k, nums.end());
}
```

### Q13. Find the Duplicate Number (Floyd's)
```cpp
int findDuplicate(vector<int>& nums) {
    int slow = nums[0], fast = nums[0];
    do { slow = nums[slow]; fast = nums[nums[fast]]; } while (slow != fast);
    slow = nums[0];
    while (slow != fast) { slow = nums[slow]; fast = nums[fast]; }
    return slow;
}
```

### Q14. Merge Intervals
```cpp
vector<vector<int>> merge(vector<vector<int>>& iv) {
    sort(iv.begin(), iv.end());
    vector<vector<int>> res;
    for (auto& cur : iv) {
        if (!res.empty() && cur[0] <= res.back()[1])
            res.back()[1] = max(res.back()[1], cur[1]);
        else res.push_back(cur);
    }
    return res;
}
```

### Q15. Insert Interval
```cpp
vector<vector<int>> insert(vector<vector<int>>& iv, vector<int> ni) {
    vector<vector<int>> res;
    int i = 0, n = iv.size();
    while (i < n && iv[i][1] < ni[0]) res.push_back(iv[i++]);
    while (i < n && iv[i][0] <= ni[1]) {
        ni[0] = min(ni[0], iv[i][0]);
        ni[1] = max(ni[1], iv[i][1]);
        i++;
    }
    res.push_back(ni);
    while (i < n) res.push_back(iv[i++]);
    return res;
}
```

### Q16. Jump Game
```cpp
bool canJump(vector<int>& nums) {
    int reach = 0;
    for (int i = 0; i < nums.size(); i++) {
        if (i > reach) return false;
        reach = max(reach, i + nums[i]);
    }
    return true;
}
```

### Q17. Jump Game II
```cpp
int jump(vector<int>& nums) {
    int jumps = 0, end = 0, far = 0;
    for (int i = 0; i < nums.size() - 1; i++) {
        far = max(far, i + nums[i]);
        if (i == end) { jumps++; end = far; }
    }
    return jumps;
}
```

### Q18. Gas Station
```cpp
int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
    int total = 0, tank = 0, start = 0;
    for (int i = 0; i < gas.size(); i++) {
        int d = gas[i] - cost[i];
        total += d; tank += d;
        if (tank < 0) { start = i + 1; tank = 0; }
    }
    return total < 0 ? -1 : start;
}
```

### Q19. First Missing Positive
```cpp
int firstMissingPositive(vector<int>& nums) {
    int n = nums.size();
    for (int i = 0; i < n; i++)
        while (nums[i] > 0 && nums[i] <= n && nums[nums[i]-1] != nums[i])
            swap(nums[i], nums[nums[i]-1]);
    for (int i = 0; i < n; i++) if (nums[i] != i+1) return i+1;
    return n + 1;
}
```

### Q20. Trapping Rain Water
```cpp
int trap(vector<int>& h) {
    int l = 0, r = h.size() - 1, lMax = 0, rMax = 0, water = 0;
    while (l < r) {
        if (h[l] < h[r]) {
            lMax = max(lMax, h[l]);
            water += lMax - h[l++];
        } else {
            rMax = max(rMax, h[r]);
            water += rMax - h[r--];
        }
    }
    return water;
}
```

### Q21. Median of Two Sorted Arrays
```cpp
double findMedianSortedArrays(vector<int>& a, vector<int>& b) {
    if (a.size() > b.size()) return findMedianSortedArrays(b, a);
    int m = a.size(), n = b.size(), lo = 0, hi = m;
    while (lo <= hi) {
        int i = (lo + hi) / 2, j = (m + n + 1) / 2 - i;
        int aL = i == 0 ? INT_MIN : a[i-1];
        int aR = i == m ? INT_MAX : a[i];
        int bL = j == 0 ? INT_MIN : b[j-1];
        int bR = j == n ? INT_MAX : b[j];
        if (aL <= bR && bL <= aR) {
            if ((m+n) % 2) return max(aL, bL);
            return (max(aL,bL) + min(aR,bR)) / 2.0;
        } else if (aL > bR) hi = i - 1;
        else lo = i + 1;
    }
    return 0;
}
```

### Q22. Majority Element (Boyer-Moore)
```cpp
int majorityElement(vector<int>& nums) {
    int cand = 0, cnt = 0;
    for (int n : nums) {
        if (cnt == 0) cand = n;
        cnt += (n == cand) ? 1 : -1;
    }
    return cand;
}
```

### Q23. Single Number
```cpp
int singleNumber(vector<int>& nums) {
    int x = 0;
    for (int n : nums) x ^= n;
    return x;
}
```

### Q24. Subarray Sum Equals K
```cpp
int subarraySum(vector<int>& nums, int k) {
    unordered_map<int,int> mp{{0,1}};
    int sum = 0, cnt = 0;
    for (int n : nums) {
        sum += n;
        if (mp.count(sum - k)) cnt += mp[sum - k];
        mp[sum]++;
    }
    return cnt;
}
```

### Q25. Longest Consecutive Sequence
```cpp
int longestConsecutive(vector<int>& nums) {
    unordered_set<int> s(nums.begin(), nums.end());
    int best = 0;
    for (int n : s) {
        if (!s.count(n - 1)) {
            int cur = n, len = 1;
            while (s.count(cur + 1)) { cur++; len++; }
            best = max(best, len);
        }
    }
    return best;
}
```

### Q26. Find All Duplicates in an Array
```cpp
vector<int> findDuplicates(vector<int>& nums) {
    vector<int> res;
    for (int n : nums) {
        int idx = abs(n) - 1;
        if (nums[idx] < 0) res.push_back(idx + 1);
        else nums[idx] = -nums[idx];
    }
    return res;
}
```

### Q27. Set Matrix Zeroes
```cpp
void setZeroes(vector<vector<int>>& m) {
    int rows = m.size(), cols = m[0].size();
    bool firstCol = false;
    for (int i = 0; i < rows; i++) {
        if (m[i][0] == 0) firstCol = true;
        for (int j = 1; j < cols; j++)
            if (m[i][j] == 0) m[i][0] = m[0][j] = 0;
    }
    for (int i = rows - 1; i >= 0; i--) {
        for (int j = cols - 1; j >= 1; j--)
            if (m[i][0] == 0 || m[0][j] == 0) m[i][j] = 0;
        if (firstCol) m[i][0] = 0;
    }
}
```

### Q28. Spiral Matrix
```cpp
vector<int> spiralOrder(vector<vector<int>>& m) {
    vector<int> res;
    int top = 0, bot = m.size() - 1, lt = 0, rt = m[0].size() - 1;
    while (top <= bot && lt <= rt) {
        for (int j = lt; j <= rt; j++) res.push_back(m[top][j]); top++;
        for (int i = top; i <= bot; i++) res.push_back(m[i][rt]); rt--;
        if (top <= bot) { for (int j = rt; j >= lt; j--) res.push_back(m[bot][j]); bot--; }
        if (lt <= rt) { for (int i = bot; i >= top; i--) res.push_back(m[i][lt]); lt++; }
    }
    return res;
}
```

### Q29. Rotate Image 90° Clockwise
```cpp
void rotate(vector<vector<int>>& m) {
    int n = m.size();
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++) swap(m[i][j], m[j][i]);
    for (auto& row : m) reverse(row.begin(), row.end());
}
```

### Q30. Pascal's Triangle
```cpp
vector<vector<int>> generate(int n) {
    vector<vector<int>> r(n);
    for (int i = 0; i < n; i++) {
        r[i].assign(i + 1, 1);
        for (int j = 1; j < i; j++) r[i][j] = r[i-1][j-1] + r[i-1][j];
    }
    return r;
}
```

---

## Strings (Q31–Q55)

### Q31. Valid Anagram
```cpp
bool isAnagram(string s, string t) {
    if (s.size() != t.size()) return false;
    int c[26] = {};
    for (int i = 0; i < s.size(); i++) { c[s[i]-'a']++; c[t[i]-'a']--; }
    for (int x : c) if (x) return false;
    return true;
}
```

### Q32. Valid Palindrome
```cpp
bool isPalindrome(string s) {
    int l = 0, r = s.size() - 1;
    while (l < r) {
        while (l < r && !isalnum(s[l])) l++;
        while (l < r && !isalnum(s[r])) r--;
        if (tolower(s[l++]) != tolower(s[r--])) return false;
    }
    return true;
}
```

### Q33. Longest Substring Without Repeating Characters
```cpp
int lengthOfLongestSubstring(string s) {
    unordered_map<char,int> mp;
    int l = 0, best = 0;
    for (int r = 0; r < s.size(); r++) {
        if (mp.count(s[r]) && mp[s[r]] >= l) l = mp[s[r]] + 1;
        mp[s[r]] = r;
        best = max(best, r - l + 1);
    }
    return best;
}
```

### Q34. Longest Palindromic Substring
```cpp
string longestPalindrome(string s) {
    int n = s.size(), start = 0, mx = 1;
    auto expand = [&](int l, int r) {
        while (l >= 0 && r < n && s[l] == s[r]) { l--; r++; }
        if (r - l - 1 > mx) { mx = r - l - 1; start = l + 1; }
    };
    for (int i = 0; i < n; i++) { expand(i, i); expand(i, i + 1); }
    return s.substr(start, mx);
}
```

### Q35. Palindromic Substrings (count)
```cpp
int countSubstrings(string s) {
    int n = s.size(), cnt = 0;
    auto expand = [&](int l, int r) {
        while (l >= 0 && r < n && s[l--] == s[r++]) cnt++;
    };
    for (int i = 0; i < n; i++) { expand(i, i); expand(i, i + 1); }
    return cnt;
}
```

### Q36. Group Anagrams
```cpp
vector<vector<string>> groupAnagrams(vector<string>& strs) {
    unordered_map<string, vector<string>> mp;
    for (auto& s : strs) { string k = s; sort(k.begin(), k.end()); mp[k].push_back(s); }
    vector<vector<string>> res;
    for (auto& p : mp) res.push_back(p.second);
    return res;
}
```

### Q37. Encode and Decode Strings
```cpp
string encode(vector<string>& v) {
    string s;
    for (auto& x : v) s += to_string(x.size()) + "#" + x;
    return s;
}
vector<string> decode(string s) {
    vector<string> r;
    int i = 0;
    while (i < s.size()) {
        int j = s.find('#', i);
        int n = stoi(s.substr(i, j - i));
        r.push_back(s.substr(j + 1, n));
        i = j + 1 + n;
    }
    return r;
}
```

### Q38. Minimum Window Substring
```cpp
string minWindow(string s, string t) {
    if (t.empty()) return "";
    unordered_map<char,int> need, have;
    for (char c : t) need[c]++;
    int req = need.size(), made = 0, l = 0, bestL = 0, bestLen = INT_MAX;
    for (int r = 0; r < s.size(); r++) {
        have[s[r]]++;
        if (need.count(s[r]) && have[s[r]] == need[s[r]]) made++;
        while (made == req) {
            if (r - l + 1 < bestLen) { bestLen = r - l + 1; bestL = l; }
            have[s[l]]--;
            if (need.count(s[l]) && have[s[l]] < need[s[l]]) made--;
            l++;
        }
    }
    return bestLen == INT_MAX ? "" : s.substr(bestL, bestLen);
}
```

### Q39. Reverse Words in a String
```cpp
string reverseWords(string s) {
    stringstream ss(s); string w, res;
    while (ss >> w) res = w + (res.empty() ? "" : " " + res);
    return res;
}
```

### Q40. String to Integer (atoi)
```cpp
int myAtoi(string s) {
    int i = 0, n = s.size(), sign = 1;
    long res = 0;
    while (i < n && s[i] == ' ') i++;
    if (i < n && (s[i] == '+' || s[i] == '-')) sign = s[i++] == '-' ? -1 : 1;
    while (i < n && isdigit(s[i])) {
        res = res * 10 + (s[i++] - '0');
        if (sign * res > INT_MAX) return INT_MAX;
        if (sign * res < INT_MIN) return INT_MIN;
    }
    return sign * res;
}
```

### Q41. Implement strStr() (KMP)
```cpp
int strStr(string h, string n) {
    if (n.empty()) return 0;
    int nn = n.size();
    vector<int> lps(nn, 0);
    for (int i = 1, len = 0; i < nn;) {
        if (n[i] == n[len]) lps[i++] = ++len;
        else if (len) len = lps[len-1];
        else lps[i++] = 0;
    }
    for (int i = 0, j = 0; i < h.size();) {
        if (h[i] == n[j]) { i++; j++; }
        if (j == nn) return i - j;
        else if (i < h.size() && h[i] != n[j]) {
            if (j) j = lps[j-1]; else i++;
        }
    }
    return -1;
}
```

### Q42. Count and Say
```cpp
string countAndSay(int n) {
    string s = "1";
    for (int i = 1; i < n; i++) {
        string t;
        for (int j = 0; j < s.size();) {
            int k = j;
            while (k < s.size() && s[k] == s[j]) k++;
            t += to_string(k - j) + s[j];
            j = k;
        }
        s = t;
    }
    return s;
}
```

### Q43. Longest Common Prefix
```cpp
string longestCommonPrefix(vector<string>& strs) {
    if (strs.empty()) return "";
    string p = strs[0];
    for (int i = 1; i < strs.size(); i++) {
        while (strs[i].find(p) != 0) p = p.substr(0, p.size() - 1);
        if (p.empty()) return "";
    }
    return p;
}
```

### Q44. Valid Parentheses
```cpp
bool isValid(string s) {
    stack<char> st;
    for (char c : s) {
        if (c == '(' || c == '[' || c == '{') st.push(c);
        else {
            if (st.empty()) return false;
            char t = st.top(); st.pop();
            if ((c == ')' && t != '(') || (c == ']' && t != '[') || (c == '}' && t != '{')) return false;
        }
    }
    return st.empty();
}
```

### Q45. Generate Parentheses
```cpp
void gen(int o, int c, int n, string s, vector<string>& res) {
    if (s.size() == 2 * n) { res.push_back(s); return; }
    if (o < n) gen(o + 1, c, n, s + '(', res);
    if (c < o) gen(o, c + 1, n, s + ')', res);
}
vector<string> generateParenthesis(int n) {
    vector<string> res; gen(0, 0, n, "", res); return res;
}
```

### Q46. Longest Repeating Character Replacement
```cpp
int characterReplacement(string s, int k) {
    int cnt[26] = {}, l = 0, maxF = 0, best = 0;
    for (int r = 0; r < s.size(); r++) {
        maxF = max(maxF, ++cnt[s[r]-'A']);
        while ((r - l + 1) - maxF > k) cnt[s[l++]-'A']--;
        best = max(best, r - l + 1);
    }
    return best;
}
```

### Q47. Is Subsequence
```cpp
bool isSubsequence(string s, string t) {
    int i = 0;
    for (char c : t) if (i < s.size() && c == s[i]) i++;
    return i == s.size();
}
```

### Q48. ZigZag Conversion
```cpp
string convert(string s, int r) {
    if (r == 1) return s;
    vector<string> rows(min((int)s.size(), r));
    int cur = 0, dir = -1;
    for (char c : s) {
        rows[cur] += c;
        if (cur == 0 || cur == r - 1) dir = -dir;
        cur += dir;
    }
    string res;
    for (auto& x : rows) res += x;
    return res;
}
```

### Q49. Roman to Integer
```cpp
int romanToInt(string s) {
    unordered_map<char,int> m{{'I',1},{'V',5},{'X',10},{'L',50},{'C',100},{'D',500},{'M',1000}};
    int res = 0;
    for (int i = 0; i < s.size(); i++) {
        if (i + 1 < s.size() && m[s[i]] < m[s[i+1]]) res -= m[s[i]];
        else res += m[s[i]];
    }
    return res;
}
```

### Q50. Integer to Roman
```cpp
string intToRoman(int n) {
    vector<pair<int,string>> v{{1000,"M"},{900,"CM"},{500,"D"},{400,"CD"},{100,"C"},{90,"XC"},{50,"L"},{40,"XL"},{10,"X"},{9,"IX"},{5,"V"},{4,"IV"},{1,"I"}};
    string r;
    for (auto& p : v) while (n >= p.first) { r += p.second; n -= p.first; }
    return r;
}
```

### Q51. Word Search (DFS)
```cpp
bool dfs(vector<vector<char>>& b, string& w, int i, int j, int k) {
    if (k == w.size()) return true;
    if (i<0||j<0||i>=b.size()||j>=b[0].size()||b[i][j]!=w[k]) return false;
    char t = b[i][j]; b[i][j] = '#';
    bool ok = dfs(b,w,i+1,j,k+1)||dfs(b,w,i-1,j,k+1)||dfs(b,w,i,j+1,k+1)||dfs(b,w,i,j-1,k+1);
    b[i][j] = t;
    return ok;
}
bool exist(vector<vector<char>>& b, string w) {
    for (int i = 0; i < b.size(); i++)
        for (int j = 0; j < b[0].size(); j++)
            if (dfs(b, w, i, j, 0)) return true;
    return false;
}
```

### Q52. Regular Expression Matching
```cpp
bool isMatch(string s, string p) {
    int m = s.size(), n = p.size();
    vector<vector<bool>> dp(m+1, vector<bool>(n+1, false));
    dp[0][0] = true;
    for (int j = 1; j <= n; j++) if (p[j-1] == '*') dp[0][j] = dp[0][j-2];
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++) {
            if (p[j-1] == '*') {
                dp[i][j] = dp[i][j-2] || ((p[j-2] == s[i-1] || p[j-2] == '.') && dp[i-1][j]);
            } else if (p[j-1] == '.' || p[j-1] == s[i-1]) dp[i][j] = dp[i-1][j-1];
        }
    return dp[m][n];
}
```

### Q53. Wildcard Matching
```cpp
bool isMatchW(string s, string p) {
    int m = s.size(), n = p.size();
    vector<vector<bool>> dp(m+1, vector<bool>(n+1, false));
    dp[0][0] = true;
    for (int j = 1; j <= n && p[j-1] == '*'; j++) dp[0][j] = true;
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++) {
            if (p[j-1] == '*') dp[i][j] = dp[i-1][j] || dp[i][j-1];
            else if (p[j-1] == '?' || p[j-1] == s[i-1]) dp[i][j] = dp[i-1][j-1];
        }
    return dp[m][n];
}
```

### Q54. Scramble String
```cpp
unordered_map<string,bool> memo;
bool isScramble(string s1, string s2) {
    string k = s1 + "#" + s2;
    if (memo.count(k)) return memo[k];
    if (s1 == s2) return memo[k] = true;
    int c[26] = {};
    for (char x : s1) c[x-'a']++;
    for (char x : s2) c[x-'a']--;
    for (int x : c) if (x) return memo[k] = false;
    int n = s1.size();
    for (int i = 1; i < n; i++) {
        if (isScramble(s1.substr(0,i), s2.substr(0,i)) && isScramble(s1.substr(i), s2.substr(i))) return memo[k] = true;
        if (isScramble(s1.substr(0,i), s2.substr(n-i)) && isScramble(s1.substr(i), s2.substr(0,n-i))) return memo[k] = true;
    }
    return memo[k] = false;
}
```

### Q55. Edit Distance
```cpp
int minDistance(string a, string b) {
    int m = a.size(), n = b.size();
    vector<vector<int>> dp(m+1, vector<int>(n+1));
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++) {
            if (a[i-1] == b[j-1]) dp[i][j] = dp[i-1][j-1];
            else dp[i][j] = 1 + min({dp[i-1][j], dp[i][j-1], dp[i-1][j-1]});
        }
    return dp[m][n];
}
```

---

## Linked List (Q56–Q75)

```cpp
struct ListNode { int val; ListNode* next; ListNode(int v): val(v), next(nullptr) {} };
```

### Q56. Reverse Linked List
```cpp
ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;
    while (head) { auto nx = head->next; head->next = prev; prev = head; head = nx; }
    return prev;
}
```

### Q57. Linked List Cycle
```cpp
bool hasCycle(ListNode* head) {
    auto s = head, f = head;
    while (f && f->next) { s = s->next; f = f->next->next; if (s == f) return true; }
    return false;
}
```

### Q58. Merge Two Sorted Lists
```cpp
ListNode* mergeTwoLists(ListNode* a, ListNode* b) {
    ListNode dummy(0), *t = &dummy;
    while (a && b) {
        if (a->val <= b->val) { t->next = a; a = a->next; }
        else { t->next = b; b = b->next; }
        t = t->next;
    }
    t->next = a ? a : b;
    return dummy.next;
}
```

### Q59. Remove Nth Node From End
```cpp
ListNode* removeNthFromEnd(ListNode* head, int n) {
    ListNode dummy(0); dummy.next = head;
    auto f = &dummy, s = &dummy;
    for (int i = 0; i <= n; i++) f = f->next;
    while (f) { f = f->next; s = s->next; }
    s->next = s->next->next;
    return dummy.next;
}
```

### Q60. Reorder List
```cpp
void reorderList(ListNode* h) {
    if (!h || !h->next) return;
    auto s = h, f = h;
    while (f->next && f->next->next) { s = s->next; f = f->next->next; }
    ListNode* prev = nullptr; auto cur = s->next; s->next = nullptr;
    while (cur) { auto nx = cur->next; cur->next = prev; prev = cur; cur = nx; }
    auto a = h, b = prev;
    while (b) { auto t1 = a->next, t2 = b->next; a->next = b; b->next = t1; a = t1; b = t2; }
}
```

### Q61. Add Two Numbers
```cpp
ListNode* addTwoNumbers(ListNode* a, ListNode* b) {
    ListNode dummy(0), *t = &dummy; int carry = 0;
    while (a || b || carry) {
        int s = carry + (a ? a->val : 0) + (b ? b->val : 0);
        carry = s / 10;
        t->next = new ListNode(s % 10); t = t->next;
        if (a) a = a->next;
        if (b) b = b->next;
    }
    return dummy.next;
}
```

### Q62. Copy List with Random Pointer
```cpp
struct Node { int val; Node *next, *random; };
Node* copyRandomList(Node* head) {
    unordered_map<Node*, Node*> mp;
    for (auto c = head; c; c = c->next) mp[c] = new Node{c->val, nullptr, nullptr};
    for (auto c = head; c; c = c->next) {
        mp[c]->next = mp[c->next];
        mp[c]->random = mp[c->random];
    }
    return mp[head];
}
```

### Q63. Intersection of Two Linked Lists
```cpp
ListNode* getIntersectionNode(ListNode* a, ListNode* b) {
    auto p1 = a, p2 = b;
    while (p1 != p2) {
        p1 = p1 ? p1->next : b;
        p2 = p2 ? p2->next : a;
    }
    return p1;
}
```

### Q64. Remove Duplicates from Sorted List
```cpp
ListNode* deleteDuplicates(ListNode* h) {
    for (auto c = h; c && c->next;)
        if (c->val == c->next->val) c->next = c->next->next;
        else c = c->next;
    return h;
}
```

### Q65. Palindrome Linked List
```cpp
bool isPalindrome(ListNode* h) {
    auto s = h, f = h;
    while (f && f->next) { s = s->next; f = f->next->next; }
    ListNode* prev = nullptr;
    while (s) { auto nx = s->next; s->next = prev; prev = s; s = nx; }
    while (prev) { if (prev->val != h->val) return false; prev = prev->next; h = h->next; }
    return true;
}
```

### Q66. Swap Nodes in Pairs
```cpp
ListNode* swapPairs(ListNode* h) {
    ListNode dummy(0); dummy.next = h;
    auto p = &dummy;
    while (p->next && p->next->next) {
        auto a = p->next, b = a->next;
        a->next = b->next; b->next = a; p->next = b;
        p = a;
    }
    return dummy.next;
}
```

### Q67. Flatten Multilevel Doubly Linked List
```cpp
struct DNode { int val; DNode *prev, *next, *child; };
DNode* flatten(DNode* h) {
    stack<DNode*> st;
    if (h) st.push(h);
    DNode* prev = nullptr;
    while (!st.empty()) {
        auto c = st.top(); st.pop();
        if (prev) { prev->next = c; c->prev = prev; }
        if (c->next) st.push(c->next);
        if (c->child) { st.push(c->child); c->child = nullptr; }
        prev = c;
    }
    return h;
}
```

### Q68. LRU Cache
```cpp
class LRUCache {
    int cap;
    list<pair<int,int>> lst;
    unordered_map<int, list<pair<int,int>>::iterator> mp;
public:
    LRUCache(int c) : cap(c) {}
    int get(int k) {
        if (!mp.count(k)) return -1;
        lst.splice(lst.begin(), lst, mp[k]);
        return mp[k]->second;
    }
    void put(int k, int v) {
        if (mp.count(k)) { mp[k]->second = v; lst.splice(lst.begin(), lst, mp[k]); return; }
        if (lst.size() == cap) { mp.erase(lst.back().first); lst.pop_back(); }
        lst.push_front({k, v}); mp[k] = lst.begin();
    }
};
```

### Q69. Sort List (Merge Sort)
```cpp
ListNode* sortList(ListNode* h) {
    if (!h || !h->next) return h;
    auto s = h, f = h->next;
    while (f && f->next) { s = s->next; f = f->next->next; }
    auto mid = s->next; s->next = nullptr;
    return mergeTwoLists(sortList(h), sortList(mid));
}
```

### Q70. Rotate List
```cpp
ListNode* rotateRight(ListNode* h, int k) {
    if (!h) return h;
    int len = 1; auto t = h;
    while (t->next) { t = t->next; len++; }
    t->next = h;
    k = len - k % len;
    while (k--) t = t->next;
    h = t->next; t->next = nullptr;
    return h;
}
```

### Q71. Partition List
```cpp
ListNode* partition(ListNode* h, int x) {
    ListNode a(0), b(0), *ta = &a, *tb = &b;
    while (h) {
        if (h->val < x) { ta->next = h; ta = ta->next; }
        else { tb->next = h; tb = tb->next; }
        h = h->next;
    }
    tb->next = nullptr; ta->next = b.next;
    return a.next;
}
```

### Q72. Reverse Nodes in k-Group
```cpp
ListNode* reverseKGroup(ListNode* h, int k) {
    auto c = h; int cnt = 0;
    while (c && cnt < k) { c = c->next; cnt++; }
    if (cnt < k) return h;
    ListNode* prev = reverseKGroup(c, k);
    while (cnt--) { auto nx = h->next; h->next = prev; prev = h; h = nx; }
    return prev;
}
```

### Q73. Merge K Sorted Lists
```cpp
ListNode* mergeKLists(vector<ListNode*>& v) {
    auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);
    for (auto x : v) if (x) pq.push(x);
    ListNode dummy(0), *t = &dummy;
    while (!pq.empty()) {
        auto n = pq.top(); pq.pop();
        t->next = n; t = t->next;
        if (n->next) pq.push(n->next);
    }
    return dummy.next;
}
```

### Q74. Middle of the Linked List
```cpp
ListNode* middleNode(ListNode* h) {
    auto s = h, f = h;
    while (f && f->next) { s = s->next; f = f->next->next; }
    return s;
}
```

### Q75. Odd Even Linked List
```cpp
ListNode* oddEvenList(ListNode* h) {
    if (!h) return h;
    auto o = h, e = h->next, eh = e;
    while (e && e->next) {
        o->next = e->next; o = o->next;
        e->next = o->next; e = e->next;
    }
    o->next = eh;
    return h;
}
```

---

## Stack & Queue (Q76–Q90)

### Q76. Min Stack
```cpp
class MinStack {
    stack<int> s, mn;
public:
    void push(int v) { s.push(v); if (mn.empty() || v <= mn.top()) mn.push(v); }
    void pop() { if (s.top() == mn.top()) mn.pop(); s.pop(); }
    int top() { return s.top(); }
    int getMin() { return mn.top(); }
};
```

### Q77. Evaluate Reverse Polish Notation
```cpp
int evalRPN(vector<string>& v) {
    stack<int> s;
    for (auto& t : v) {
        if (t == "+" || t == "-" || t == "*" || t == "/") {
            int b = s.top(); s.pop(); int a = s.top(); s.pop();
            if (t == "+") s.push(a + b);
            else if (t == "-") s.push(a - b);
            else if (t == "*") s.push(a * b);
            else s.push(a / b);
        } else s.push(stoi(t));
    }
    return s.top();
}
```

### Q78. Daily Temperatures
```cpp
vector<int> dailyTemperatures(vector<int>& t) {
    vector<int> res(t.size(), 0);
    stack<int> st;
    for (int i = 0; i < t.size(); i++) {
        while (!st.empty() && t[i] > t[st.top()]) {
            res[st.top()] = i - st.top(); st.pop();
        }
        st.push(i);
    }
    return res;
}
```

### Q79. Next Greater Element I
```cpp
vector<int> nextGreaterElement(vector<int>& a, vector<int>& b) {
    unordered_map<int,int> mp;
    stack<int> st;
    for (int x : b) {
        while (!st.empty() && st.top() < x) { mp[st.top()] = x; st.pop(); }
        st.push(x);
    }
    vector<int> r;
    for (int x : a) r.push_back(mp.count(x) ? mp[x] : -1);
    return r;
}
```

### Q80. Next Greater Element II (circular)
```cpp
vector<int> nextGreaterElements(vector<int>& nums) {
    int n = nums.size();
    vector<int> res(n, -1);
    stack<int> st;
    for (int i = 0; i < 2 * n; i++) {
        while (!st.empty() && nums[st.top()] < nums[i % n]) {
            res[st.top()] = nums[i % n]; st.pop();
        }
        if (i < n) st.push(i);
    }
    return res;
}
```

### Q81. Largest Rectangle in Histogram
```cpp
int largestRectangleArea(vector<int>& h) {
    h.push_back(0);
    stack<int> st;
    int best = 0;
    for (int i = 0; i < h.size(); i++) {
        while (!st.empty() && h[st.top()] > h[i]) {
            int top = st.top(); st.pop();
            int w = st.empty() ? i : i - st.top() - 1;
            best = max(best, h[top] * w);
        }
        st.push(i);
    }
    return best;
}
```

### Q82. Maximal Rectangle
```cpp
int maximalRectangle(vector<vector<char>>& m) {
    if (m.empty()) return 0;
    vector<int> h(m[0].size(), 0);
    int best = 0;
    for (auto& row : m) {
        for (int j = 0; j < row.size(); j++)
            h[j] = row[j] == '1' ? h[j] + 1 : 0;
        best = max(best, largestRectangleArea(h));
    }
    return best;
}
```

### Q83. Sliding Window Maximum
```cpp
vector<int> maxSlidingWindow(vector<int>& nums, int k) {
    deque<int> dq;
    vector<int> res;
    for (int i = 0; i < nums.size(); i++) {
        while (!dq.empty() && dq.front() <= i - k) dq.pop_front();
        while (!dq.empty() && nums[dq.back()] < nums[i]) dq.pop_back();
        dq.push_back(i);
        if (i >= k - 1) res.push_back(nums[dq.front()]);
    }
    return res;
}
```

### Q84. Valid Parentheses
*(See Q44)*

### Q85. Decode String
```cpp
string decodeString(string s) {
    stack<int> ns; stack<string> ss;
    string cur; int k = 0;
    for (char c : s) {
        if (isdigit(c)) k = k * 10 + (c - '0');
        else if (c == '[') { ns.push(k); ss.push(cur); k = 0; cur = ""; }
        else if (c == ']') {
            string t = cur; cur = ss.top(); ss.pop();
            int rep = ns.top(); ns.pop();
            while (rep--) cur += t;
        } else cur += c;
    }
    return cur;
}
```

### Q86. Implement Queue using Stacks
```cpp
class MyQueue {
    stack<int> in, out;
public:
    void push(int x) { in.push(x); }
    int pop() { peek(); int v = out.top(); out.pop(); return v; }
    int peek() {
        if (out.empty()) while (!in.empty()) { out.push(in.top()); in.pop(); }
        return out.top();
    }
    bool empty() { return in.empty() && out.empty(); }
};
```

### Q87. Implement Stack using Queues
```cpp
class MyStack {
    queue<int> q;
public:
    void push(int x) { q.push(x); for (int i = 1; i < q.size(); i++) { q.push(q.front()); q.pop(); } }
    int pop() { int v = q.front(); q.pop(); return v; }
    int top() { return q.front(); }
    bool empty() { return q.empty(); }
};
```

### Q88. Number of Visible People in a Queue
```cpp
vector<int> canSeePersonsCount(vector<int>& h) {
    int n = h.size();
    vector<int> res(n, 0);
    stack<int> st;
    for (int i = n - 1; i >= 0; i--) {
        while (!st.empty() && st.top() < h[i]) { res[i]++; st.pop(); }
        if (!st.empty()) res[i]++;
        st.push(h[i]);
    }
    return res;
}
```

### Q89. Remove K Digits
```cpp
string removeKdigits(string num, int k) {
    string s;
    for (char c : num) {
        while (k && !s.empty() && s.back() > c) { s.pop_back(); k--; }
        s += c;
    }
    while (k--) s.pop_back();
    int i = 0;
    while (i < s.size() && s[i] == '0') i++;
    s = s.substr(i);
    return s.empty() ? "0" : s;
}
```

### Q90. Asteroid Collision
```cpp
vector<int> asteroidCollision(vector<int>& a) {
    vector<int> st;
    for (int x : a) {
        bool alive = true;
        while (alive && x < 0 && !st.empty() && st.back() > 0) {
            if (st.back() < -x) st.pop_back();
            else if (st.back() == -x) { st.pop_back(); alive = false; }
            else alive = false;
        }
        if (alive) st.push_back(x);
    }
    return st;
}
```

---

## Binary Tree / BST (Q91–Q115)

```cpp
struct TreeNode { int val; TreeNode *left, *right; TreeNode(int v): val(v), left(nullptr), right(nullptr) {} };
```

### Q91. Maximum Depth of Binary Tree
```cpp
int maxDepth(TreeNode* r) {
    return !r ? 0 : 1 + max(maxDepth(r->left), maxDepth(r->right));
}
```

### Q92. Same Tree
```cpp
bool isSameTree(TreeNode* a, TreeNode* b) {
    if (!a && !b) return true;
    if (!a || !b || a->val != b->val) return false;
    return isSameTree(a->left, b->left) && isSameTree(a->right, b->right);
}
```

### Q93. Invert Binary Tree
```cpp
TreeNode* invertTree(TreeNode* r) {
    if (!r) return r;
    swap(r->left, r->right);
    invertTree(r->left); invertTree(r->right);
    return r;
}
```

### Q94. Level Order Traversal
```cpp
vector<vector<int>> levelOrder(TreeNode* r) {
    vector<vector<int>> res;
    if (!r) return res;
    queue<TreeNode*> q; q.push(r);
    while (!q.empty()) {
        int sz = q.size();
        vector<int> lvl;
        while (sz--) {
            auto n = q.front(); q.pop();
            lvl.push_back(n->val);
            if (n->left) q.push(n->left);
            if (n->right) q.push(n->right);
        }
        res.push_back(lvl);
    }
    return res;
}
```

### Q95. Zigzag Level Order
```cpp
vector<vector<int>> zigzagLevelOrder(TreeNode* r) {
    vector<vector<int>> res;
    if (!r) return res;
    queue<TreeNode*> q; q.push(r); bool ltr = true;
    while (!q.empty()) {
        int sz = q.size();
        vector<int> lvl(sz);
        for (int i = 0; i < sz; i++) {
            auto n = q.front(); q.pop();
            lvl[ltr ? i : sz - 1 - i] = n->val;
            if (n->left) q.push(n->left);
            if (n->right) q.push(n->right);
        }
        res.push_back(lvl); ltr = !ltr;
    }
    return res;
}
```

### Q96. Maximum Width of Binary Tree
```cpp
int widthOfBinaryTree(TreeNode* r) {
    if (!r) return 0;
    queue<pair<TreeNode*, unsigned long long>> q;
    q.push({r, 0});
    unsigned long long best = 0;
    while (!q.empty()) {
        int sz = q.size();
        auto lo = q.front().second, hi = q.back().second;
        best = max(best, hi - lo + 1);
        while (sz--) {
            auto [n, i] = q.front(); q.pop();
            if (n->left) q.push({n->left, 2 * i});
            if (n->right) q.push({n->right, 2 * i + 1});
        }
    }
    return best;
}
```

### Q97. Binary Tree Right Side View
```cpp
vector<int> rightSideView(TreeNode* r) {
    vector<int> res;
    if (!r) return res;
    queue<TreeNode*> q; q.push(r);
    while (!q.empty()) {
        int sz = q.size();
        for (int i = 0; i < sz; i++) {
            auto n = q.front(); q.pop();
            if (i == sz - 1) res.push_back(n->val);
            if (n->left) q.push(n->left);
            if (n->right) q.push(n->right);
        }
    }
    return res;
}
```

### Q98. Count Complete Tree Nodes
```cpp
int countNodes(TreeNode* r) {
    if (!r) return 0;
    int l = 0, rr = 0;
    for (auto x = r; x; x = x->left) l++;
    for (auto x = r; x; x = x->right) rr++;
    if (l == rr) return (1 << l) - 1;
    return 1 + countNodes(r->left) + countNodes(r->right);
}
```

### Q99. Path Sum
```cpp
bool hasPathSum(TreeNode* r, int s) {
    if (!r) return false;
    if (!r->left && !r->right) return s == r->val;
    return hasPathSum(r->left, s - r->val) || hasPathSum(r->right, s - r->val);
}
```

### Q100. Binary Tree Maximum Path Sum
```cpp
int best;
int dfs(TreeNode* r) {
    if (!r) return 0;
    int l = max(0, dfs(r->left)), rr = max(0, dfs(r->right));
    best = max(best, l + rr + r->val);
    return r->val + max(l, rr);
}
int maxPathSum(TreeNode* r) { best = INT_MIN; dfs(r); return best; }
```

### Q101. Lowest Common Ancestor of Binary Tree
```cpp
TreeNode* lca(TreeNode* r, TreeNode* p, TreeNode* q) {
    if (!r || r == p || r == q) return r;
    auto l = lca(r->left, p, q), rr = lca(r->right, p, q);
    return (l && rr) ? r : (l ? l : rr);
}
```

### Q102. Flatten Binary Tree to Linked List
```cpp
void flatten(TreeNode* r) {
    if (!r) return;
    flatten(r->left); flatten(r->right);
    auto rr = r->right;
    r->right = r->left; r->left = nullptr;
    auto c = r;
    while (c->right) c = c->right;
    c->right = rr;
}
```

### Q103. Build Tree from Preorder + Inorder
```cpp
unordered_map<int,int> inMap;
int preIdx = 0;
TreeNode* build(vector<int>& pre, int l, int r) {
    if (l > r) return nullptr;
    int v = pre[preIdx++];
    auto n = new TreeNode(v);
    int i = inMap[v];
    n->left = build(pre, l, i - 1);
    n->right = build(pre, i + 1, r);
    return n;
}
TreeNode* buildTree(vector<int>& pre, vector<int>& in) {
    for (int i = 0; i < in.size(); i++) inMap[in[i]] = i;
    preIdx = 0;
    return build(pre, 0, in.size() - 1);
}
```

### Q104. Build Tree from Inorder + Postorder
```cpp
int postIdx;
unordered_map<int,int> inIdx;
TreeNode* b2(vector<int>& post, int l, int r) {
    if (l > r) return nullptr;
    int v = post[postIdx--];
    auto n = new TreeNode(v);
    int i = inIdx[v];
    n->right = b2(post, i + 1, r);
    n->left = b2(post, l, i - 1);
    return n;
}
TreeNode* buildTree2(vector<int>& in, vector<int>& post) {
    for (int i = 0; i < in.size(); i++) inIdx[in[i]] = i;
    postIdx = post.size() - 1;
    return b2(post, 0, in.size() - 1);
}
```

### Q105. Serialize and Deserialize Binary Tree
```cpp
string serialize(TreeNode* r) {
    if (!r) return "#";
    return to_string(r->val) + "," + serialize(r->left) + "," + serialize(r->right);
}
TreeNode* deser(queue<string>& q) {
    string s = q.front(); q.pop();
    if (s == "#") return nullptr;
    auto n = new TreeNode(stoi(s));
    n->left = deser(q); n->right = deser(q);
    return n;
}
TreeNode* deserialize(string s) {
    queue<string> q; string t;
    for (char c : s) {
        if (c == ',') { q.push(t); t = ""; }
        else t += c;
    }
    q.push(t);
    return deser(q);
}
```

### Q106. Diameter of Binary Tree
```cpp
int diam;
int dDfs(TreeNode* r) {
    if (!r) return 0;
    int l = dDfs(r->left), rr = dDfs(r->right);
    diam = max(diam, l + rr);
    return 1 + max(l, rr);
}
int diameterOfBinaryTree(TreeNode* r) { diam = 0; dDfs(r); return diam; }
```

### Q107. Balanced Binary Tree
```cpp
int chk(TreeNode* r) {
    if (!r) return 0;
    int l = chk(r->left); if (l < 0) return -1;
    int rr = chk(r->right); if (rr < 0) return -1;
    if (abs(l - rr) > 1) return -1;
    return 1 + max(l, rr);
}
bool isBalanced(TreeNode* r) { return chk(r) >= 0; }
```

### Q108. Symmetric Tree
```cpp
bool mir(TreeNode* a, TreeNode* b) {
    if (!a && !b) return true;
    if (!a || !b || a->val != b->val) return false;
    return mir(a->left, b->right) && mir(a->right, b->left);
}
bool isSymmetric(TreeNode* r) { return !r || mir(r->left, r->right); }
```

### Q109. Sum Root to Leaf Numbers
```cpp
int sumDfs(TreeNode* r, int s) {
    if (!r) return 0;
    s = s * 10 + r->val;
    if (!r->left && !r->right) return s;
    return sumDfs(r->left, s) + sumDfs(r->right, s);
}
int sumNumbers(TreeNode* r) { return sumDfs(r, 0); }
```

### Q110. Populating Next Right Pointers
```cpp
struct PNode { int val; PNode *left, *right, *next; };
PNode* connect(PNode* r) {
    auto lvl = r;
    while (lvl && lvl->left) {
        auto c = lvl;
        while (c) {
            c->left->next = c->right;
            if (c->next) c->right->next = c->next->left;
            c = c->next;
        }
        lvl = lvl->left;
    }
    return r;
}
```

### Q111. Validate BST
```cpp
bool valid(TreeNode* r, long mn, long mx) {
    if (!r) return true;
    if (r->val <= mn || r->val >= mx) return false;
    return valid(r->left, mn, r->val) && valid(r->right, r->val, mx);
}
bool isValidBST(TreeNode* r) { return valid(r, LONG_MIN, LONG_MAX); }
```

### Q112. Kth Smallest Element in BST
```cpp
int kthSmallest(TreeNode* r, int k) {
    stack<TreeNode*> st;
    while (r || !st.empty()) {
        while (r) { st.push(r); r = r->left; }
        r = st.top(); st.pop();
        if (--k == 0) return r->val;
        r = r->right;
    }
    return -1;
}
```

### Q113. Delete Node in BST
```cpp
TreeNode* deleteNode(TreeNode* r, int k) {
    if (!r) return r;
    if (k < r->val) r->left = deleteNode(r->left, k);
    else if (k > r->val) r->right = deleteNode(r->right, k);
    else {
        if (!r->left) return r->right;
        if (!r->right) return r->left;
        auto t = r->right;
        while (t->left) t = t->left;
        r->val = t->val;
        r->right = deleteNode(r->right, t->val);
    }
    return r;
}
```

### Q114. Insert into BST
```cpp
TreeNode* insertIntoBST(TreeNode* r, int v) {
    if (!r) return new TreeNode(v);
    if (v < r->val) r->left = insertIntoBST(r->left, v);
    else r->right = insertIntoBST(r->right, v);
    return r;
}
```

### Q115. Recover BST
```cpp
TreeNode *first = nullptr, *second = nullptr, *prev = nullptr;
void inorder(TreeNode* r) {
    if (!r) return;
    inorder(r->left);
    if (prev && prev->val > r->val) {
        if (!first) first = prev;
        second = r;
    }
    prev = r;
    inorder(r->right);
}
void recoverTree(TreeNode* r) {
    inorder(r);
    swap(first->val, second->val);
}
```

---

## Graphs (Q116–Q135)

### Q116. Number of Islands
```cpp
void dfsIsland(vector<vector<char>>& g, int i, int j) {
    if (i<0||j<0||i>=g.size()||j>=g[0].size()||g[i][j]!='1') return;
    g[i][j] = '0';
    dfsIsland(g,i+1,j); dfsIsland(g,i-1,j); dfsIsland(g,i,j+1); dfsIsland(g,i,j-1);
}
int numIslands(vector<vector<char>>& g) {
    int cnt = 0;
    for (int i = 0; i < g.size(); i++)
        for (int j = 0; j < g[0].size(); j++)
            if (g[i][j] == '1') { dfsIsland(g, i, j); cnt++; }
    return cnt;
}
```

### Q117. Clone Graph
```cpp
struct GNode { int val; vector<GNode*> neighbors; };
unordered_map<GNode*, GNode*> mp;
GNode* cloneGraph(GNode* n) {
    if (!n) return n;
    if (mp.count(n)) return mp[n];
    auto c = new GNode{n->val, {}};
    mp[n] = c;
    for (auto nb : n->neighbors) c->neighbors.push_back(cloneGraph(nb));
    return c;
}
```

### Q118. Pacific Atlantic Water Flow
```cpp
void paDfs(vector<vector<int>>& h, vector<vector<bool>>& vis, int i, int j, int prev) {
    if (i<0||j<0||i>=h.size()||j>=h[0].size()||vis[i][j]||h[i][j]<prev) return;
    vis[i][j] = true;
    paDfs(h, vis, i+1, j, h[i][j]); paDfs(h, vis, i-1, j, h[i][j]);
    paDfs(h, vis, i, j+1, h[i][j]); paDfs(h, vis, i, j-1, h[i][j]);
}
vector<vector<int>> pacificAtlantic(vector<vector<int>>& h) {
    int m = h.size(), n = h[0].size();
    vector<vector<bool>> p(m, vector<bool>(n)), a(m, vector<bool>(n));
    for (int i = 0; i < m; i++) { paDfs(h,p,i,0,0); paDfs(h,a,i,n-1,0); }
    for (int j = 0; j < n; j++) { paDfs(h,p,0,j,0); paDfs(h,a,m-1,j,0); }
    vector<vector<int>> res;
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++)
            if (p[i][j] && a[i][j]) res.push_back({i,j});
    return res;
}
```

### Q119. Course Schedule
```cpp
bool canFinish(int n, vector<vector<int>>& pre) {
    vector<vector<int>> g(n);
    vector<int> in(n, 0);
    for (auto& p : pre) { g[p[1]].push_back(p[0]); in[p[0]]++; }
    queue<int> q;
    for (int i = 0; i < n; i++) if (!in[i]) q.push(i);
    int done = 0;
    while (!q.empty()) {
        int c = q.front(); q.pop(); done++;
        for (int nx : g[c]) if (--in[nx] == 0) q.push(nx);
    }
    return done == n;
}
```

### Q120. Course Schedule II
```cpp
vector<int> findOrder(int n, vector<vector<int>>& pre) {
    vector<vector<int>> g(n);
    vector<int> in(n, 0), res;
    for (auto& p : pre) { g[p[1]].push_back(p[0]); in[p[0]]++; }
    queue<int> q;
    for (int i = 0; i < n; i++) if (!in[i]) q.push(i);
    while (!q.empty()) {
        int c = q.front(); q.pop(); res.push_back(c);
        for (int nx : g[c]) if (--in[nx] == 0) q.push(nx);
    }
    return res.size() == n ? res : vector<int>{};
}
```

### Q121. Number of Connected Components (Union-Find)
```cpp
int findP(vector<int>& p, int x) { return p[x] == x ? x : p[x] = findP(p, p[x]); }
int countComponents(int n, vector<vector<int>>& edges) {
    vector<int> p(n); iota(p.begin(), p.end(), 0);
    int c = n;
    for (auto& e : edges) {
        int a = findP(p, e[0]), b = findP(p, e[1]);
        if (a != b) { p[a] = b; c--; }
    }
    return c;
}
```

### Q122. Redundant Connection
```cpp
vector<int> findRedundantConnection(vector<vector<int>>& edges) {
    int n = edges.size();
    vector<int> p(n + 1); iota(p.begin(), p.end(), 0);
    for (auto& e : edges) {
        int a = findP(p, e[0]), b = findP(p, e[1]);
        if (a == b) return e;
        p[a] = b;
    }
    return {};
}
```

### Q123. Word Ladder
```cpp
int ladderLength(string b, string e, vector<string>& wl) {
    unordered_set<string> dict(wl.begin(), wl.end());
    if (!dict.count(e)) return 0;
    queue<string> q; q.push(b);
    int steps = 1;
    while (!q.empty()) {
        int sz = q.size();
        while (sz--) {
            string w = q.front(); q.pop();
            if (w == e) return steps;
            for (int i = 0; i < w.size(); i++) {
                char orig = w[i];
                for (char c = 'a'; c <= 'z'; c++) {
                    w[i] = c;
                    if (dict.count(w)) { q.push(w); dict.erase(w); }
                }
                w[i] = orig;
            }
        }
        steps++;
    }
    return 0;
}
```

### Q124. Alien Dictionary
```cpp
string alienOrder(vector<string>& words) {
    unordered_map<char, unordered_set<char>> g;
    unordered_map<char, int> in;
    for (auto& w : words) for (char c : w) in[c] = 0;
    for (int i = 0; i + 1 < words.size(); i++) {
        auto& a = words[i]; auto& b = words[i+1];
        if (a.size() > b.size() && a.substr(0, b.size()) == b) return "";
        for (int j = 0; j < min(a.size(), b.size()); j++) {
            if (a[j] != b[j]) {
                if (!g[a[j]].count(b[j])) { g[a[j]].insert(b[j]); in[b[j]]++; }
                break;
            }
        }
    }
    queue<char> q;
    for (auto& [c, d] : in) if (!d) q.push(c);
    string res;
    while (!q.empty()) {
        char c = q.front(); q.pop(); res += c;
        for (char nx : g[c]) if (--in[nx] == 0) q.push(nx);
    }
    return res.size() == in.size() ? res : "";
}
```

### Q125. Graph Valid Tree
```cpp
bool validTree(int n, vector<vector<int>>& edges) {
    if (edges.size() != n - 1) return false;
    vector<int> p(n); iota(p.begin(), p.end(), 0);
    for (auto& e : edges) {
        int a = findP(p, e[0]), b = findP(p, e[1]);
        if (a == b) return false;
        p[a] = b;
    }
    return true;
}
```

### Q126. Minimum Spanning Tree (Kruskal)
```cpp
int kruskal(int n, vector<vector<int>>& edges) {
    sort(edges.begin(), edges.end(), [](auto& a, auto& b){ return a[2] < b[2]; });
    vector<int> p(n); iota(p.begin(), p.end(), 0);
    int cost = 0, used = 0;
    for (auto& e : edges) {
        int a = findP(p, e[0]), b = findP(p, e[1]);
        if (a != b) { p[a] = b; cost += e[2]; if (++used == n - 1) break; }
    }
    return cost;
}
```

### Q127. Cheapest Flights Within K Stops (Bellman-Ford)
```cpp
int findCheapestPrice(int n, vector<vector<int>>& flights, int src, int dst, int k) {
    vector<int> dist(n, INT_MAX); dist[src] = 0;
    for (int i = 0; i <= k; i++) {
        vector<int> tmp = dist;
        for (auto& f : flights)
            if (dist[f[0]] != INT_MAX)
                tmp[f[1]] = min(tmp[f[1]], dist[f[0]] + f[2]);
        dist = tmp;
    }
    return dist[dst] == INT_MAX ? -1 : dist[dst];
}
```

### Q128. Network Delay Time (Dijkstra)
```cpp
int networkDelayTime(vector<vector<int>>& times, int n, int k) {
    vector<vector<pair<int,int>>> g(n + 1);
    for (auto& t : times) g[t[0]].push_back({t[1], t[2]});
    vector<int> dist(n + 1, INT_MAX); dist[k] = 0;
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
    pq.push({0, k});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;
        for (auto& [v, w] : g[u])
            if (d + w < dist[v]) { dist[v] = d + w; pq.push({dist[v], v}); }
    }
    int ans = 0;
    for (int i = 1; i <= n; i++) {
        if (dist[i] == INT_MAX) return -1;
        ans = max(ans, dist[i]);
    }
    return ans;
}
```

### Q129. Surrounded Regions
```cpp
void srDfs(vector<vector<char>>& b, int i, int j) {
    if (i<0||j<0||i>=b.size()||j>=b[0].size()||b[i][j]!='O') return;
    b[i][j] = '#';
    srDfs(b,i+1,j); srDfs(b,i-1,j); srDfs(b,i,j+1); srDfs(b,i,j-1);
}
void solve(vector<vector<char>>& b) {
    int m = b.size(), n = b[0].size();
    for (int i = 0; i < m; i++) { srDfs(b,i,0); srDfs(b,i,n-1); }
    for (int j = 0; j < n; j++) { srDfs(b,0,j); srDfs(b,m-1,j); }
    for (auto& row : b) for (auto& c : row)
        c = (c == '#') ? 'O' : (c == 'O') ? 'X' : c;
}
```

### Q130. Rotting Oranges
```cpp
int orangesRotting(vector<vector<int>>& g) {
    int m = g.size(), n = g[0].size(), fresh = 0, t = 0;
    queue<pair<int,int>> q;
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++) {
            if (g[i][j] == 2) q.push({i,j});
            if (g[i][j] == 1) fresh++;
        }
    vector<pair<int,int>> d{{1,0},{-1,0},{0,1},{0,-1}};
    while (!q.empty() && fresh > 0) {
        int sz = q.size();
        while (sz--) {
            auto [x, y] = q.front(); q.pop();
            for (auto& [dx, dy] : d) {
                int nx = x + dx, ny = y + dy;
                if (nx<0||ny<0||nx>=m||ny>=n||g[nx][ny]!=1) continue;
                g[nx][ny] = 2; fresh--; q.push({nx, ny});
            }
        }
        t++;
    }
    return fresh ? -1 : t;
}
```

### Q131. Walls and Gates
```cpp
void wallsAndGates(vector<vector<int>>& r) {
    int m = r.size(), n = r[0].size();
    queue<pair<int,int>> q;
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++)
            if (r[i][j] == 0) q.push({i,j});
    vector<pair<int,int>> d{{1,0},{-1,0},{0,1},{0,-1}};
    while (!q.empty()) {
        auto [x, y] = q.front(); q.pop();
        for (auto& [dx, dy] : d) {
            int nx = x+dx, ny = y+dy;
            if (nx<0||ny<0||nx>=m||ny>=n||r[nx][ny]!=INT_MAX) continue;
            r[nx][ny] = r[x][y] + 1; q.push({nx, ny});
        }
    }
}
```

### Q132. Find the Town Judge
```cpp
int findJudge(int n, vector<vector<int>>& trust) {
    vector<int> cnt(n + 1, 0);
    for (auto& t : trust) { cnt[t[0]]--; cnt[t[1]]++; }
    for (int i = 1; i <= n; i++) if (cnt[i] == n - 1) return i;
    return -1;
}
```

### Q133. All Paths From Source to Target
```cpp
vector<vector<int>> ap; vector<int> path;
void apDfs(vector<vector<int>>& g, int u) {
    path.push_back(u);
    if (u == g.size() - 1) ap.push_back(path);
    else for (int v : g[u]) apDfs(g, v);
    path.pop_back();
}
vector<vector<int>> allPathsSourceTarget(vector<vector<int>>& g) {
    ap.clear(); path.clear();
    apDfs(g, 0);
    return ap;
}
```

### Q134. Is Graph Bipartite?
```cpp
bool isBipartite(vector<vector<int>>& g) {
    int n = g.size();
    vector<int> col(n, 0);
    for (int s = 0; s < n; s++) {
        if (col[s]) continue;
        queue<int> q; q.push(s); col[s] = 1;
        while (!q.empty()) {
            int u = q.front(); q.pop();
            for (int v : g[u]) {
                if (!col[v]) { col[v] = -col[u]; q.push(v); }
                else if (col[v] == col[u]) return false;
            }
        }
    }
    return true;
}
```

### Q135. Topological Sort (Kahn)
```cpp
vector<int> topoSort(int n, vector<vector<int>>& edges) {
    vector<vector<int>> g(n);
    vector<int> in(n, 0), res;
    for (auto& e : edges) { g[e[0]].push_back(e[1]); in[e[1]]++; }
    queue<int> q;
    for (int i = 0; i < n; i++) if (!in[i]) q.push(i);
    while (!q.empty()) {
        int u = q.front(); q.pop(); res.push_back(u);
        for (int v : g[u]) if (--in[v] == 0) q.push(v);
    }
    return res.size() == n ? res : vector<int>{};
}
```

---

## Dynamic Programming (Q136–Q165)

### Q136. Climbing Stairs
```cpp
int climbStairs(int n) {
    int a = 1, b = 1;
    for (int i = 2; i <= n; i++) { int c = a + b; a = b; b = c; }
    return b;
}
```

### Q137. House Robber
```cpp
int rob(vector<int>& nums) {
    int prev = 0, cur = 0;
    for (int x : nums) { int t = max(cur, prev + x); prev = cur; cur = t; }
    return cur;
}
```

### Q138. House Robber II (circular)
```cpp
int robLine(vector<int>& nums, int l, int r) {
    int prev = 0, cur = 0;
    for (int i = l; i <= r; i++) { int t = max(cur, prev + nums[i]); prev = cur; cur = t; }
    return cur;
}
int rob2(vector<int>& nums) {
    if (nums.size() == 1) return nums[0];
    return max(robLine(nums, 0, nums.size()-2), robLine(nums, 1, nums.size()-1));
}
```

### Q139. Coin Change (min coins)
```cpp
int coinChange(vector<int>& coins, int amt) {
    vector<int> dp(amt + 1, amt + 1); dp[0] = 0;
    for (int i = 1; i <= amt; i++)
        for (int c : coins) if (c <= i) dp[i] = min(dp[i], dp[i-c] + 1);
    return dp[amt] > amt ? -1 : dp[amt];
}
```

### Q140. Coin Change II (count ways)
```cpp
int change(int amt, vector<int>& coins) {
    vector<int> dp(amt + 1, 0); dp[0] = 1;
    for (int c : coins)
        for (int i = c; i <= amt; i++) dp[i] += dp[i-c];
    return dp[amt];
}
```

### Q141. Longest Increasing Subsequence (O(n log n))
```cpp
int lengthOfLIS(vector<int>& nums) {
    vector<int> t;
    for (int x : nums) {
        auto it = lower_bound(t.begin(), t.end(), x);
        if (it == t.end()) t.push_back(x);
        else *it = x;
    }
    return t.size();
}
```

### Q142. Longest Common Subsequence
```cpp
int longestCommonSubsequence(string a, string b) {
    int m = a.size(), n = b.size();
    vector<vector<int>> dp(m+1, vector<int>(n+1, 0));
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            dp[i][j] = a[i-1] == b[j-1] ? dp[i-1][j-1] + 1 : max(dp[i-1][j], dp[i][j-1]);
    return dp[m][n];
}
```

### Q143. Word Break
```cpp
bool wordBreak(string s, vector<string>& wd) {
    unordered_set<string> d(wd.begin(), wd.end());
    int n = s.size();
    vector<bool> dp(n + 1, false); dp[0] = true;
    for (int i = 1; i <= n; i++)
        for (int j = 0; j < i; j++)
            if (dp[j] && d.count(s.substr(j, i - j))) { dp[i] = true; break; }
    return dp[n];
}
```

### Q144. Combination Sum IV
```cpp
int combinationSum4(vector<int>& nums, int t) {
    vector<unsigned> dp(t + 1, 0); dp[0] = 1;
    for (int i = 1; i <= t; i++)
        for (int x : nums) if (x <= i) dp[i] += dp[i-x];
    return dp[t];
}
```

### Q145. Partition Equal Subset Sum
```cpp
bool canPartition(vector<int>& nums) {
    int sum = accumulate(nums.begin(), nums.end(), 0);
    if (sum % 2) return false;
    int t = sum / 2;
    vector<bool> dp(t + 1, false); dp[0] = true;
    for (int x : nums)
        for (int i = t; i >= x; i--) dp[i] = dp[i] || dp[i-x];
    return dp[t];
}
```

### Q146. Target Sum
```cpp
int findTargetSumWays(vector<int>& nums, int T) {
    int sum = accumulate(nums.begin(), nums.end(), 0);
    if (abs(T) > sum || (sum + T) % 2) return 0;
    int s = (sum + T) / 2;
    vector<int> dp(s + 1, 0); dp[0] = 1;
    for (int x : nums)
        for (int i = s; i >= x; i--) dp[i] += dp[i-x];
    return dp[s];
}
```

### Q147. Unique Paths
```cpp
int uniquePaths(int m, int n) {
    vector<vector<int>> dp(m, vector<int>(n, 1));
    for (int i = 1; i < m; i++)
        for (int j = 1; j < n; j++) dp[i][j] = dp[i-1][j] + dp[i][j-1];
    return dp[m-1][n-1];
}
```

### Q148. Minimum Path Sum
```cpp
int minPathSum(vector<vector<int>>& g) {
    int m = g.size(), n = g[0].size();
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++) {
            if (i == 0 && j == 0) continue;
            int t = INT_MAX;
            if (i > 0) t = min(t, g[i-1][j]);
            if (j > 0) t = min(t, g[i][j-1]);
            g[i][j] += t;
        }
    return g[m-1][n-1];
}
```

### Q149. Triangle
```cpp
int minimumTotal(vector<vector<int>>& t) {
    vector<int> dp = t.back();
    for (int i = t.size() - 2; i >= 0; i--)
        for (int j = 0; j <= i; j++) dp[j] = t[i][j] + min(dp[j], dp[j+1]);
    return dp[0];
}
```

### Q150. Maximal Square
```cpp
int maximalSquare(vector<vector<char>>& m) {
    int r = m.size(), c = m[0].size(), best = 0;
    vector<vector<int>> dp(r+1, vector<int>(c+1, 0));
    for (int i = 1; i <= r; i++)
        for (int j = 1; j <= c; j++) {
            if (m[i-1][j-1] == '1') {
                dp[i][j] = 1 + min({dp[i-1][j], dp[i][j-1], dp[i-1][j-1]});
                best = max(best, dp[i][j]);
            }
        }
    return best * best;
}
```

### Q151. Palindrome Partitioning II (min cuts)
```cpp
int minCut(string s) {
    int n = s.size();
    vector<vector<bool>> p(n, vector<bool>(n, false));
    vector<int> dp(n, 0);
    for (int i = 0; i < n; i++) {
        int mn = i;
        for (int j = 0; j <= i; j++) {
            if (s[i] == s[j] && (i - j < 2 || p[j+1][i-1])) {
                p[j][i] = true;
                mn = j == 0 ? 0 : min(mn, dp[j-1] + 1);
            }
        }
        dp[i] = mn;
    }
    return dp[n-1];
}
```

### Q152. Burst Balloons
```cpp
int maxCoins(vector<int>& nums) {
    nums.insert(nums.begin(), 1); nums.push_back(1);
    int n = nums.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int len = 2; len < n; len++)
        for (int l = 0; l + len < n; l++) {
            int r = l + len;
            for (int k = l + 1; k < r; k++)
                dp[l][r] = max(dp[l][r], dp[l][k] + dp[k][r] + nums[l]*nums[k]*nums[r]);
        }
    return dp[0][n-1];
}
```

### Q153. Decode Ways
```cpp
int numDecodings(string s) {
    int n = s.size();
    vector<int> dp(n + 1, 0);
    dp[0] = 1; dp[1] = s[0] != '0';
    for (int i = 2; i <= n; i++) {
        if (s[i-1] != '0') dp[i] += dp[i-1];
        int two = stoi(s.substr(i-2, 2));
        if (two >= 10 && two <= 26) dp[i] += dp[i-2];
    }
    return dp[n];
}
```

### Q154. Distinct Subsequences
```cpp
int numDistinct(string s, string t) {
    int m = s.size(), n = t.size();
    vector<vector<unsigned>> dp(m+1, vector<unsigned>(n+1, 0));
    for (int i = 0; i <= m; i++) dp[i][0] = 1;
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++) {
            dp[i][j] = dp[i-1][j];
            if (s[i-1] == t[j-1]) dp[i][j] += dp[i-1][j-1];
        }
    return dp[m][n];
}
```

### Q155. Interleaving String
```cpp
bool isInterleave(string a, string b, string c) {
    if (a.size() + b.size() != c.size()) return false;
    int m = a.size(), n = b.size();
    vector<vector<bool>> dp(m+1, vector<bool>(n+1, false));
    dp[0][0] = true;
    for (int i = 1; i <= m; i++) dp[i][0] = dp[i-1][0] && a[i-1] == c[i-1];
    for (int j = 1; j <= n; j++) dp[0][j] = dp[0][j-1] && b[j-1] == c[j-1];
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            dp[i][j] = (dp[i-1][j] && a[i-1] == c[i+j-1]) ||
                       (dp[i][j-1] && b[j-1] == c[i+j-1]);
    return dp[m][n];
}
```

### Q156. Regex Matching (DP)
*(See Q52)*

### Q157. Egg Drop Problem
```cpp
int superEggDrop(int k, int n) {
    vector<vector<int>> dp(k + 1, vector<int>(n + 1, 0));
    int m = 0;
    while (dp[k][m] < n) {
        m++;
        for (int i = 1; i <= k; i++)
            dp[i][m] = dp[i-1][m-1] + dp[i][m-1] + 1;
    }
    return m;
}
```

### Q158. 0/1 Knapsack
```cpp
int knapsack(int W, vector<int>& wt, vector<int>& val) {
    int n = wt.size();
    vector<int> dp(W + 1, 0);
    for (int i = 0; i < n; i++)
        for (int w = W; w >= wt[i]; w--)
            dp[w] = max(dp[w], dp[w-wt[i]] + val[i]);
    return dp[W];
}
```

### Q159. Matrix Chain Multiplication
```cpp
int matrixChain(vector<int>& p) {
    int n = p.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int len = 2; len < n; len++)
        for (int i = 1; i + len - 1 < n; i++) {
            int j = i + len - 1;
            dp[i][j] = INT_MAX;
            for (int k = i; k < j; k++)
                dp[i][j] = min(dp[i][j], dp[i][k] + dp[k+1][j] + p[i-1]*p[k]*p[j]);
        }
    return dp[1][n-1];
}
```

### Q160. Minimum Cost to Cut Sticks
```cpp
int minCost(int n, vector<int>& cuts) {
    cuts.push_back(0); cuts.push_back(n);
    sort(cuts.begin(), cuts.end());
    int c = cuts.size();
    vector<vector<int>> dp(c, vector<int>(c, 0));
    for (int len = 2; len < c; len++)
        for (int i = 0; i + len < c; i++) {
            int j = i + len;
            dp[i][j] = INT_MAX;
            for (int k = i + 1; k < j; k++)
                dp[i][j] = min(dp[i][j], dp[i][k] + dp[k][j] + cuts[j] - cuts[i]);
        }
    return dp[0][c-1];
}
```

### Q161. Jump Game (DP)
*(See Q16 — greedy is optimal)*

### Q162. Paint Fence
```cpp
int paintFence(int n, int k) {
    if (n == 0) return 0;
    if (n == 1) return k;
    int same = k, diff = k * (k - 1);
    for (int i = 3; i <= n; i++) {
        int t = diff;
        diff = (same + diff) * (k - 1);
        same = t;
    }
    return same + diff;
}
```

### Q163. Maximum Sum Rectangle in 2D
```cpp
int maxSumRect(vector<vector<int>>& m) {
    int rows = m.size(), cols = m[0].size(), best = INT_MIN;
    for (int l = 0; l < cols; l++) {
        vector<int> sum(rows, 0);
        for (int r = l; r < cols; r++) {
            for (int i = 0; i < rows; i++) sum[i] += m[i][r];
            int cur = sum[0], mx = sum[0];
            for (int i = 1; i < rows; i++) {
                cur = max(sum[i], cur + sum[i]);
                mx = max(mx, cur);
            }
            best = max(best, mx);
        }
    }
    return best;
}
```

### Q164. Minimum Number of Jumps
*(See Q17)*

### Q165. Dice Throw
```cpp
int diceThrow(int m, int n, int x) {
    vector<vector<int>> dp(n + 1, vector<int>(x + 1, 0));
    dp[0][0] = 1;
    for (int i = 1; i <= n; i++)
        for (int j = 1; j <= x; j++)
            for (int k = 1; k <= m && k <= j; k++)
                dp[i][j] += dp[i-1][j-k];
    return dp[n][x];
}
```

---

## Backtracking (Q166–Q175)

### Q166. Subsets
```cpp
void sub(vector<int>& nums, int i, vector<int>& cur, vector<vector<int>>& res) {
    if (i == nums.size()) { res.push_back(cur); return; }
    sub(nums, i+1, cur, res);
    cur.push_back(nums[i]);
    sub(nums, i+1, cur, res);
    cur.pop_back();
}
vector<vector<int>> subsets(vector<int>& nums) {
    vector<vector<int>> res; vector<int> cur;
    sub(nums, 0, cur, res);
    return res;
}
```

### Q167. Subsets II (duplicates)
```cpp
void sub2(vector<int>& nums, int i, vector<int>& cur, vector<vector<int>>& res) {
    res.push_back(cur);
    for (int j = i; j < nums.size(); j++) {
        if (j > i && nums[j] == nums[j-1]) continue;
        cur.push_back(nums[j]);
        sub2(nums, j+1, cur, res);
        cur.pop_back();
    }
}
vector<vector<int>> subsetsWithDup(vector<int>& nums) {
    sort(nums.begin(), nums.end());
    vector<vector<int>> res; vector<int> cur;
    sub2(nums, 0, cur, res);
    return res;
}
```

### Q168. Permutations
```cpp
void perm(vector<int>& nums, int i, vector<vector<int>>& res) {
    if (i == nums.size()) { res.push_back(nums); return; }
    for (int j = i; j < nums.size(); j++) {
        swap(nums[i], nums[j]);
        perm(nums, i+1, res);
        swap(nums[i], nums[j]);
    }
}
vector<vector<int>> permute(vector<int>& nums) {
    vector<vector<int>> res; perm(nums, 0, res); return res;
}
```

### Q169. Permutations II (duplicates)
```cpp
void perm2(vector<int>& nums, vector<bool>& used, vector<int>& cur, vector<vector<int>>& res) {
    if (cur.size() == nums.size()) { res.push_back(cur); return; }
    for (int i = 0; i < nums.size(); i++) {
        if (used[i] || (i && nums[i] == nums[i-1] && !used[i-1])) continue;
        used[i] = true; cur.push_back(nums[i]);
        perm2(nums, used, cur, res);
        used[i] = false; cur.pop_back();
    }
}
vector<vector<int>> permuteUnique(vector<int>& nums) {
    sort(nums.begin(), nums.end());
    vector<vector<int>> res; vector<int> cur;
    vector<bool> used(nums.size(), false);
    perm2(nums, used, cur, res);
    return res;
}
```

### Q170. Combination Sum
```cpp
void cs(vector<int>& c, int t, int i, vector<int>& cur, vector<vector<int>>& res) {
    if (t == 0) { res.push_back(cur); return; }
    if (t < 0 || i == c.size()) return;
    cur.push_back(c[i]);
    cs(c, t - c[i], i, cur, res);
    cur.pop_back();
    cs(c, t, i + 1, cur, res);
}
vector<vector<int>> combinationSum(vector<int>& c, int t) {
    vector<vector<int>> res; vector<int> cur;
    cs(c, t, 0, cur, res);
    return res;
}
```

### Q171. Combination Sum II
```cpp
void cs2(vector<int>& c, int t, int i, vector<int>& cur, vector<vector<int>>& res) {
    if (t == 0) { res.push_back(cur); return; }
    for (int j = i; j < c.size(); j++) {
        if (j > i && c[j] == c[j-1]) continue;
        if (c[j] > t) break;
        cur.push_back(c[j]);
        cs2(c, t - c[j], j + 1, cur, res);
        cur.pop_back();
    }
}
vector<vector<int>> combinationSum2(vector<int>& c, int t) {
    sort(c.begin(), c.end());
    vector<vector<int>> res; vector<int> cur;
    cs2(c, t, 0, cur, res);
    return res;
}
```

### Q172. N-Queens
```cpp
void nq(int n, int row, vector<string>& board, vector<vector<string>>& res, vector<bool>& cols, vector<bool>& d1, vector<bool>& d2) {
    if (row == n) { res.push_back(board); return; }
    for (int c = 0; c < n; c++) {
        if (cols[c] || d1[row+c] || d2[row-c+n-1]) continue;
        board[row][c] = 'Q'; cols[c] = d1[row+c] = d2[row-c+n-1] = true;
        nq(n, row+1, board, res, cols, d1, d2);
        board[row][c] = '.'; cols[c] = d1[row+c] = d2[row-c+n-1] = false;
    }
}
vector<vector<string>> solveNQueens(int n) {
    vector<vector<string>> res;
    vector<string> board(n, string(n, '.'));
    vector<bool> cols(n), d1(2*n), d2(2*n);
    nq(n, 0, board, res, cols, d1, d2);
    return res;
}
```

### Q173. Sudoku Solver
```cpp
bool isValid(vector<vector<char>>& b, int r, int c, char ch) {
    for (int i = 0; i < 9; i++) {
        if (b[r][i] == ch || b[i][c] == ch) return false;
        if (b[3*(r/3)+i/3][3*(c/3)+i%3] == ch) return false;
    }
    return true;
}
bool solve(vector<vector<char>>& b) {
    for (int i = 0; i < 9; i++)
        for (int j = 0; j < 9; j++) {
            if (b[i][j] != '.') continue;
            for (char c = '1'; c <= '9'; c++) {
                if (isValid(b, i, j, c)) {
                    b[i][j] = c;
                    if (solve(b)) return true;
                    b[i][j] = '.';
                }
            }
            return false;
        }
    return true;
}
void solveSudoku(vector<vector<char>>& b) { solve(b); }
```

### Q174. Letter Combinations of Phone Number
```cpp
void lc(string& d, int i, string cur, vector<string>& res, vector<string>& m) {
    if (i == d.size()) { if (!cur.empty()) res.push_back(cur); return; }
    for (char c : m[d[i] - '0']) lc(d, i + 1, cur + c, res, m);
}
vector<string> letterCombinations(string d) {
    vector<string> res, m{"","","abc","def","ghi","jkl","mno","pqrs","tuv","wxyz"};
    if (d.empty()) return res;
    lc(d, 0, "", res, m);
    return res;
}
```

### Q175. Palindrome Partitioning
```cpp
bool isPal(string& s, int l, int r) {
    while (l < r) if (s[l++] != s[r--]) return false;
    return true;
}
void pp(string& s, int i, vector<string>& cur, vector<vector<string>>& res) {
    if (i == s.size()) { res.push_back(cur); return; }
    for (int j = i; j < s.size(); j++)
        if (isPal(s, i, j)) {
            cur.push_back(s.substr(i, j - i + 1));
            pp(s, j + 1, cur, res);
            cur.pop_back();
        }
}
vector<vector<string>> partition(string s) {
    vector<vector<string>> res; vector<string> cur;
    pp(s, 0, cur, res);
    return res;
}
```

---

## Heaps / Priority Queue (Q176–Q183)

### Q176. Kth Largest Element in Array
```cpp
int findKthLargest(vector<int>& nums, int k) {
    priority_queue<int, vector<int>, greater<>> pq;
    for (int x : nums) { pq.push(x); if (pq.size() > k) pq.pop(); }
    return pq.top();
}
```

### Q177. Top K Frequent Elements
```cpp
vector<int> topKFrequent(vector<int>& nums, int k) {
    unordered_map<int,int> f;
    for (int x : nums) f[x]++;
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
    for (auto& [v, c] : f) {
        pq.push({c, v});
        if (pq.size() > k) pq.pop();
    }
    vector<int> res;
    while (!pq.empty()) { res.push_back(pq.top().second); pq.pop(); }
    return res;
}
```

### Q178. Find Median from Data Stream
```cpp
class MedianFinder {
    priority_queue<int> lo;
    priority_queue<int, vector<int>, greater<>> hi;
public:
    void addNum(int n) {
        lo.push(n);
        hi.push(lo.top()); lo.pop();
        if (lo.size() < hi.size()) { lo.push(hi.top()); hi.pop(); }
    }
    double findMedian() {
        return lo.size() > hi.size() ? lo.top() : (lo.top() + hi.top()) / 2.0;
    }
};
```

### Q179. K Closest Points to Origin
```cpp
vector<vector<int>> kClosest(vector<vector<int>>& pts, int k) {
    priority_queue<pair<int, vector<int>>> pq;
    for (auto& p : pts) {
        int d = p[0]*p[0] + p[1]*p[1];
        pq.push({d, p});
        if (pq.size() > k) pq.pop();
    }
    vector<vector<int>> res;
    while (!pq.empty()) { res.push_back(pq.top().second); pq.pop(); }
    return res;
}
```

### Q180. Task Scheduler
```cpp
int leastInterval(vector<char>& tasks, int n) {
    vector<int> c(26, 0);
    for (char t : tasks) c[t - 'A']++;
    int mx = *max_element(c.begin(), c.end()), cnt = count(c.begin(), c.end(), mx);
    return max((int)tasks.size(), (mx - 1) * (n + 1) + cnt);
}
```

### Q181. Reorganize String
```cpp
string reorganizeString(string s) {
    vector<int> c(26, 0);
    for (char ch : s) c[ch - 'a']++;
    priority_queue<pair<int,char>> pq;
    for (int i = 0; i < 26; i++) if (c[i]) pq.push({c[i], 'a' + i});
    string res;
    while (pq.size() >= 2) {
        auto [c1, ch1] = pq.top(); pq.pop();
        auto [c2, ch2] = pq.top(); pq.pop();
        res += ch1; res += ch2;
        if (--c1) pq.push({c1, ch1});
        if (--c2) pq.push({c2, ch2});
    }
    if (!pq.empty()) {
        if (pq.top().first > 1) return "";
        res += pq.top().second;
    }
    return res;
}
```

### Q182. Merge K Sorted Lists (Heap)
*(See Q73)*

### Q183. Sliding Window Median
```cpp
vector<double> medianSlidingWindow(vector<int>& nums, int k) {
    multiset<int> w(nums.begin(), nums.begin() + k);
    auto mid = next(w.begin(), (k - 1) / 2);
    vector<double> res;
    for (int i = k;; i++) {
        res.push_back(((double)*mid + *next(mid, 1 - k % 2)) / 2);
        if (i == nums.size()) break;
        w.insert(nums[i]);
        if (nums[i] < *mid) mid--;
        if (nums[i - k] <= *mid) mid++;
        w.erase(w.lower_bound(nums[i - k]));
    }
    return res;
}
```

---

## Binary Search (Q184–Q190)

### Q184. Binary Search
```cpp
int search(vector<int>& nums, int t) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (nums[m] == t) return m;
        if (nums[m] < t) l = m + 1; else r = m - 1;
    }
    return -1;
}
```

### Q185. Search a 2D Matrix
```cpp
bool searchMatrix(vector<vector<int>>& m, int t) {
    int rows = m.size(), cols = m[0].size();
    int l = 0, r = rows * cols - 1;
    while (l <= r) {
        int mid = (l + r) / 2;
        int v = m[mid / cols][mid % cols];
        if (v == t) return true;
        if (v < t) l = mid + 1; else r = mid - 1;
    }
    return false;
}
```

### Q186. Koko Eating Bananas
```cpp
int minEatingSpeed(vector<int>& piles, int h) {
    int l = 1, r = *max_element(piles.begin(), piles.end());
    while (l < r) {
        int m = (l + r) / 2;
        long t = 0;
        for (int p : piles) t += (p + m - 1) / m;
        if (t <= h) r = m; else l = m + 1;
    }
    return l;
}
```

### Q187. Find Peak Element
```cpp
int findPeakElement(vector<int>& nums) {
    int l = 0, r = nums.size() - 1;
    while (l < r) {
        int m = (l + r) / 2;
        if (nums[m] > nums[m+1]) r = m;
        else l = m + 1;
    }
    return l;
}
```

### Q188. Search Insert Position
```cpp
int searchInsert(vector<int>& nums, int t) {
    int l = 0, r = nums.size();
    while (l < r) {
        int m = (l + r) / 2;
        if (nums[m] < t) l = m + 1; else r = m;
    }
    return l;
}
```

### Q189. Capacity to Ship Packages
```cpp
int shipWithinDays(vector<int>& w, int D) {
    int l = *max_element(w.begin(), w.end()), r = accumulate(w.begin(), w.end(), 0);
    while (l < r) {
        int m = (l + r) / 2, days = 1, cur = 0;
        for (int x : w) {
            if (cur + x > m) { days++; cur = 0; }
            cur += x;
        }
        if (days <= D) r = m; else l = m + 1;
    }
    return l;
}
```

### Q190. Split Array Largest Sum
```cpp
int splitArray(vector<int>& nums, int k) {
    int l = *max_element(nums.begin(), nums.end()), r = accumulate(nums.begin(), nums.end(), 0);
    while (l < r) {
        int m = (l + r) / 2, parts = 1, cur = 0;
        for (int x : nums) {
            if (cur + x > m) { parts++; cur = 0; }
            cur += x;
        }
        if (parts <= k) r = m; else l = m + 1;
    }
    return l;
}
```

---

## Bit Manipulation (Q191–Q195)

### Q191. Number of 1 Bits
```cpp
int hammingWeight(uint32_t n) {
    int c = 0;
    while (n) { c++; n &= n - 1; }
    return c;
}
```

### Q192. Counting Bits
```cpp
vector<int> countBits(int n) {
    vector<int> r(n + 1, 0);
    for (int i = 1; i <= n; i++) r[i] = r[i >> 1] + (i & 1);
    return r;
}
```

### Q193. Reverse Bits
```cpp
uint32_t reverseBits(uint32_t n) {
    uint32_t r = 0;
    for (int i = 0; i < 32; i++) { r = (r << 1) | (n & 1); n >>= 1; }
    return r;
}
```

### Q194. Missing Number
```cpp
int missingNumber(vector<int>& nums) {
    int x = nums.size();
    for (int i = 0; i < nums.size(); i++) x ^= i ^ nums[i];
    return x;
}
```

### Q195. Sum of Two Integers (without `+`)
```cpp
int getSum(int a, int b) {
    while (b) {
        unsigned c = (unsigned)(a & b) << 1;
        a = a ^ b;
        b = c;
    }
    return a;
}
```

---

## Math (Q196–Q200)

### Q196. Fibonacci Number
```cpp
int fib(int n) {
    if (n < 2) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; i++) { int c = a + b; a = b; b = c; }
    return b;
}
```

### Q197. Pow(x, n)
```cpp
double myPow(double x, int n) {
    long N = n;
    if (N < 0) { x = 1 / x; N = -N; }
    double res = 1;
    while (N) {
        if (N & 1) res *= x;
        x *= x; N >>= 1;
    }
    return res;
}
```

### Q198. Sqrt(x)
```cpp
int mySqrt(int x) {
    long l = 0, r = x;
    while (l <= r) {
        long m = (l + r) / 2;
        if (m * m <= x) l = m + 1; else r = m - 1;
    }
    return r;
}
```

### Q199. Excel Sheet Column Number
```cpp
int titleToNumber(string s) {
    int n = 0;
    for (char c : s) n = n * 26 + (c - 'A' + 1);
    return n;
}
```

### Q200. Happy Number
```cpp
int squareSum(int n) {
    int s = 0;
    while (n) { int d = n % 10; s += d * d; n /= 10; }
    return s;
}
bool isHappy(int n) {
    int slow = n, fast = n;
    do {
        slow = squareSum(slow);
        fast = squareSum(squareSum(fast));
    } while (slow != fast);
    return slow == 1;
}
```

---

## Quick Topic Tracker

| Topic | Range | Confident | Needs Review |
|---|---|---|---|
| Arrays | Q1–Q30 | ☐ | ☐ |
| Strings | Q31–Q55 | ☐ | ☐ |
| Linked List | Q56–Q75 | ☐ | ☐ |
| Stack & Queue | Q76–Q90 | ☐ | ☐ |
| Binary Tree / BST | Q91–Q115 | ☐ | ☐ |
| Graphs | Q116–Q135 | ☐ | ☐ |
| Dynamic Programming | Q136–Q165 | ☐ | ☐ |
| Backtracking | Q166–Q175 | ☐ | ☐ |
| Heaps / Priority Queue | Q176–Q183 | ☐ | ☐ |
| Binary Search | Q184–Q190 | ☐ | ☐ |
| Bit Manipulation | Q191–Q195 | ☐ | ☐ |
| Math | Q196–Q200 | ☐ | ☐ |

---

## Recommended Headers for C++ Solutions

```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

*Pro tip: For each problem, first state the brute-force, then optimize. Always discuss time/space complexity in interviews.*
