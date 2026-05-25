# 200 Most-Asked DSA Interview Questions - ANNOTATED PART 2

## Arrays (Q26–Q30) + Strings (Q31–Q55) with Context & Comments

---

## Arrays (Q26–Q30)

### Q26. Find All Duplicates in an Array
**Problem Statement:**
Given an array of n+1 integers where each is between 1 and n, find all numbers that appear twice. Do it in O(n) time and O(1) space (excluding output).

**Example:**
```
Input: nums = [4, 3, 2, 7, 8, 2, 3, 1]
Output: [2, 3]
Explanation: 2 and 3 appear twice in the array
```

**Approach:** Use array indices as hash map by marking elements as negative.
- Time: O(n), Space: O(1)

```cpp
vector<int> findDuplicates(vector<int>& nums) {
    vector<int> res;
    
    for (int n : nums) {
        int idx = abs(n) - 1;  // Get index from value (1-based to 0-based)
        
        // If already marked negative, this number is duplicate
        if (nums[idx] < 0) {
            res.push_back(idx + 1);  // Convert back to original number
        } else {
            nums[idx] = -nums[idx];  // Mark as visited by negating
        }
    }
    return res;
}
```

---

### Q27. Set Matrix Zeroes
**Problem Statement:**
If a cell in a matrix is 0, set its entire row and column to 0 in-place without using extra space for tracking zeros.

**Example:**
```
Input: matrix = [[1, 1, 1], [1, 0, 1], [1, 1, 1]]
Output: [[1, 0, 1], [0, 0, 0], [1, 0, 1]]
```

**Approach:** Use first row and column as markers, handle them separately.
- Time: O(m*n), Space: O(1)

```cpp
void setZeroes(vector<vector<int>>& m) {
    int rows = m.size(), cols = m[0].size();
    bool firstCol = false;  // Track if first column has zeros
    
    // Find zeros and mark their rows/columns
    for (int i = 0; i < rows; i++) {
        if (m[i][0] == 0) firstCol = true;  // Mark first column
        
        // Mark row and column headers
        for (int j = 1; j < cols; j++) {
            if (m[i][j] == 0) {
                m[i][0] = 0;  // Mark row header
                m[0][j] = 0;  // Mark column header
            }
        }
    }
    
    // Set zeros based on markers (from right to left, bottom to top)
    for (int i = rows - 1; i >= 0; i--) {
        for (int j = cols - 1; j >= 1; j--) {
            // If row or column header is 0, set cell to 0
            if (m[i][0] == 0 || m[0][j] == 0) {
                m[i][j] = 0;
            }
        }
        
        // Handle first column separately
        if (firstCol) m[i][0] = 0;
    }
}
```

---

### Q28. Spiral Matrix
**Problem Statement:**
Return all elements of a matrix in spiral order (clockwise from outside to inside).

**Example:**
```
Input: matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
Output: [1, 2, 3, 6, 9, 8, 7, 4, 5]
```

**Approach:** Traverse in layers - right, down, left, up, then move inward.
- Time: O(m*n), Space: O(1) excluding output

```cpp
vector<int> spiralOrder(vector<vector<int>>& m) {
    vector<int> res;
    int top = 0, bot = m.size() - 1;     // Top and bottom boundaries
    int lt = 0, rt = m[0].size() - 1;    // Left and right boundaries
    
    while (top <= bot && lt <= rt) {
        // Traverse right along top row
        for (int j = lt; j <= rt; j++) {
            res.push_back(m[top][j]);
        }
        top++;
        
        // Traverse down along right column
        for (int i = top; i <= bot; i++) {
            res.push_back(m[i][rt]);
        }
        rt--;
        
        // Traverse left along bottom row (if exists)
        if (top <= bot) {
            for (int j = rt; j >= lt; j--) {
                res.push_back(m[bot][j]);
            }
            bot--;
        }
        
        // Traverse up along left column (if exists)
        if (lt <= rt) {
            for (int i = bot; i >= top; i--) {
                res.push_back(m[i][lt]);
            }
            lt++;
        }
    }
    return res;
}
```

---

### Q29. Rotate Image 90° Clockwise
**Problem Statement:**
Rotate a matrix 90 degrees clockwise in-place (cannot use extra matrix).

