Link: 

[115. Distinct Subsequences](https://leetcode.com/problems/distinct-subsequences/description/)

Problem:

- 给定两个字符串S和T，求S的子字符串集合中有多少个T
- 子字符串的定义是，S删除0至S.length个字符，但保留剩下字符的顺序得到的字符串。例如，`abc`和`acd`都是`abcd`的子字符串，但`adc`不是
- 例如，`S=absageab T=ab` 那么结果是4
```
absageab
^^
absageab
^      ^
absageab
   ^   ^
absageab
      ^^
```

Solution:

采用DP算法，`dp[len(T)+1][len(S)+1]`，`dp[i][j]`表示`T`的前`i-1`个字符在`S`的前`j-1`个字符的子字符串中出现了多少次。
dp计算过程是：
1. `i=0`表示T为空时，S任意长度的子字符串集合都包含空字符串，但都只出现1次，故`dp[0][j]`都为1
2. `j=0`表示S为空时，子字符串集合只有空字符串，故`dp[i][0]=0`
3. `i>0, j>0`:
  - 若`S[j-1] == T[i-1]`，表示T的第`i`个字符与S的第`j`个字符相同，`dp[i][j]`应该包含`T[0:i]`在`S[0:j]`子字符串集合中出现的次数，和`T[0:i+1]`已经在`S[0:j+1]`子字符串集合中出现的次数，那么`dp[i][j]=dp[i-1][j-1] + dp[i][j-1]`
  - 否则，`dp[i][j]`与`dp[i][j-1]`相同，不能累加`dp[i-1]`中的结果
  
Code:
```
class Solution:
    def numDistinct(self, s, t):
        """
        :type s: str
        :type t: str
        :rtype: int
        """
        dp = [[0 for _ in range(len(s)+1)] for _ in range(len(t)+1)]
        for i in range(len(s)+1):
            dp[0][i] = 1
            
        for i in range(1, len(t)+1):
            for j in range(1, len(s)+1):
                if s[j-1] == t[i-1]:
                    dp[i][j] = dp[i][j-1] + dp[i-1][j-1]
                else:
                    dp[i][j] = dp[i][j-1]
        return dp[len(t)][len(s)]
```
