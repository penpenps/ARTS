**Problem**

[945. Minimum Increment to Make Array Unique](https://leetcode.com/problems/minimum-increment-to-make-array-unique/description/)

Given an array of integers A, a move consists of choosing any A[i], and incrementing it by 1.

Return the least number of moves to make every value in A unique.

Example 1:
```
Input: [1,2,2]
Output: 1
Explanation:  After 1 move, the array could be [1, 2, 3].
```
Example 2:
```
Input: [3,2,1,2,1,7]
Output: 6
Explanation:  After 6 moves, the array could be [3, 4, 1, 2, 5, 7].
It can be shown with 5 or less moves that it is impossible for the array to have all unique values.
```

**Solution**

It's straight forword to in a for loop to scan all the elements and look for next greater number which not appear.

Here is the implement about that idea:
```python
class Solution:
    def minIncrementForUnique(self, A):
        """
        :type A: List[int]
        :rtype: int
        """
        ans = 0
        s = set()
        for i, x in enumerate(A):
            while x in s:
                x += 1
                ans += 1
            s.add(x)
        return ans
```
The time complexity is O(N^2) as for each element, there is a while loop to lookup the satisfied number. And the space complexity is O(len(A)) as we use a set to store the numbers has appeared.

Unfortunately, this solution is time out.

To avoid the whole loop inside for loop, let's think about is it necessary to increase one by one to get next number? If the array `A` is sorted, the next number about current element `x` is alternatively `x+1` or just the next number which doesn't need to increase. So let's optimize the solution:
```python
class Solution:
    def minIncrementForUnique(self, A):
        """
        :type A: List[int]
        :rtype: int
        """
        A.sort()
        ans = 0
        need = 0
        for x in A:
            ans += max(need-x, 0)
            need = max(need, x) + 1
        return ans
```
The time complexity is O(NlogN) as there is a sort operation about the array and the space complexity is O(1) now.