**Example:**
```
Input: matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
Output: [[7, 4, 1], [8, 5, 2], [9, 6, 3]]
```

**Approach:** Transpose matrix, then reverse each row.
- Time: O(m*n), Space: O(1)

```cpp
void rotate(vector<vector<int>>& m) {
    int n = m.size();
    
    // Step 1: Transpose - swap m[i][j] with m[j][i]
    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {
            swap(m[i][j], m[j][i]);
        }
    }
    
    // Step 2: Reverse each row
    for (auto& row : m) {
        reverse(row.begin(), row.end());
    }
}
```

---

### Q30. Pascal's Triangle
**Problem Statement:**
Generate Pascal's triangle with n rows where each element is sum of two elements above it.

**Example:**
```
Input: numRows = 5
Output: [[1], [1,1], [1,2,1], [1,3,3,1], [1,4,6,4,1]]
```

**Approach:** Build row by row, each element is sum of two above.
- Time: O(n²), Space: O(n²)

```cpp
vector<vector<int>> generate(int n) {
    vector<vector<int>> r(n);  // n rows
    
    for (int i = 0; i < n; i++) {
        r[i].assign(i + 1, 1);  // Initialize with 1s (edges are always 1)
        
        // Fill middle elements
        for (int j = 1; j < i; j++) {
            // Element = sum of two elements above
            r[i][j] = r[i-1][j-1] + r[i-1][j];
        }
    }
    return r;
}
```

---

## Strings (Q31–Q55)

### Q31. Valid Anagram
**Problem Statement:**
Determine if two strings are anagrams (contain same characters with same frequencies).

**Example:**
```
Input: s = "anagram", t = "nagaram"
Output: true
Input: s = "rat", t = "car"
Output: false
```

**Approach:** Count character frequencies and compare.
- Time: O(n), Space: O(1) - max 26 letters

```cpp
bool isAnagram(string s, string t) {
    // Different lengths can't be anagrams
    if (s.size() != t.size()) return false;
    
    int c[26] = {};  // Frequency counter for 26 letters
    
    // Count characters in s, subtract from t
    for (int i = 0; i < s.size(); i++) {
        c[s[i]-'a']++;      // Increment for s
        c[t[i]-'a']--;      // Decrement for t
    }
    
    // If all counts are 0, they're anagrams
    for (int x : c) {
        if (x) return false;
    }
    return true;
}
```

---

### Q32. Valid Palindrome
**Problem Statement:**
Check if a string is a palindrome considering only alphanumeric characters (case-insensitive).

**Example:**
```
Input: s = "A man, a plan, a canal: Panama"
Output: true
Input: s = "0P"
Output: false
```

**Approach:** Two-pointer, skip non-alphanumeric characters.
- Time: O(n), Space: O(1)

```cpp
bool isPalindrome(string s) {
    int l = 0, r = s.size() - 1;  // Two pointers
    
    while (l < r) {
        // Skip non-alphanumeric from left
        while (l < r && !isalnum(s[l])) l++;
        
        // Skip non-alphanumeric from right
        while (l < r && !isalnum(s[r])) r--;
        
        // Compare characters (case-insensitive)
        if (tolower(s[l++]) != tolower(s[r--])) {
            return false;
        }
    }
    return true;
}
```

---

### Q33. Longest Substring Without Repeating Characters
**Problem Statement:**
Find the length of the longest substring without repeating characters.

**Example:**
```
Input: s = "abcabcbb"
Output: 3
Explanation: "abc" is the answer
```

**Approach:** Sliding window with hash map to track character positions.
- Time: O(n), Space: O(min(n, charset_size))

```cpp
int lengthOfLongestSubstring(string s) {
    unordered_map<char,int> mp;  // Character -> latest index
    int l = 0, best = 0;          // Left pointer and max length
    
    for (int r = 0; r < s.size(); r++) {
        // If character seen and within current window
        if (mp.count(s[r]) && mp[s[r]] >= l) {
            l = mp[s[r]] + 1;  // Move left pointer to after previous occurrence
        }
        
        mp[s[r]] = r;                           // Update character's latest index
        best = max(best, r - l + 1);           // Update max length
    }
    return best;
}
```

---

### Q34. Longest Palindromic Substring
**Problem Statement:**
Find the longest palindromic substring in a string.

**Example:**
```
Input: s = "babad"
Output: "bab" or "aba"
Input: s = "ac"
Output: "a"
```

**Approach:** Expand around center for each position.
- Time: O(n²), Space: O(1)

```cpp
string longestPalindrome(string s) {
    int n = s.size(), start = 0, mx = 1;
    
    // Lambda to expand around center
    auto expand = [&](int l, int r) {
        // Expand while characters match
        while (l >= 0 && r < n && s[l] == s[r]) {
            l--;
            r++;
        }
        
        // Check if this palindrome is longest
        if (r - l - 1 > mx) {
            mx = r - l - 1;
            start = l + 1;
        }
    };
    
    // Try each position as center
    for (int i = 0; i < n; i++) {
        expand(i, i);          // Odd length palindromes
        expand(i, i + 1);      // Even length palindromes
    }
    
    return s.substr(start, mx);
}
```

---

### Q35. Palindromic Substrings (Count)
**Problem Statement:**
Count the number of palindromic substrings in a string.

**Example:**
```
Input: s = "abc"
Output: 3
Explanation: "a", "b", "c" (single chars are palindromes)
Input: s = "aab"
Output: 4
Explanation: "a", "a", "b", "aa"
```

**Approach:** Expand around each center and count palindromes.
- Time: O(n²), Space: O(1)

```cpp
int countSubstrings(string s) {
    int n = s.size(), cnt = 0;
    
    // Lambda to expand around center and count
    auto expand = [&](int l, int r) {
        while (l >= 0 && r < n && s[l--] == s[r++]) {
            cnt++;  // Count this palindrome
        }
    };
    
    // Try each position as center
    for (int i = 0; i < n; i++) {
        expand(i, i);          // Odd length
        expand(i, i + 1);      // Even length
    }
    return cnt;
}
```

---

### Q36. Group Anagrams
**Problem Statement:**
Group strings that are anagrams of each other together.

**Example:**
```
Input: strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
Output: [["bat"], ["nat", "tan"], ["ate", "eat", "tea"]]
```

**Approach:** Use sorted string as key in hash map.
- Time: O(n*k log k) where k is max string length, Space: O(n*k)

```cpp
vector<vector<string>> groupAnagrams(vector<string>& strs) {
    unordered_map<string, vector<string>> mp;  // sorted string -> list of anagrams
    
    for (auto& s : strs) {
        string k = s;
        sort(k.begin(), k.end());  // Sort to get canonical form
        mp[k].push_back(s);        // Add to anagram group
    }
    
    vector<vector<string>> res;
    for (auto& p : mp) {
        res.push_back(p.second);
    }
    return res;
}
```

---

### Q37. Encode and Decode Strings
**Problem Statement:**
Encode a list of strings into a single string and decode it back to the original list.

**Example:**
```
Input: ["hello", "world"]
Encode: "5#hello5#world"
Decode: ["hello", "world"]
```

**Approach:** Use length prefix delimited by special character.
- Time: O(n), Space: O(n)

```cpp
// Encode list of strings
string encode(vector<string>& v) {
    string s;
    for (auto& x : v) {
        // Format: length#string
        s += to_string(x.size()) + "#" + x;
    }
    return s;
}

// Decode string back to list
vector<string> decode(string s) {
    vector<string> r;
    int i = 0;
    
    while (i < s.size()) {
        // Find the # delimiter
        int j = s.find('#', i);
        
        // Extract length
        int n = stoi(s.substr(i, j - i));
        
        // Extract string of that length
        r.push_back(s.substr(j + 1, n));
        
        // Move to next encoded string
        i = j + 1 + n;
    }
    return r;
}
```

---

### Q38. Minimum Window Substring
**Problem Statement:**
Find the minimum window in a string containing all characters of another string.

**Example:**
```
Input: s = "ADOBECODEBANC", t = "ABC"
Output: "BANC"
```

**Approach:** Sliding window with two-pointer technique.
- Time: O(n), Space: O(charset_size)

```cpp
string minWindow(string s, string t) {
    if (t.empty()) return "";
    
    unordered_map<char,int> need, have;  // Character frequency maps
    for (char c : t) need[c]++;          // Count required characters
    
    int req = need.size();  // Number of unique chars needed
    int made = 0;           // Number of unique chars satisfied
    int l = 0;
    int bestL = 0, bestLen = INT_MAX;
    
    for (int r = 0; r < s.size(); r++) {
        have[s[r]]++;
        
        // If this character count matches requirement
        if (need.count(s[r]) && have[s[r]] == need[s[r]]) {
            made++;
        }
        
        // Shrink window from left while valid
        while (made == req) {
            // Update best window
            if (r - l + 1 < bestLen) {
                bestLen = r - l + 1;
                bestL = l;
            }
            
            have[s[l]]--;
            if (need.count(s[l]) && have[s[l]] < need[s[l]]) {
                made--;
            }
            l++;
        }
    }
    
    return bestLen == INT_MAX ? "" : s.substr(bestL, bestLen);
}
```

---

### Q39. Reverse Words in a String
**Problem Statement:**
Reverse the order of words in a string (multiple spaces become single space).

**Example:**
```
Input: s = "  Hello World  "
Output: "World Hello"
```

**Approach:** Use stringstream to parse words and build reversed result.
- Time: O(n), Space: O(n)

```cpp
string reverseWords(string s) {
    stringstream ss(s);
    string w, res;
    
    // Read each word
    while (ss >> w) {
        // Prepend to result, add space if not first word
        res = w + (res.empty() ? "" : " " + res);
    }
    return res;
}
```

---

### Q40. String to Integer (atoi)
**Problem Statement:**
Convert a string to an integer with rules: skip leading spaces, handle +/-, stop at non-digit, handle overflow.

**Example:**
```
Input: s = "42"
Output: 42
Input: s = "-91283472332"
Output: -2147483648 (overflow, return INT_MIN)
```

**Approach:** Manual parsing with overflow checking.
- Time: O(n), Space: O(1)

```cpp
int myAtoi(string s) {
    int i = 0, n = s.size(), sign = 1;
    long res = 0;
    
    // Step 1: Skip leading spaces
    while (i < n && s[i] == ' ') i++;
    
    // Step 2: Check sign
    if (i < n && (s[i] == '+' || s[i] == '-')) {
        sign = s[i++] == '-' ? -1 : 1;
    }
    
    // Step 3: Read digits
    while (i < n && isdigit(s[i])) {
        res = res * 10 + (s[i++] - '0');
        
        // Step 4: Check overflow
        if (sign * res > INT_MAX) return INT_MAX;
        if (sign * res < INT_MIN) return INT_MIN;
    }
    
    return sign * res;
}
```

---

### Q41. Implement strStr() (KMP Algorithm)
**Problem Statement:**
Find the first occurrence of a needle in a haystack (implement indexOf).

**Example:**
```
Input: haystack = "sadbutsad", needle = "sad"
Output: 0
Input: haystack = "leetcode", needle = "leeto"
Output: -1
```

**Approach:** KMP (Knuth-Morris-Pratt) pattern matching for O(n) solution.
- Time: O(n + m), Space: O(m)

```cpp
int strStr(string h, string n) {
    if (n.empty()) return 0;
    
    int nn = n.size();
    vector<int> lps(nn, 0);  // Longest Proper Prefix which is also Suffix
    
    // Build LPS array
    for (int i = 1, len = 0; i < nn;) {
        if (n[i] == n[len]) {
            lps[i++] = ++len;
        } else if (len) {
            len = lps[len-1];
        } else {
            lps[i++] = 0;
        }
    }
    
    // Search for pattern
    for (int i = 0, j = 0; i < h.size();) {
        if (h[i] == n[j]) {
            i++;
            j++;
        }
        
        if (j == nn) {
            return i - j;  // Found at position i-j
        } else if (i < h.size() && h[i] != n[j]) {
            if (j) {
                j = lps[j-1];
            } else {
                i++;
            }
        }
    }
    return -1;
}
```

---

### Q42. Count and Say
**Problem Statement:**
Generate the nth term of the look-and-say sequence.

**Example:**
```
n = 1: "1"
n = 2: "11" (one 1)
n = 3: "21" (two 1s)
n = 4: "1211" (one 2, one 1)
n = 5: "111221" (one 1, one 2, two 1s)
```

**Approach:** Iteratively build each term by counting consecutive characters.
- Time: O(n*m), Space: O(m)

```cpp
string countAndSay(int n) {
    string s = "1";  // Start with first term
    
    // Generate terms 2 to n
    for (int i = 1; i < n; i++) {
        string t;
        
        // Group consecutive characters
        for (int j = 0; j < s.size();) {
            int k = j;
            // Count consecutive same characters
            while (k < s.size() && s[k] == s[j]) k++;
            
            // Append count and character
            t += to_string(k - j) + s[j];
            j = k;
        }
        s = t;
    }
    return s;
}
```

---

### Q43. Longest Common Prefix
**Problem Statement:**
Find the longest common prefix among all strings in an array.

**Example:**
```
Input: strs = ["flower", "flow", "flight"]
Output: "fl"
Input: strs = ["dog", "racecar", "car"]
Output: ""
```

**Approach:** Vertical scanning - compare characters column by column.
- Time: O(n*m) where n=strings, m=min length, Space: O(1)

```cpp
string longestCommonPrefix(vector<string>& strs) {
    if (strs.empty()) return "";
    
    string p = strs[0];  // Start with first string
    
    // Check against all other strings
    for (int i = 1; i < strs.size(); i++) {
        // Reduce prefix while it's not found in current string
        while (strs[i].find(p) != 0) {  // 0 means starts with
            p = p.substr(0, p.size() - 1);  // Remove last character
            if (p.empty()) return "";
        }
    }
    return p;
}
```

---

### Q44. Valid Parentheses
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
        // Push opening brackets
        if (c == '(' || c == '[' || c == '{') {
            st.push(c);
        } else {
            // Check closing brackets
            if (st.empty()) return false;
            
            char t = st.top();
            st.pop();
            
            // Verify matching pair
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

### Q45. Generate Parentheses
**Problem Statement:**
Generate all valid combinations of n pairs of parentheses.

**Example:**
```
Input: n = 3
Output: ["((()))", "(()())", "(())()", "()(())", "()()()"]
```

**Approach:** Backtracking - track open and close counts.
- Time: O(4^n / sqrt(n)), Space: O(n)

```cpp
void gen(int o, int c, int n, string s, vector<string>& res) {
    // Base case: string is complete
    if (s.size() == 2 * n) {
        res.push_back(s);
        return;
    }
    
    // Add opening bracket if haven't used all
    if (o < n) {
        gen(o + 1, c, n, s + '(', res);
    }
    
    // Add closing bracket if it doesn't exceed opening
    if (c < o) {
        gen(o, c + 1, n, s + ')', res);
    }
}

vector<string> generateParenthesis(int n) {
    vector<string> res;
    gen(0, 0, n, "", res);
    return res;
}
```

---

### Q46. Longest Repeating Character Replacement
**Problem Statement:**
Find longest substring where you can make all characters same by replacing at most k characters.

**Example:**
```
Input: s = "ABAB", k = 2
Output: 4 (replace 2 As with Bs -> "BBBB")
Input: s = "AABCCCCCCCD", k = 2
Output: 5 (window "CCCCC")
```

**Approach:** Sliding window - track character frequency and validity.
- Time: O(n), Space: O(26)

```cpp
int characterReplacement(string s, int k) {
    int cnt[26] = {};   // Character frequency
    int l = 0;
    int maxF = 0;       // Max frequency in current window
    int best = 0;
    
    for (int r = 0; r < s.size(); r++) {
        // Update max frequency
        maxF = max(maxF, ++cnt[s[r]-'A']);
        
        // If window is invalid (too many different characters)
        while ((r - l + 1) - maxF > k) {
            cnt[s[l++]-'A']--;  // Shrink window
        }
        
        best = max(best, r - l + 1);
    }
    return best;
}
```

---

### Q47. Is Subsequence
**Problem Statement:**
Check if s is a subsequence of t (characters of s appear in t in same order but not necessarily consecutive).

**Example:**
```
Input: s = "ace", t = "abcde"
Output: true
Input: s = "aec", t = "abcde"
Output: false
```

**Approach:** Single pass - move pointer in s only when character matches.
- Time: O(n), Space: O(1)

```cpp
bool isSubsequence(string s, string t) {
    int i = 0;  // Pointer in s
    
    for (char c : t) {
        // If character matches and we haven't exhausted s
        if (i < s.size() && c == s[i]) {
            i++;
        }
    }
    
    return i == s.size();  // All of s matched
}
```

---

### Q48. ZigZag Conversion
**Problem Statement:**
Convert string by writing in zigzag pattern with n rows and read left to right.

**Example:**
```
Input: s = "PAYPALISHIRING", numRows = 3
Output: "PAHNAPLSIIGYIR"

Pattern (numRows=3):
P   A   H   N
 A P L S I I G
  Y   I   R
```

**Approach:** Simulate zigzag by tracking current row and direction.
- Time: O(n), Space: O(n)

```cpp
string convert(string s, int r) {
    if (r == 1) return s;
    
    vector<string> rows(min((int)s.size(), r));
    int cur = 0, dir = -1;  // Current row and direction (-1=up, 1=down)
    
    for (char c : s) {
        rows[cur] += c;  // Add character to current row
        
        // Change direction at top or bottom
        if (cur == 0 || cur == r - 1) {
            dir = -dir;
        }
        cur += dir;
    }
    
    // Concatenate all rows
    string res;
    for (auto& x : rows) {
        res += x;
    }
    return res;
}
```

---

### Q49. Roman to Integer
**Problem Statement:**
Convert a Roman numeral to an integer.

**Example:**
```
Input: s = "MCMXCIV"
Output: 1994 (1000 + 900 + 90 + 4)
```

**Approach:** Iterate through string, if character value < next, subtract; otherwise add.
- Time: O(n), Space: O(1)

```cpp
int romanToInt(string s) {
    unordered_map<char,int> m{
        {'I',1}, {'V',5}, {'X',10}, {'L',50}, 
        {'C',100}, {'D',500}, {'M',1000}
    };
    
    int res = 0;
    for (int i = 0; i < s.size(); i++) {
        // If smaller value before larger, subtract (like IV=4)
        if (i + 1 < s.size() && m[s[i]] < m[s[i+1]]) {
            res -= m[s[i]];
        } else {
            res += m[s[i]];
        }
    }
    return res;
}
```

---

### Q50. Integer to Roman
**Problem Statement:**
Convert an integer to a Roman numeral.

**Example:**
```
Input: num = 1994
Output: "MCMXCIV"
```

**Approach:** Greedy - use largest possible values first.
- Time: O(1) - fixed set of symbols, Space: O(1)

```cpp
string intToRoman(int n) {
    // Pairs of values and their symbols in descending order
    vector<pair<int,string>> v{
        {1000,"M"}, {900,"CM"}, {500,"D"}, {400,"CD"}, 
        {100,"C"}, {90,"XC"}, {50,"L"}, {40,"XL"}, 
        {10,"X"}, {9,"IX"}, {5,"V"}, {4,"IV"}, {1,"I"}
    };
    
    string r;
    for (auto& p : v) {
        // Use symbol as many times as possible
        while (n >= p.first) {
            r += p.second;
            n -= p.first;
        }
    }
    return r;
}
```

---

### Q51. Word Search (DFS)
**Problem Statement:**
Find if a word exists in a 2D board where you can move up/down/left/right (no cell reuse).

**Example:**
```
Input: board = [['A','B','C','E'],['S','F','C','S'],['A','D','E','E']], word = "ABCCED"
Output: true
```

**Approach:** DFS with backtracking, mark visited cells with special character.
- Time: O(m*n*4^L) where L is word length, Space: O(L)

```cpp
bool dfs(vector<vector<char>>& b, string& w, int i, int j, int k) {
    // Base: matched entire word
    if (k == w.size()) return true;
    
    // Out of bounds or character mismatch or already visited
    if (i<0 || j<0 || i>=b.size() || j>=b[0].size() || b[i][j]!= w[k]) {
        return false;
    }
    
    // Mark as visited
    char t = b[i][j];
    b[i][j] = '#';
    
    // Try all 4 directions
    bool ok = dfs(b,w,i+1,j,k+1) || dfs(b,w,i-1,j,k+1) || 
             dfs(b,w,i,j+1,k+1) || dfs(b,w,i,j-1,k+1);
    
    // Restore cell
    b[i][j] = t;
    return ok;
}

bool exist(vector<vector<char>>& b, string w) {
    for (int i = 0; i < b.size(); i++) {
        for (int j = 0; j < b[0].size(); j++) {
            if (dfs(b, w, i, j, 0)) {
                return true;
            }
        }
    }
    return false;
}
```

---

### Q52. Regular Expression Matching
**Problem Statement:**
Implement regex matching with support for '.' (any character) and '*' (0 or more of previous).

**Example:**
```
Input: s = "aa", p = "a"
Output: false
Input: s = "aa", p = "a*"
Output: true (0 or more 'a')
```

**Approach:** Dynamic programming with 2D DP table.
- Time: O(m*n), Space: O(m*n)

```cpp
bool isMatch(string s, string p) {
    int m = s.size(), n = p.size();
    vector<vector<bool>> dp(m+1, vector<bool>(n+1, false));
    dp[0][0] = true;  // Empty string matches empty pattern
    
    // Handle patterns like a*, a*b* (empty string match)
    for (int j = 1; j <= n; j++) {
        if (p[j-1] == '*') {
            dp[0][j] = dp[0][j-2];
        }
    }
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (p[j-1] == '*') {
                // Two options: 0 occurrences (dp[i][j-2]) or 
                // 1+ occurrences (dp[i-1][j] if char matches)
                dp[i][j] = dp[i][j-2] || 
                          ((p[j-2] == s[i-1] || p[j-2] == '.') && dp[i-1][j]);
            } else if (p[j-1] == '.' || p[j-1] == s[i-1]) {
                // Character matches or . matches anything
                dp[i][j] = dp[i-1][j-1];
            }
        }
    }
    return dp[m][n];
}
```

---

### Q53. Wildcard Matching
**Problem Statement:**
Implement wildcard pattern matching with '?' (any single char) and '*' (any sequence).

**Example:**
```
Input: s = "aa", p = "*"
Output: true
Input: s = "aa", p = "*?"
Output: false
```

**Approach:** Dynamic programming similar to regex but simpler rules.
- Time: O(m*n), Space: O(m*n)

```cpp
bool isMatchW(string s, string p) {
    int m = s.size(), n = p.size();
    vector<vector<bool>> dp(m+1, vector<bool>(n+1, false));
    dp[0][0] = true;
    
    // Handle leading * in pattern
    for (int j = 1; j <= n && p[j-1] == '*'; j++) {
        dp[0][j] = true;
    }
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (p[j-1] == '*') {
                // * matches empty or one or more characters
                dp[i][j] = dp[i-1][j] || dp[i][j-1];
            } else if (p[j-1] == '?' || p[j-1] == s[i-1]) {
                // ? matches any, or exact match
                dp[i][j] = dp[i-1][j-1];
            }
        }
    }
    return dp[m][n];
}
```

---

### Q54. Scramble String
**Problem Statement:**
Check if s2 is a scrambled version of s1 (can recursively split and swap halves).

**Example:**
```
Input: s1 = "great", s2 = "rgeat"
Output: true
```

**Approach:** Recursion with memoization - try all split points.
- Time: O(4^n), Space: O(n^3)

```cpp
unordered_map<string,bool> memo;

bool isScramble(string s1, string s2) {
    string k = s1 + "#" + s2;
    if (memo.count(k)) return memo[k];
    
    // Base case: same strings
    if (s1 == s2) return memo[k] = true;
    
    // Check if have same characters
    int c[26] = {};
    for (char x : s1) c[x-'a']++;
    for (char x : s2) c[x-'a']--;
    for (int x : c) if (x) return memo[k] = false;
    
    // Try all split points
    int n = s1.size();
    for (int i = 1; i < n; i++) {
        // Case 1: Don't swap (split at same position)
        if (isScramble(s1.substr(0,i), s2.substr(0,i)) && 
            isScramble(s1.substr(i), s2.substr(i))) {
            return memo[k] = true;
        }
        
        // Case 2: Swap (left of s1 matches right of s2)
        if (isScramble(s1.substr(0,i), s2.substr(n-i)) && 
            isScramble(s1.substr(i), s2.substr(0,n-i))) {
            return memo[k] = true;
        }
    }
    return memo[k] = false;
}
```

---

### Q55. Edit Distance
**Problem Statement:**
Find minimum edits (insert, delete, replace) to transform one string into another.

**Example:**
```
Input: word1 = "horse", word2 = "ros"
Output: 3 (delete 'h', replace 'r' with 'o', delete 'e')
```

**Approach:** Dynamic programming - DP[i][j] = edits for first i chars of word1 to j chars of word2.
- Time: O(m*n), Space: O(m*n)

```cpp
int minDistance(string a, string b) {
    int m = a.size(), n = b.size();
    vector<vector<int>> dp(m+1, vector<int>(n+1));
    
    // Base cases: transform from/to empty string
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    
    // Fill DP table
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (a[i-1] == b[j-1]) {
                // Characters match, no edit needed
                dp[i][j] = dp[i-1][j-1];
            } else {
                // Minimum of: replace, delete, insert
                dp[i][j] = 1 + min({
                    dp[i-1][j],      // Delete from a
                    dp[i][j-1],      // Insert into a
                    dp[i-1][j-1]     // Replace
                });
            }
        }
    }
    return dp[m][n];
}
```

---

## Summary Table for Q26-Q55

| Question | Topic | Approach | Time | Space |
|----------|-------|----------|------|-------|
| Q26 | Find Duplicates | Array as Hash | O(n) | O(1) |
| Q27 | Set Matrix Zeros | In-place Markers | O(m*n) | O(1) |
| Q28 | Spiral Matrix | Layer Traversal | O(m*n) | O(1) |
| Q29 | Rotate Image | Transpose + Reverse | O(m*n) | O(1) |
| Q30 | Pascal's Triangle | DP Row-by-row | O(n²) | O(n²) |
| Q31 | Valid Anagram | Frequency Count | O(n) | O(1) |
| Q32 | Valid Palindrome | Two Pointer | O(n) | O(1) |
| Q33 | Longest Substring | Sliding Window | O(n) | O(min(n, m)) |
| Q34 | Longest Palindrome | Expand Around Center | O(n²) | O(1) |
| Q35 | Count Palindromes | Expand Around Center | O(n²) | O(1) |
| Q36 | Group Anagrams | Sorting + Hash | O(n*k log k) | O(n*k) |
| Q37 | Encode/Decode | Length Prefix | O(n) | O(n) |
| Q38 | Min Window | Sliding Window | O(n) | O(charset) |
| Q39 | Reverse Words | Stringstream | O(n) | O(n) |
| Q40 | String to Integer | Manual Parsing | O(n) | O(1) |
| Q41 | Implement strStr | KMP | O(n+m) | O(m) |
| Q42 | Count and Say | Iterative | O(n*m) | O(m) |
| Q43 | Longest Common Prefix | Vertical Scan | O(n*m) | O(1) |
| Q44 | Valid Parentheses | Stack | O(n) | O(n) |
| Q45 | Generate Parentheses | Backtracking | O(4^n/√n) | O(n) |
| Q46 | Char Replacement | Sliding Window | O(n) | O(26) |
| Q47 | Is Subsequence | Single Pass | O(n) | O(1) |
| Q48 | ZigZag Conversion | Simulation | O(n) | O(n) |
| Q49 | Roman to Integer | Hash Map | O(n) | O(1) |
| Q50 | Integer to Roman | Greedy | O(1) | O(1) |
| Q51 | Word Search | DFS + Backtracking | O(m*n*4^L) | O(L) |
| Q52 | Regex Matching | DP | O(m*n) | O(m*n) |
| Q53 | Wildcard Matching | DP | O(m*n) | O(m*n) |
| Q54 | Scramble String | Recursion + Memo | O(4^n) | O(n³) |
| Q55 | Edit Distance | DP | O(m*n) | O(m*n) |

---

**END OF PART 2 (Q26-Q55)**

Next: Part 3 will cover Linked Lists (Q56-Q75)

