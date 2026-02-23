---
title: Leetcode review
date: 2024-10-15 22:53:51
tags:
  - Data Structure and Algorithm
  - Leetcode
categories:
  - Data Structure and Algorithm
  - Leetcode
cover: https://pics.findfuns.org/leetcode.jpg
---
# [399. Evaluate Division](https://leetcode.com/problems/evaluate-division/)

问题描述：

You are given an array of variable pairs `equations` and an array of real numbers `values`, where `equations[i] = [Ai, Bi]` and `values[i]` represent the equation `Ai / Bi = values[i]`. Each `Ai` or `Bi` is a string that represents a single variable.

You are also given some `queries`, where `queries[j] = [Cj, Dj]` represents the `jth` query where you must find the answer for `Cj / Dj = ?`.

Return *the answers to all queries*. If a single answer cannot be determined, return `-1.0`.

**Note:** The input is always valid. You may assume that evaluating the queries will not result in division by zero and that there is no contradiction.

**Note:** The variables that do not occur in the list of equations are undefined, so the answer cannot be determined for them.

**Example 1:**

```
Input: equations = [["a","b"],["b","c"]], values = [2.0,3.0], queries = [["a","c"],["b","a"],["a","e"],["a","a"],["x","x"]]
Output: [6.00000,0.50000,-1.00000,1.00000,-1.00000]
Explanation: 
Given: a / b = 2.0, b / c = 3.0
queries are: a / c = ?, b / a = ?, a / e = ?, a / a = ?, x / x = ? 
return: [6.0, 0.5, -1.0, 1.0, -1.0 ]
note: x is undefined => -1.0
```

**Example 2:**

```
Input: equations = [["a","b"],["b","c"],["bc","cd"]], values = [1.5,2.5,5.0], queries = [["a","c"],["c","b"],["bc","cd"],["cd","bc"]]
Output: [3.75000,0.40000,5.00000,0.20000]
```

**Example 3:**

```
Input: equations = [["a","b"]], values = [0.5], queries = [["a","b"],["b","a"],["a","c"],["x","y"]]
Output: [0.50000,2.00000,-1.00000,-1.00000]
```

<img src="https://pics.findfuns.org/Problem 399.png" style="zoom:50%;" />

这是一道典型的图论相关问题，思路是构造一个邻接矩阵`matrix`， `matrix[i][j]`代表`i/j`的值，如果没有这个值就为0。

构造好这个矩阵之后就可以通过BFS去递归的寻找关系，比如在Example 1中需要找到`a/c`，那么就可以先通过dfs找到`a/b`，然后再通过dfs进一步找到`b/c`从而获得`a/c`。

具体的代码思路如下

```java
class Solution {
    Map<String, Integer> map;
    double[][] matrix;
    int cnt = 0;
    double[] res;
    public double[] calcEquation(List<List<String>> equations, double[] values, List<List<String>> queries) {
        map = new HashMap<>();
        for(List<String> list: equations) {
            for(String str: list) {
                if(!map.containsKey(str)) {
                    map.put(str, cnt ++);
                }
            }
        }
        int num = map.size();
        matrix = new double[num][num];
        for(int i = 0; i < num; i ++) {
            matrix[i][i] = 1.0;
        }
        for(int i = 0; i < equations.size(); i ++) {
            String from = equations.get(i).get(0);
            String to = equations.get(i).get(1);
            matrix[map.get(from)][map.get(to)] = values[i];
            matrix[map.get(to)][map.get(from)] = 1.0 / values[i];
        }
        res = new double[queries.size()];
        cnt = 0;
        for(List<String> list: queries) {
            if(!map.containsKey(list.get(0)) || !map.containsKey(list.get(1))) {
                res[cnt] = -1.0;
            } else {
                int from = map.get(list.get(0));
                int to = map.get(list.get(1));
                dfs(from, to, 1.0, new HashSet<>());
                if(res[cnt] == 0) {
                    res[cnt] = -1.0;
                }
            }   
            cnt ++;
        }
        return res;
    }
    private void dfs(int from, int to, double multiple, Set<Integer> set) {
        if(set.contains(from)) {
            return;
        }
        for(int i = 0; i < matrix[from].length; i ++) {
            if(i == to && matrix[from][to] != 0.0) {
                res[cnt] = multiple * matrix[from][to];
                return;
            }
            if(matrix[from][i] != 0.0) {
                set.add(from);
                dfs(i, to, multiple * matrix[from][i], set);
                set.remove(from);
            }
        }
    }
}
```

时间复杂度分析：

- 构建map映射，即字符串和matrix中对应下标时的复杂度为`O(E)`,E为Equations的大小。

- 构建邻接矩阵时的复杂度为`O(E)`。

- dfs方法在调用时最坏情况下会遍历邻接矩阵的每一个值，假设共有n个不同的字符串，即邻接矩阵有n^2个值，所以时间复杂度为`O(n^2)`。
- for循环遍历queries时，假设共有Q个query，那么每次调用一次dfs，时间复杂度为`O(Q * n^2)`

综上时间复杂度为`O(E + Q * n^2)`。

空间复杂度分析：

- matrix共有n^2个节点，复杂度为`O(n^2)`
- map共存储n个节点，复杂度为`O(n)`
- dfs时递归的深度可能会达到所有节点，复杂度为`O(n)`
- dfs时的set最坏情况会包含所有节点，复杂度为`O(n)`
- res数组假设有Q个query，复杂度为`O(Q)`

综上空间复杂度为`O(Q + n^2)`



# [994. Rotting Oranges](https://leetcode.com/problems/rotting-oranges/)

问题描述：

![](https://pics.findfuns.org/Problem 994.png)



思路是先遍历grid，如果找到“烂橘子”就加入到Queue中，遍历完成之后对Queue中的所有“烂橘子”进行BFS并记录每次BFS的时间，动态更新最长时间。最后检查一遍grid，如果还存在新鲜的橘子就返回-1，如果没有就返回最长时间。

代码如下：

```java
class Solution {
    public int orangesRotting(int[][] grid) {
        Queue<int[]> queue = new LinkedList<>();
        int res = 0;
        for(int i = 0; i < grid.length; i ++) {
            for(int j = 0; j < grid[i].length; j ++) {
                if(grid[i][j] == 2) {
                    queue.offer(new int[] {i, j, 0});
                }
            }
        }
        int[][] directions = new int[][] {{0, 1}, {0, -1}, {1, 0}, {-1, 0}};
        while(!queue.isEmpty()) {
            int[] cur = queue.poll();
            for(int[] direction: directions) {
                int x = cur[0] + direction[0];
                int y = cur[1] + direction[1];
                if(x < 0 || x >= grid.length || y < 0 || y >= grid[x].length || grid[x][y] == 2 || grid[x][y] == 0) {
                    res = Math.max(res, cur[2]);
                } else {
                    grid[x][y] = 2;
                    queue.offer(new int[] {x, y, cur[2] + 1});
                }
            }
        }
        for(int i = 0; i < grid.length; i ++) {
            for(int j = 0; j < grid[i].length; j ++) {
                if(grid[i][j] == 1) {
                    return -1;
                }
            }
        }
        return res;
    }
}
```

时间复杂度分析：

- 两个for循环的时间复杂度为`O(n*m)`，n和m分别为grid的长和宽
- bfs最坏情况下会遍历每一个节点，时间复杂度为O(n*m)。

综上时间复杂度为`O(n*m)`

空间复杂度分析：

- queue最坏情况下会包含所有节点，空间复杂度为`O(n*m)`

综上空间复杂度为`O(n*m)`

# [875. Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas/)

Koko loves to eat bananas. There are `n` piles of bananas, the `ith` pile has `piles[i]` bananas. The guards have gone and will come back in `h` hours.

Koko can decide her bananas-per-hour eating speed of `k`. Each hour, she chooses some pile of bananas and eats `k` bananas from that pile. If the pile has less than `k` bananas, she eats all of them instead and will not eat any more bananas during this hour.

Koko likes to eat slowly but still wants to finish eating all the bananas before the guards return.

Return *the minimum integer* `k` *such that she can eat all the bananas within* `h` *hours*.

 

**Example 1:**

```
Input: piles = [3,6,7,11], h = 8
Output: 4
```

**Example 2:**

```
Input: piles = [30,11,23,4,20], h = 5
Output: 30
```

**Example 3:**

```
Input: piles = [30,11,23,4,20], h = 6
Output: 23
```

可以使用二分查找不断缩小符合要求的最小速度，直到找到最小的速度

代码如下：

```java
class Solution {
    public int minEatingSpeed(int[] piles, int h) {
        int slow = 1;
        int fast = 1000000000;
        int result = -1;
        while(slow <= fast) {
            int mid = slow + (fast - slow) / 2;
            if(check(mid, piles, h)) {
                result = mid;
                fast = mid - 1;
            } else {
                slow = mid + 1;
            }
        }
        return result;
    }

    private boolean check(int speed, int[] piles, int h) {
        long hours = 0;
        for(Integer pile: piles) {
            hours += pile % speed == 0 ? pile / speed : pile / speed + 1;
        }
        if(hours <= h) {
            return true;
        }
        return false;
    }
}
```

因为每一堆香蕉的最大数量为10^9，所以可以设置最大速度为10^9，最小速度为1。然后就可以进行二分查找，不断缩小速度。

值得注意的是在`check()方法`中用于记录用时的变量`hours`需要设置为long，不然会有溢出的风险。

时间复杂度分析：

- piles的长度为n，每次二分查找都需要遍历一次piles数组，复杂度为`O(n)`
- 二分查找的范围是1-10^9，所以复杂度为`O(10^9)`
- 综上时间复杂度为`O(10^9n)`

空间复杂度为`O(1)`





# [962. Maximum Width Ramp](https://leetcode.com/problems/maximum-width-ramp/)

A **ramp** in an integer array `nums` is a pair `(i, j)` for which `i < j` and `nums[i] <= nums[j]`. The **width** of such a ramp is `j - i`.

Given an integer array `nums`, return *the maximum width of a **ramp** in* `nums`. If there is no **ramp** in `nums`, return `0`.

**Example 1:**

```
Input: nums = [6,0,8,2,1,5]
Output: 4
Explanation: The maximum width ramp is achieved at (i, j) = (1, 5): nums[1] = 0 and nums[5] = 5.
```

**Example 2:**

```
Input: nums = [9,8,1,0,1,9,4,0,4,1]
Output: 7
Explanation: The maximum width ramp is achieved at (i, j) = (2, 9): nums[2] = 1 and nums[9] = 1.
```

这道题其实简单来说就是找到和每个元素对应的不小于它的最远的元素，并且得到距离的最大值。

具体的做法是维护一个单调递减栈，从左向右遍历。再从右向左遍历，如果栈顶的元素小于等于遍历到的元素，就出栈并记录最大距离，知道栈空。

代码如下：

```java
class Solution {
    public int maxWidthRamp(int[] nums) {
        Deque<Integer> stack = new LinkedList<>();
        for(int i = 0; i < nums.length; i ++) {
            if(stack.isEmpty() || nums[stack.peekLast()] > nums[i]) {
                stack.offerLast(i);
            }
        }

        int res = 0;
        for(int i = nums.length - 1; i >= 0; i --) {
            while(!stack.isEmpty() && nums[stack.peekLast()] <= nums[i]) {
                res = Math.max(res, i - stack.pollLast());
            }
        } 
        return res;
    }
}
```

时间复杂度分析：

- 维护单调栈需要遍历每个元素， `O(n)`。
- 第二次从右向左遍历元素最坏情况下需要遍历全部元素，`O(n)`。

时间复杂度为`O(n)`。

空间复杂度分析:

- 最坏情况`stack`需要记录全部元素， `O(n)`。

空间复杂度为`O(n)`。

# [2406. Divide Intervals Into Minimum Number of Groups](https://leetcode.com/problems/divide-intervals-into-minimum-number-of-groups/)

You are given a 2D integer array `intervals` where `intervals[i] = [lefti, righti]` represents the **inclusive** interval `[lefti, righti]`.

You have to divide the intervals into one or more **groups** such that each interval is in **exactly** one group, and no two intervals that are in the same group **intersect** each other.

Return *the **minimum** number of groups you need to make*.

Two intervals **intersect** if there is at least one common number between them. For example, the intervals `[1, 5]` and `[5, 8]` intersect.

**Example 1:**

```
Input: intervals = [[5,10],[6,8],[1,5],[2,3],[1,10]]
Output: 3
Explanation: We can divide the intervals into the following groups:
- Group 1: [1, 5], [6, 8].
- Group 2: [2, 3], [5, 10].
- Group 3: [1, 10].
It can be proven that it is not possible to divide the intervals into fewer than 3 groups.
```

**Example 2:**

```
Input: intervals = [[1,3],[5,6],[8,10],[11,13]]
Output: 1
Explanation: None of the intervals overlap, so we can put all of them in one group.
```

这道题可以用时间线和事件的思路去解决，不必拘泥于时间对，而是把每一个时间对拆开，将其视为开始时间和结束时间。

同时使用一个最小堆对时间进行排序，当存在开始时间和结束时间相同时**先处理开始时间**。维护一个变量记录同时存在的事件的最大数量，即为答案。

代码如下：

```java
class Solution {
    public int minGroups(int[][] intervals) {
        Queue<int[]> events = new PriorityQueue<>((a, b) -> {
            if(a[0] == b[0]) {
                return b[1] - a[1];
            }
            return a[0] - b[0];
        });
        for(int[] interval: intervals) {
            events.offer(new int[] {interval[0], 1}); // 1 is starting
            events.offer(new int[] {interval[1], -1}); // 0 is ending
        }

        int cur = 0;
        int res = 0;
        while(!events.isEmpty()) {
            int[] curEvent = events.poll();
            cur += curEvent[1];
            res = Math.max(cur, res);
        }
        return res;
    }
}
```

时间复杂度分析：

- 假设intervals有n个事件，那么一共有2n个时间点，插入堆的复杂度为`O(log2n)`,综合复杂度为`O(2nlog(2n)) = O(nlogn)`。

空间复杂度分析：

- 堆需要`O(2n) = O(n)`的空间

# [632. Smallest Range Covering Elements from K Lists](https://leetcode.com/problems/smallest-range-covering-elements-from-k-lists/)

You have `k` lists of sorted integers in **non-decreasing order**. Find the **smallest** range that includes at least one number from each of the `k` lists.

We define the range `[a, b]` is smaller than range `[c, d]` if `b - a < d - c` **or** `a < c` if `b - a == d - c`.



**Example 1:**

```
Input: nums = [[4,10,15,24,26],[0,9,12,20],[5,18,22,30]]
Output: [20,24]
Explanation: 
List 1: [4, 10, 15, 24,26], 24 is in range [20,24].
List 2: [0, 9, 12, 20], 20 is in range [20,24].
List 3: [5, 18, 22, 30], 22 is in range [20,24].
```

**Example 2:**

```
Input: nums = [[1,2,3],[1,2,3],[1,2,3]]
Output: [1,1]
```

涉及到“最大”“最小”问题时，往往需要考虑使用heap，因为这类问题需要频繁地获得最大最小值，而堆可以实现在`O(1)`的复杂度下得到最值，从而降低时间复杂度。

每一轮将一列元素存入heap，将最小值移出，并且动态的更新最大最小值，并记录range，再将移出元素的下一个元素加入heap，直到heap中的元素数量小于nums中数组的数量，最终获得的最小range即为答案。

```java
class Solution {
    public int[] smallestRange(List<List<Integer>> nums) {
        Queue<int[]> pq = new PriorityQueue<>((a, b) -> {
            return a[0] - b[0];
        });
        int max = Integer.MIN_VALUE;
        for(int i = 0; i < nums.size(); i ++) {
            pq.offer(new int[] {nums.get(i).get(0), i, 0});
            max = Math.max(max, nums.get(i).get(0));
        }
        int[] res = new int[2];
        int range = Integer.MAX_VALUE;
        while(pq.size() == nums.size()) {
            int[] cur = pq.poll();
            int min = cur[0];
            if(range > max - min) {
                range = max - min;
                res[0] = min;
                res[1] = max;
            }

            if(cur[2] < nums.get(cur[1]).size() - 1) {
                pq.offer(new int[] {nums.get(cur[1]).get(cur[2] + 1), cur[1], cur[2] + 1});
                max = Math.max(max, nums.get(cur[1]).get(cur[2] + 1));
            }
        }
        return res;
    }
}
```

```
例一的每一轮处理过程如下

heap 0 4 5
max 5
min 0
range 5

heap 4 5 9
max 9
min 4
range 5

heap 5 9 10
max 10
min 5
range 5

heap 9 10 18
max 18
min 9
range 9

heap 10 12 18
max 18
min 10
range 8

heap 12 15 18
max 18
min 12 
range 6

heap 15 18 20
min 15
max 20
range 5

heap 18 20 24
min 18
max 24
range 6

heap 20 22 24
min 20
max 24
range 4
```

时间复杂度分析：

- 设nums中共有k个数组，设共有N个元素，最坏情况下heap需要遍历每一个元素，每次插入和删除元素的复杂度为`O(logk)`，时间复杂度为`O(Nlogk)`。

空间复杂度分析：

- heap中始终存在k个元素，空间复杂度为`O(k)`

# [1942. The Number of the Smallest Unoccupied Chair](https://leetcode.com/problems/the-number-of-the-smallest-unoccupied-chair/)

There is a party where `n` friends numbered from `0` to `n - 1` are attending. There is an **infinite** number of chairs in this party that are numbered from `0` to `infinity`. When a friend arrives at the party, they sit on the unoccupied chair with the **smallest number**.

- For example, if chairs `0`, `1`, and `5` are occupied when a friend comes, they will sit on chair number `2`.

When a friend leaves the party, their chair becomes unoccupied at the moment they leave. If another friend arrives at that same moment, they can sit in that chair.

You are given a **0-indexed** 2D integer array `times` where `times[i] = [arrivali, leavingi]`, indicating the arrival and leaving times of the `ith` friend respectively, and an integer `targetFriend`. All arrival times are **distinct**.

Return *the **chair number** that the friend numbered* `targetFriend` *will sit on*.

**Example 1:**

```
Input: times = [[1,4],[2,3],[4,6]], targetFriend = 1
Output: 1
Explanation: 
- Friend 0 arrives at time 1 and sits on chair 0.
- Friend 1 arrives at time 2 and sits on chair 1.
- Friend 1 leaves at time 3 and chair 1 becomes empty.
- Friend 0 leaves at time 4 and chair 0 becomes empty.
- Friend 2 arrives at time 4 and sits on chair 0.
Since friend 1 sat on chair 1, we return 1.
```

**Example 2:**

```
Input: times = [[3,10],[1,5],[2,6]], targetFriend = 0
Output: 2
Explanation: 
- Friend 1 arrives at time 1 and sits on chair 0.
- Friend 2 arrives at time 2 and sits on chair 1.
- Friend 0 arrives at time 3 and sits on chair 2.
- Friend 1 leaves at time 5 and chair 0 becomes empty.
- Friend 2 leaves at time 6 and chair 1 becomes empty.
- Friend 0 leaves at time 10 and chair 2 becomes empty.
Since friend 0 sat on chair 2, we return 2.
```

- 这个问题可以考虑将每个人的时间拆开，分成开始时间和结束时间。将拆开后的时间以数组的形式存入小顶堆中，arr[0]是时间，arr[1]用来记录是第几个人，arr[2]用来记录是到达时间还是离开时间。以arr[0]为依据排序。
- 同时，一个优先队列来记录当前时刻下空的椅子，这样总是可以得到最小的可以利用的椅子。
- 用一个Map来记录当前时刻下已经被占用的椅子，key为人的序号，value是椅子序号。
- 具体的流程是，按发生顺序依次遍历每一个时间。如果为到达时间，就从availableChairs中分配一把椅子，并存入occupiedChairs。并且检查是否是targetFriend；如果为结束时间，就将对应的occupiedChairs中的椅子放回avaliableChairs。

```java
class Solution {
    public int smallestChair(int[][] times, int targetFriend) {
      	
        Queue<int[]> events = new PriorityQueue<>((a, b) -> {
            if(a[0] == b[0]) {
                return a[2] - b[2];
            }
            return a[0] - b[0];
        });

        Queue<Integer> avaliableChairs = new PriorityQueue<>();
        Map<Integer, Integer> occupiedChairs = new HashMap<>();

        for(int i = 0; i < times.length; i ++) {
            events.offer(new int[] {times[i][0], i, 1}); // arrival
            events.offer(new int[] {times[i][1], i, 0}); // leaving
        }

        for(int i = 0; i < times.length; i ++) {
            avaliableChairs.offer(i);
        }

        while(!events.isEmpty()) {
            int[] cur = events.poll();
            int time = cur[0];
            int number = cur[1];
            int event = cur[2];

            if(event == 1) { // arrival
                int chair = avaliableChairs.poll();
                if(number == targetFriend) {
                    return chair;
                }
                occupiedChairs.put(number, chair);
            } else { // leaving
                int chair = occupiedChairs.get(number);
                avaliableChairs.offer(chair);
            }
        }
        return -1;
    }
}
```

时间复杂度分析：

- 设一共有n个人，events共插入2n次，每次插入的时间复杂度为`O(logn)`，复杂度为`O(2nlog2n) = O(nlogn)`。
- 将所有椅子加入avaliableChairs，时间复杂度为`O(nlogn)`。
- 依次遍历每个event，最坏情况下每次都要取一个新的椅子，从avaliableChairs取椅子复杂度为`O(logn)`, 插入occupiedChairs复杂度为(O(1))，总复杂度为`2n(O(logn) + O(1)) = O(nlogn)`。

综上时间复杂度为`O(nlogn)`。

空间复杂度分析：

- events需要`O(2n)`
- avaliableChair最多需要`O(n)`
- occupiedChair最多需要`O(n)`

综上空间复杂度为`O(n)`。



# [**632. Smallest Range Covering Elements from K Lists**](https://leetcode.com/problems/smallest-range-covering-elements-from-k-lists/)

You have `k` lists of sorted integers in **non-decreasing order**. Find the **smallest** range that includes at least one number from each of the `k` lists.

We define the range `[a, b]` is smaller than range `[c, d]` if `b - a < d - c` **or** `a < c` if `b - a == d - c`.

**Example 1:**

```plain
Input: nums = [[4,10,15,24,26],[0,9,12,20],[5,18,22,30]]
Output: [20,24]
Explanation: 
List 1: [4, 10, 15, 24,26], 24 is in range [20,24].
List 2: [0, 9, 12, 20], 20 is in range [20,24].
List 3: [5, 18, 22, 30], 22 is in range [20,24].
```

**Example 2:**

```plain
Input: nums = [[1,2,3],[1,2,3],[1,2,3]]
Output: [1,1]
```



**Constraints:**

- `nums.length == k`
- `1 <= k <= 3500`
- `1 <= nums[i].length <= 50`
- `-10^5 <= nums[i][j] <= 10^5`
- `nums[i]` is sorted in **non-decreasing** order.



这个题的核心在于要保证选取的范围至少包含每个子数组中一个元素。考虑到这一点可以在遍历时使用一个容器，并始终保证容器中恰好有**k**个元素(k为子数组的数量)。同时在每次遍历时需要获得最大值和最小值来确定范围，所以理所应当使用一个最小堆。

- 初始化：先把每个子数组的第一个元素放入堆中并记录最大值。
- 遍历：一个while循环，每次取出一个元素（最小值），更新最小值，从而更新最小范围。如果最小范围变小就记录当前的状态。
- 在遍历的最后把取出的最小元素的下一个元素（如果有）加入堆中，并更新最大值。
- 循环结束时记录的状态即为答案。

代码如下：

```java
class Solution {
    public int[] smallestRange(List<List<Integer>> nums) {
        Queue<int[]> q = new PriorityQueue<>((a, b) -> {
            return a[0] - b[0];
        }); 
        int max = Integer.MIN_VALUE;
        for(int i = 0; i < nums.size(); i ++) {
            q.offer(new int[] {nums.get(i).get(0), i, 0});
          // arr[0] for value. arr[1] for i of nums[i], arr[2] for j of nums[i][j];
            max = Math.max(max, nums.get(i).get(0));
        }
        int len = Integer.MAX_VALUE;
        int[] res = new int[2];
        while(q.size() == nums.size()) {
            int[] cur = q.poll();
            int min = cur[0];
            int i = cur[1];
            int j = cur[2];
            if(len > max - min) {
                res[0] = min;
                res[1] = max;
                len = max - min;
            }
            if(j != nums.get(i).size() - 1) {
                q.offer(new int[] {nums.get(i).get(j + 1), i, j + 1});
                max = Math.max(max, nums.get(i).get(j + 1));
            }
        }
        return res;
    }
}
```

时间复杂度分析：

- 设共有N个元素， 最坏情况下需要遍历所有元素。
- 每个元素只进行一次进出堆的操作，堆中元素的数量始终为K，进入堆时维护最小堆的复杂度为`O(logK)`，出堆时的复杂度为`O(1)`， 维护堆的时间复杂度为`O(logK)`。共有N个元素，重复N次，时间复杂度为`O(N*(O(logK) + O(logK) + O(1))) = O(NlogK)`。

空间复杂度分析：

- 堆中最多有K个元素，K为nums的size。空间复杂度为`O(K)`。

# [2406. Divide Intervals Into Minimum Number of Groups](https://leetcode.com/problems/divide-intervals-into-minimum-number-of-groups/)

You are given a 2D integer array `intervals` where `intervals[i] = [lefti, righti]` represents the **inclusive** interval `[lefti, righti]`.

You have to divide the intervals into one or more **groups** such that each interval is in **exactly** one group, and no two intervals that are in the same group **intersect** each other.

Return *the* ***minimum\*** *number of groups you need to make*.

Two intervals **intersect** if there is at least one common number between them. For example, the intervals `[1, 5]` and `[5, 8]` intersect.



**Example 1:**

```plain
Input: intervals = [[5,10],[6,8],[1,5],[2,3],[1,10]]
Output: 3
Explanation: We can divide the intervals into the following groups:
- Group 1: [1, 5], [6, 8].
- Group 2: [2, 3], [5, 10].
- Group 3: [1, 10].
It can be proven that it is not possible to divide the intervals into fewer than 3 groups.
```

**Example 2:**

```plain
Input: intervals = [[1,3],[5,6],[8,10],[11,13]]
Output: 1
Explanation: None of the intervals overlap, so we can put all of them in one group.
```



**Constraints:**

- `1 <= intervals.length <= 10^5`
- `intervals[i].length == 2`
- `1 <= lefti <= righti <= 10^6`



可以画一个时间轴，把intervals中的每个元素当成一个时间段画在时间轴上，重叠时间段最多的时间点对应的重叠的数量就是最小需要的分组数。

- 把每个时间段拆成开始时间和结束时间，放入最小堆。
- 依次遍历堆中每一个元素，在遍历时，记录同时存在的时间段的最大数量即为答案。

代码如下：

```java
class Solution {
    public int minGroups(int[][] intervals) {
        Queue<int[]> q = new PriorityQueue<>((a, b) -> {
            return a[0] - b[0];
        });
        for(int[] arr: intervals) {
            q.offer(new int[] {arr[0], 0});
            q.offer(new int[] {arr[1], 1});
        }
        int overlapped = 0;
        int mostOverlapped = 0;
        while(!q.isEmpty()) {
            int[] cur = q.poll();
            if(cur[1] == 0) {
                mostOverlapped = Math.max(mostOverlapped, ++ overlapped);
            } else {
                overlapped --;
            }
        }
        return mostOverlapped;
    }
}
```

时间复杂度分析：

- 设intervals数组共有N个元素，则共有2N个元素需要进栈和出栈各一次。时间复杂度为`O(2N*2*O(log2N)) = O(NlogN)`。

空间复杂度分析:

- 堆中最多存在2N个元素，空间复杂度为`O(2N) = O(N)`。



# [962. Maximum Width Ramp](https://leetcode.com/problems/maximum-width-ramp/)



A **ramp** in an integer array `nums` is a pair `(i, j)` for which `i < j` and `nums[i] <= nums[j]`. The **width** of such a ramp is `j - i`.

Given an integer array `nums`, return *the maximum width of a* ***ramp\*** *in* `nums`. If there is no **ramp** in `nums`, return `0`.



**Example 1:**

```plain
Input: nums = [6,0,8,2,1,5]
Output: 4
Explanation: The maximum width ramp is achieved at (i, j) = (1, 5): nums[1] = 0 and nums[5] = 5.
```

**Example 2:**

```plain
Input: nums = [9,8,1,0,1,9,4,0,4,1]
Output: 7
Explanation: The maximum width ramp is achieved at (i, j) = (2, 9): nums[2] = 1 and nums[9] = 1.
```

 

**Constraints:**

- `2 <= nums.length <= 5 * 10^4`
- `0 <= nums[i] <= 5 * 10^4`



第一种方法是把所有元素按从小到大排序，再遍历一遍，遍历过程中维护最小下标并更新最长ramp的距离。

```java
class Solution {
    public int maxWidthRamp(int[] nums) {
        List<int[]> arr = new ArrayList<>();
        for(int i = 0; i < nums.length; i ++) {
            arr.add(new int[] {nums[i], i});
        }
        Collections.sort(arr, (a, b) -> {
            return a[0] - b[0];
        });
        int min = arr.get(0)[1];
        int res = 0;
        for(int[] num: arr) {
            min = Math.min(min, num[1]);
            res = Math.max(num[1] - min, res);
        }
        return res;
    }
}
```

时间复杂度分析：

- 共有N个元素，排序时间复杂度为`O(NlogN)`。
- 遍历一遍复杂度为`O(N)`。
- 综合时间复杂度为`O(N + NlogN) = O(NlogN)`。



第二种方法较为巧妙，采用一个单调栈（单调递减）记录nums中的元素，再倒着遍历，当栈顶元素不大于当前元素时出栈并更新最大值，直到栈顶元素大于当前元素或栈为空。

单调栈的作用在这里其实是一个按倒序排序的”set“， 效果是记录了每一个可能入栈的元素的最左边位置（如果有多个相同的元素，只记录最左边的那个），这样就满足了最大ramp的要求。

```java
class Solution {
    public int maxWidthRamp(int[] nums) {
        Deque<Integer> stack = new LinkedList<>();
        for(int i = 0; i < nums.length; i ++) {
            if(stack.isEmpty() || nums[stack.peekLast()] > nums[i]) {
                stack.offerLast(i);
            }
        }   
        int res = 0;
        for(int i = nums.length - 1; i >= 0; i --) {
            while(!stack.isEmpty() && nums[stack.peekLast()] <= nums[i]) {
                res = Math.max(res, i - stack.pollLast());
            }
        }
        return res;
    }
}
```

时间复杂度分析：

- 构建单调栈的过程需要遍历整个数组，复杂度为`O(N)`。
- 第二次遍历最坏情况下需要遍历整个数组，复杂度为`O(N)`。
- 综合复杂度为`O(N)`。



# [3152. Special Array II](https://leetcode.com/problems/special-array-ii/)



An array is considered **special** if every pair of its adjacent elements contains two numbers with different parity.

You are given an array of integer `nums` and a 2D integer matrix `queries`, where for `queries[i] = [fromi, toi]` your task is to check that subarray `nums[fromi..toi]` is **special** or not.

Return an array of booleans `answer` such that `answer[i]` is `true` if `nums[fromi..toi]` is special.



**Example 1:**

**Input:** nums = [3,4,1,2,6], queries = [[0,4]]

**Output:** [false]

**Explanation:**

The subarray is `[3,4,1,2,6]`. 2 and 6 are both even.

**Example 2:**

**Input:** nums = [4,3,1,6], queries = [[0,2],[2,3]]

**Output:** [false,true]

**Explanation:**

1. The subarray is `[4,3,1]`. 3 and 1 are both odd. So the answer to this query is `false`.
2. The subarray is `[1,6]`. There is only one pair: `(1,6)` and it contains numbers with different parity. So the answer to this query is `true`.



**Constraints:**

- `1 <= nums.length <= 10^5`
- `1 <= nums[i] <= 10^5`
- `1 <= queries.length <= 10^5`
- `queries[i].length == 2`
- `0 <= queries[i][0] <= queries[i][1] <= nums.length - 1`



第一种方法：

- 先遍历数组，如果找到都是偶数（或奇数）的pair，将较小的下标存入list
- 对每一个query
  - 进行二分查找
  - 尝试在list中找到query[0] ～ query[1]范围内的值
  - 找到结果为false，没找到为true

```java
class Solution {
    public boolean[] isArraySpecial(int[] nums, int[][] queries) {
        List<Integer> list = new ArrayList<>();
        for(int i = 0; i < nums.length - 1; i ++) {
            if(((nums[i] & 1) ^ (nums[i + 1] & 1)) == 0) {
                list.add(i);
            }
        }
        boolean[] res = new boolean[queries.length];
        for(int i = 0; i < queries.length; i ++) {
            int left = queries[i][0], right = queries[i][1] - 1;
            int l = 0, r = list.size() - 1;
            boolean has = false;
            while(l <= r) {
                int mid = l + (r - l) / 2;
                if(left <= list.get(mid) && list.get(mid) <= right) {
                    has = true;
                    break;
                } else if(list.get(mid) > right) {
                    r = mid - 1;
                } else if(list.get(mid) < left) {
                    l = mid + 1;
                }
            }
            res[i] = !has;
        }
        return res;
    }
}
```

时间复杂度分析：

- 设共有N个元素，遍历一遍复杂度为`O(N)`
- 设共有Q个query，对每一个query进行二分查找，复杂度为`O(QlogN)`
- 综合复杂度为`O(N + QlogN)`

空间复杂度分析：

- 需要`O(Q)`的空间来存储同奇偶对



还有一种效率更高的解法：

- 遍历数组，初始化prefixSum，`prefixSum[0] = 0`
- 如果当前元素和上一个元素同为奇偶，`prefixSum[i] = prefixSum[i - 1] + 1`

- 对每一个query，如果`query[1] != query[0]`说明范围内有相邻的奇数或偶数

```java
class Solution {
    public boolean[] isArraySpecial(int[] nums, int[][] queries) {
        boolean[] ans=new boolean[queries.length];
        int[] prefixSum=new int[nums.length];
        for(int i=1;i<nums.length;i++){
            prefixSum[i]=prefixSum[i-1];
            if(nums[i]%2==nums[i-1]%2){
                prefixSum[i]+=1;
            }
        }
        int count=0;
        for(int[] q:queries){
            int from=q[0];
            int to=q[1];
            ans[count]=prefixSum[to]-prefixSum[from]==0 ? true:false;
            count++;
        }
        return ans;
    }
}
```

时间复杂度分析：

- 遍历一遍数组`O(N)`
- 遍历Query`O(Q)`
- 综合复杂度`O(N + Q)`

空间复杂度分析：

- prefixSum需要额外的`O(N)`空间



# [2779. Maximum Beauty of an Array After Applying Operation](https://leetcode.com/problems/maximum-beauty-of-an-array-after-applying-operation/)



You are given a **0-indexed** array `nums` and a **non-negative** integer `k`.

In one operation, you can do the following:

- Choose an index `i` that **hasn't been chosen before** from the range `[0, nums.length - 1]`.
- Replace `nums[i]` with any integer from the range `[nums[i] - k, nums[i] + k]`.

The **beauty** of the array is the length of the longest subsequence consisting of equal elements.

Return *the* ***maximum\*** *possible beauty of the array* `nums` *after applying the operation any number of times.*

**Note** that you can apply the operation to each index **only once**.

A **subsequence** of an array is a new array generated from the original array by deleting some elements (possibly none) without changing the order of the remaining elements.



**Example 1:**

```plain
Input: nums = [4,6,1,2], k = 2
Output: 3
Explanation: In this example, we apply the following operations:
- Choose index 1, replace it with 4 (from range [4,8]), nums = [4,4,1,2].
- Choose index 3, replace it with 4 (from range [0,4]), nums = [4,4,1,4].
After the applied operations, the beauty of the array nums is 3 (subsequence consisting of indices 0, 1, and 3).
It can be proven that 3 is the maximum possible length we can achieve.
```

**Example 2:**

```plain
Input: nums = [1,1,1,1], k = 10
Output: 4
Explanation: In this example we don't have to apply any operations.
The beauty of the array nums is 4 (whole array).
```



- **Constraints:**
  - `1 <= nums.length <= 10^5`
  - `0 <= nums[i], k <= 10^5`



这个题类似于扎气球问题[452. Minimum Number of Arrows to Burst Balloons](https://leetcode.cn/problems/minimum-number-of-arrows-to-burst-balloons/)， 本质上都是区间问题。但这个问题变成了如何用一支箭扎破最多的气球（区间）。

虽然是子序列问题，但实际上最终的目的是找重叠数量最多的区间，所以排序是可以的。

思路是先对数组进行排序，再遍历每个区间的末尾端点， 用二分查找尝试找到第一个大于当前末尾端点的起始端点，记录此时的区间数量，每次遍历都进行更新，最终得到的就是最大值。

代码如下：

```java
class Solution {
    public int maximumBeauty(int[] nums, int k) {
        int n = nums.length;
        Arrays.sort(nums);
        int res = 1;
        for(int i = 0; i < n; i ++) {
            int left = i + 1, right = n - 1;
            int first = 0;
            while(left <= right) {
                int mid = left + (right - left) / 2;
                if(nums[mid] - k <= nums[i] + k) {
                    first = mid;
                    left = mid + 1;
                } else {
                    right = mid - 1;
                }
            }
            res = Math.max(res, first - i + 1);
        }
        return res;
    }
}
```

时间复杂度：

- 排序`O(NlogN)`
- 外层循环`O(N)`，内层循环是一个二分查找`O(logn)`,综合为`O(NlogN)`
- 综合复杂度`O(NlogN)`

空间复杂度：`O(1)`



# [2981. Find Longest Special Substring That Occurs Thrice I](https://leetcode.com/problems/find-longest-special-substring-that-occurs-thrice-i/)



You are given a string `s` that consists of lowercase English letters.

A string is called **special** if it is made up of only a single character. For example, the string `"abc"` is not special, whereas the strings `"ddd"`, `"zz"`, and `"f"` are special.

Return *the length of the* ***longest special substring\*** *of* `s` *which occurs* ***at least thrice\***, *or* `-1` *if no special substring occurs at least thrice*.

A **substring** is a contiguous **non-empty** sequence of characters within a string.



**Example 1:**

```plain
Input: s = "aaaa"
Output: 2
Explanation: The longest special substring which occurs thrice is "aa": substrings "aaaa", "aaaa", and "aaaa".
It can be shown that the maximum length achievable is 2.
```

**Example 2:**

```plain
Input: s = "abcdef"
Output: -1
Explanation: There exists no special substring which occurs at least thrice. Hence return -1.
```

**Example 3:**

```plain
Input: s = "abcaba"
Output: 1
Explanation: The longest special substring which occurs thrice is "a": substrings "abcaba", "abcaba", and "abcaba".
It can be shown that the maximum length achievable is 1.
```



**Constraints:**

- `3 <= s.length <= 50`
- `s` consists of only lowercase English letters.



最简单直接的想法是对每个字母，从长度为1到长度为n遍历查找。但是时间复杂度过高。

实际上观察一下最长子串的规律就不难发现只需要记录每个字母的最长子串和次长子串的长度和数量：

- 最长子串数量 >= 3 --- 最长子串长度
- 最长子串数量 == 2 --- 最长子串长度 - 1
- 最长子串数量 == 1
  - 次长子串长度 == 最长子串长度 - 1 --- 最长子串长度 - 1
  - 次长子串长度 < 最长子串长度 - 1 --- 最长子串长度 - 2

```java
class Solution {
    public int maximumLength(String s) {
        int n = s.length();
        int[][] map = new int[26][4]; 
      // [secondLongest, numberOfSecondLongest, longest, numberOfLongest]
        int start = 0;
        for(int end = 0; end < n; end ++) {
            // the goal is to find the longest special substring a time
            if(s.charAt(start) == s.charAt(end)) {
                if(end != n - 1) {
                    continue;
                } else {
                    update(map, s, start, end);
                }
            } else {
                update(map, s, start, end - 1);
                start = end;
            }
        }
        if(s.charAt(n - 1) != s.charAt(n - 2)) {
            update(map, s, start, start);
        }
        int res = 0;
        for(int[] arr: map) {
            if(arr[0] + arr[2] >= 3) { // make sure corner case: 1 1 2 1
                res = Math.max(res, 1);
            }
            if(arr[3] >= 3) { // num of longest >= 3
                res = Math.max(res, arr[2]);
            } else if(arr[3] == 2) { // num of longest = 2
                res = Math.max(res, arr[2] - 1);
            } else if(arr[3] == 1) { // num of longest = 1
                if(arr[0] == arr[2] - 1 && arr[1] >= 1) { 
                  // when 2ndLongest = longest - 1 and num of 2ndLongest  >= 1
                    res = Math.max(res, arr[2] - 1);
                } else {
                    res = Math.max(res, arr[2] - 2);
                }
            }
        }
    return res == 0 ? -1 : res;
    }

    private void update(int[][] map, String s, int start, int end) {
        int cur = s.charAt(start) - 'a';
        int len = end - start + 1;
        int l1 = map[cur][0], l1l = map[cur][1], l2 = map[cur][2], l2l = map[cur][3];
        if(l2 == 0 && l1 == 0) { // if no 2ndLongest and longest, update longest
            map[cur][2] = len;
            map[cur][3] = 1;
        } else if(l2 < len) { // if longer than longest, update 2ndLongest and longest
            map[cur][0] = l2;
            map[cur][1] = l2l;
            map[cur][2] = len;
            map[cur][3] = 1;
        } else if(l2 == len) { // if equals to longest
            map[cur][3] ++;
        } else if(l1 < len) { // if shorter than longest and longer than 2ndLongest, update 2nd longest
            map[cur][0] = len;
            map[cur][1] = 1;
        } else if(l1 == len) { // if equals to 2ndLongest
            map[cur][1] ++;
        }
    }
}
```

时间复杂度：

- 遍历字符串`O(n)`
- update方法复杂度为`O(1)`
- 遍历map复杂度为`O(26 * 4) == O(1)`
- 综合复杂度`O(N)`

空间复杂度：

- Map需要额外的`O(26 * 4) == O(1)`



# [3381. Maximum Subarray Sum With Length Divisible by K](https://leetcode.com/problems/maximum-subarray-sum-with-length-divisible-by-k/)



You are given an array of integers `nums` and an integer `k`.

Return the **maximum** sum of a subarray of `nums`, such that the size of the subarray is **divisible** by `k`.



**Example 1:**

**Input:** nums = [1,2], k = 1

**Output:** 3

**Explanation:**

The subarray `[1, 2]` with sum 3 has length equal to 2 which is divisible by 1.

**Example 2:**

**Input:** nums = [-1,-2,-3,-4,-5], k = 4

**Output:** -10

**Explanation:**

The maximum sum subarray is `[-1, -2, -3, -4]` which has length equal to 4 which is divisible by 4.

**Example 3:**

**Input:** nums = [-5,1,2,-3,4], k = 2

**Output:** 4

**Explanation:**

The maximum sum subarray is `[1, 2, -3, 4]` which has length equal to 4 which is divisible by 2.



**Constraints:**

- `1 <= k <= nums.length <= 2 * 10^5`
- `-10^9 <= nums[i] <= 10^9`



对于k=1的情况，可以使用kadane算法在线性的时间复杂度下得到最大子数组和。

如果K大于1，可以将[i, i + k], [i + k, i + 2 * k]....每个子数组当成一个元素，这样就可以使用kadane算法了。

代码如下：

```java
class Solution {
    public long maxSubarraySum(int[] nums, int k) {
        if (k == 1) {
            return kadane(nums);
        }
        long res = Long.MIN_VALUE;
        long[] prefix = new long[nums.length + 1];
        for (int i = 1; i <= nums.length; i++) {
            prefix[i] = prefix[i - 1] + nums[i - 1];
        }
        for (int i = 0; i < k; i++) {
            int len = (nums.length - i) / k;
            long[] tmp = new long[len];
            for (int j = 0; j < len; j++) {
                int start = i + j * k, end = start + k;
                tmp[j] = prefix[end] - prefix[start];
            }
            res = Math.max(res, kadane(tmp));
        }
        return res;
    }

    static long kadane(long[] nums) {
        long res = Long.MIN_VALUE;
        long curr = 0;

        for (int i = 0; i < nums.length; i++) {
            if (curr + nums[i] > nums[i]) {
                curr += nums[i];
            } else {
                curr = nums[i];
            }
            if (curr > res) {
                res = curr;
            }
        }
        return res;
    }
}
```

时间复杂度：

- 计算前缀和`O(N)`
- 外层循环复杂度为`O(K)`，内层循环复杂度为`O(N/K)`
- Kadane算法复杂度为`O(N/K)`
- 综合复杂度为`O(K * (N / K + N / K)) = O(N)`

空间复杂度：

- prefix需要额外的`O(N)`
- 对于临时tmp数组最大需要`O(N/K)`

- 综合空间复杂度为`O(N)`



# [3376. Minimum T3376. Minimum Time to Break Locks I](https://leetcode.com/problems/minimum-time-to-break-locks-i/)



Bob is stuck in a dungeon and must break `n` locks, each requiring some amount of **energy** to break. The required energy for each lock is stored in an array called `strength` where `strength[i]` indicates the energy needed to break the `ith` lock.

To break a lock, Bob uses a sword with the following characteristics:

- The initial energy of the sword is 0.
- The initial factor `X` by which the energy of the sword increases is 1.
- Every minute, the energy of the sword increases by the current factor `X`.
- To break the `ith` lock, the energy of the sword must reach **at least** `strength[i]`.
- After breaking a lock, the energy of the sword resets to 0, and the factor `X` increases by a given value `K`.

Your task is to determine the **minimum** time in minutes required for Bob to break all `n` locks and escape the dungeon.

Return the **minimum** time required for Bob to break all `n` locks.



**Example 1:**

**Input:** strength = [3,4,1], K = 1

**Output:** 4

**Explanation:**

| **Time** | **Energy** | **X** | **Action**     | **Updated X** |
| -------- | ---------- | ----- | -------------- | ------------- |
| 0        | 0          | 1     | Nothing        | 1             |
| 1        | 1          | 1     | Break 3rd Lock | 2             |
| 2        | 2          | 2     | Nothing        | 2             |
| 3        | 4          | 2     | Break 2nd Lock | 3             |
| 4        | 3          | 3     | Break 1st Lock | 3             |

The locks cannot be broken in less than 4 minutes; thus, the answer is 4.

**Example 2:**

**Input:** strength = [2,5,4], K = 2

**Output:** 5

**Explanation:**

| **Time** | **Energy** | **X** | **Action**     | **Updated X** |
| -------- | ---------- | ----- | -------------- | ------------- |
| 0        | 0          | 1     | Nothing        | 1             |
| 1        | 1          | 1     | Nothing        | 1             |
| 2        | 2          | 1     | Break 1st Lock | 3             |
| 3        | 3          | 3     | Nothing        | 3             |
| 4        | 6          | 3     | Break 2nd Lock | 5             |
| 5        | 5          | 5     | Break 3rd Lock | 7             |

The locks cannot be broken in less than 5 minutes; thus, the answer is 5.



**Constraints:**

- `n == strength.length`
- `1 <= n <= 8`
- `1 <= K <= 10`
- `1 <= strength[i] <= 10^6`

这个题的方法是回溯，因为问题规模较小所以回溯也不会超时。

代码如下：

```java
class Solution {
    int res = Integer.MAX_VALUE;
    public int findMinimumTime(List<Integer> strength, int K) {
        dfs(strength, 1, K, 0, new boolean[strength.size()], 0);
        return res;
    }

    private void dfs(List<Integer> list, int x, int k, int finished, boolean[] visited, int sum) {
        if(finished == list.size()) {
            res = Math.min(res, sum);
        }

        for(int i = 0; i < list.size(); i ++) {
            if(!visited[i]) {
                visited[i] = true;
                int time = (list.get(i) + x - 1) / x;
                dfs(list, x + k, k, finished + 1, visited, sum + time);
                visited[i] = false;
            }
        }
    }
}
```

时间复杂度：

- 对于每一个lock在回溯时都有两种选择，即选或者不选，所以时间复杂度为`O(2^N)`

空间复杂度：

- 在dfs中递归调用的栈空间取决于递归的深度，深度即位列表的长度，所以需要`O(N)`的栈空间。
- 标记数组`visited`需要额外的`O(N)`。
- 综合复杂度为`O(N)`



# [2762. Continuous Subarrays](https://leetcode.com/problems/continuous-subarrays/)



You are given a **0-indexed** integer array `nums`. A subarray of `nums` is called **continuous** if:

- Let `i`, `i + 1`, ..., `j` be the indices in the subarray. Then, for each pair of indices `i <= i1, i2 <= j`, `0 <= |nums[i1] - nums[i2]| <= 2`.

Return *the total number of* ***continuous\*** *subarrays.*

A subarray is a contiguous **non-empty** sequence of elements within an array.



**Example 1:**

```plain
Input: nums = [5,4,2,4]
Output: 8
Explanation: 
Continuous subarray of size 1: [5], [4], [2], [4].
Continuous subarray of size 2: [5,4], [4,2], [2,4].
Continuous subarray of size 3: [4,2,4].
Thereare no subarrys of size 4.
Total continuous subarrays = 4 + 3 + 1 = 8.
It can be shown that there are no more continuous subarrays.
```



**Example 2:**

```plain
Input: nums = [1,2,3]
Output: 6
Explanation: 
Continuous subarray of size 1: [1], [2], [3].
Continuous subarray of size 2: [1,2], [2,3].
Continuous subarray of size 3: [1,2,3].
Total continuous subarrays = 3 + 2 + 1 = 6.
```



**Constraints:**

- `1 <= nums.length <= 10^5`
- `1 <= nums[i] <= 10^9`

因为题目中说要找到某个范围内符合条件的子数组，所以自然而然的能想到应该用滑动窗口来解决这个问题。

一般的滑动窗口问题采用滑动窗口 + 一个单调队列或堆就可以解决，但由于题目中要求范围内的数的差的绝对值小于等于2，所以只用一个是不能满足要求的，因为需要同时满足范围内的最大值和最小值都满足要求，所以需要两个单调队列或者一个最大堆和最小堆。

代码如下：

```java
// heap版
class Solution {
    public long continuousSubarrays(int[] nums) {
        long res = 0;
        Queue<int[]> min = new PriorityQueue<>((a, b) -> {
            return a[0] - b[0];
        });
        Queue<int[]> max = new PriorityQueue<>((a, b) -> {
            return b[0] - a[0];
        });
        int start = 0;
        for(int end = 0; end < nums.length; end ++) {
            while(!min.isEmpty() && min.peek()[0] + 2 < nums[end]) {
                int[] cur = min.poll();
                if(start <= cur[1]) {
                    start = cur[1] + 1;
                }
            }
            min.offer(new int[] {nums[end], end});
            while(!max.isEmpty() && max.peek()[0] > nums[end] + 2) {
                int[] cur = max.poll();
                if(start <= cur[1]) {
                    start = cur[1] + 1;
                }
            }
            max.offer(new int[] {nums[end], end});
            res += end - start + 1;
        }
        return res;
    }
}
```

时间复杂度：

- 每个元素最多进出堆一次，每次进出堆之后维护堆的复杂度为`O(logN)`，N个元素复杂度为`O(NlogN)`

空间复杂度：

- 每个堆最多存储N个元素，复杂度为`O(N)`



```java
// 单调队列版
class Solution {

    public long continuousSubarrays(int[] nums) {
        // Monotonic deque to track maximum and minimum elements
        Deque<Integer> maxQ = new ArrayDeque<>();
        Deque<Integer> minQ = new ArrayDeque<>();
        int left = 0;
        long count = 0;

        for (int right = 0; right < nums.length; right++) {
            // Maintain decreasing monotonic queue for maximum values
            while (!maxQ.isEmpty() && nums[maxQ.peekLast()] < nums[right]) {
                maxQ.pollLast();
            }
            maxQ.offerLast(right);

            // Maintain increasing monotonic queue for minimum values
            while (!minQ.isEmpty() && nums[minQ.peekLast()] > nums[right]) {
                minQ.pollLast();
            }
            minQ.offerLast(right);

            // Shrink window if max-min difference exceeds 2
            while (
                !maxQ.isEmpty() &&
                !minQ.isEmpty() &&
                nums[maxQ.peekFirst()] - nums[minQ.peekFirst()] > 2
            ) {
                // Move left pointer past the element that breaks the condition
                if (maxQ.peekFirst() < minQ.peekFirst()) {
                    left = maxQ.peekFirst() + 1;
                    maxQ.pollFirst();
                } else {
                    left = minQ.peekFirst() + 1;
                    minQ.pollFirst();
                }
            }

            // Add count of all valid subarrays ending at current right pointer
            count += right - left + 1;
        }
        return count;
    }
}
```

时间复杂度：

- 每个元素最多进出队列各一次，复杂度为`O(N)`

空间复杂度：

- 每个队列最多存储N个元素，复杂度为`O(N)`


# [1734. Decode XORed Permutation](https://leetcode.com/problems/decode-xored-permutation/)



There is an integer array `perm` that is a permutation of the first `n` positive integers, where `n` is always **odd**.

It was encoded into another integer array `encoded` of length `n - 1`, such that `encoded[i] = perm[i] XOR perm[i + 1]`. For example, if `perm = [1,3,2]`, then `encoded = [2,1]`.

Given the `encoded` array, return *the original array* `perm`. It is guaranteed that the answer exists and is unique.



**Example 1:**

```plain
Input: encoded = [3,1]
Output: [1,2,3]
Explanation: If perm = [1,2,3], then encoded = [1 XOR 2,2 XOR 3] = [3,1]
```

**Example 2:**

```plain
Input: encoded = [6,5,4,6]
Output: [2,4,1,5,3]
```



**Constraints:**

- `3 <= n < 10^5`
- `n` is odd.
- `encoded.length == n - 1`



因为每一个encoded都是从两个相邻的数得来的，所以只要知道任何一个perm就可以知道所有其他的perm。

推导过程如下：

a0 ^ a1 ^ ... ^ an = 1 ^ 2 ^ ... ^ n

a1 ^ a2 ^ ... ^ an = e1 ^ e3 ^ ... ^ en - 1

所以 a0 = e1 ^ e3 ^ ... ^ en - 1 ^ 1 ^ 2 ^ ... ^ n

代码如下：

```java
class Solution {
    public int[] decode(int[] encoded) {
        int a0 = 0;
        int n = encoded.length;
        for(int i = 1; i <= n + 1; i ++) {
            a0 ^= i;
        }
        for(int i = 1; i < n; i += 2) {
            a0 ^= encoded[i];
        }
        int[] res = new int[n + 1];
        res[0] = a0;
        for(int i = 1; i < res.length; i ++) {
            res[i] = res[i - 1] ^ encoded[i - 1];
        }
        return res;
    }
}
```

时间复杂度：

- `O(n)`

空间复杂度:

- `O(1)`



# [1930. Unique Length-3 Palindromic Subsequences](https://leetcode.com/problems/unique-length-3-palindromic-subsequences/)



Given a string `s`, return *the number of **unique palindromes of length three** that are a **subsequence** of* `s`.

Note that even if there are multiple ways to obtain the same subsequence, it is still only counted **once**.

A **palindrome** is a string that reads the same forwards and backwards.

A **subsequence** of a string is a new string generated from the original string with some characters (can be none) deleted without changing the relative order of the remaining characters.

- For example, `"ace"` is a subsequence of `"abcde"`.

 

**Example 1:**

```
Input: s = "aabca"
Output: 3
Explanation: The 3 palindromic subsequences of length 3 are:
- "aba" (subsequence of "aabca")
- "aaa" (subsequence of "aabca")
- "aca" (subsequence of "aabca")
```

**Example 2:**

```
Input: s = "adc"
Output: 0
Explanation: There are no palindromic subsequences of length 3 in "adc".
```

**Example 3:**

```
Input: s = "bbcbaba"
Output: 4
Explanation: The 4 palindromic subsequences of length 3 are:
- "bbb" (subsequence of "bbcbaba")
- "bcb" (subsequence of "bbcbaba")
- "bab" (subsequence of "bbcbaba")
- "aba" (subsequence of "bbcbaba")
```

 

**Constraints:**

- `3 <= s.length <= 10^5`
- `s` consists of only lowercase English letters.



因为要求的只是长度为3的回文序列，所以只需要找到相同的两个字符然后统计中间包含的不同的字符就可以了。

比如abcba，对于a，我们只需要找到两个a的位置，然后统计夹在中间的字符数量。关键问题在于去重。

类似前缀和，采用一个二维数组来记录每个位置所有字母出现的次数，这样就可以当确定某个区间的时候知道每个字符出现的频率。

代码如下：

```java
class Solution {
    public int countPalindromicSubsequence(String s) {
        int n = s.length();
        int[][] map = new int[n][26];
        int[] first = new int[26]; // The smallest idx of each character
        Arrays.fill(first, -1);
        for(int i = 0; i < n; i ++) {
            int cur = s.charAt(i) - 'a';
            if(first[cur] == -1) {
                first[cur] = i;
            }
            if(i != 0) {
                for(int j = 0; j < 26; j ++) {
                    map[i][j] = map[i - 1][j];
                }
            }
            map[i][cur] ++;
        }
        boolean[][] exists = new boolean[26][26];
        int res = 0;
        for(int i = 2; i < n; i ++) {
            int cur = s.charAt(i) - 'a';
            if(first[cur] == -1 || i - first[cur] < 2) {
                continue;
            }
            for(int k = 0; k < 26; k ++) {
                if(map[i - 1][k] - map[first[cur]][k] > 0 && !exists[cur][k]) {
                    res ++;
                    exists[cur][k] = true;
                }
            }
        }
        return res;
    }
}
```

时间复杂度：

- ` O(n)`

空间复杂度：

- `O(n)`



# [2381. Shifting Letters II](https://leetcode.com/problems/shifting-letters-ii/)



You are given a string `s` of lowercase English letters and a 2D integer array `shifts` where `shifts[i] = [starti, endi, directioni]`. For every `i`, **shift** the characters in `s` from the index `starti` to the index `endi` (**inclusive**) forward if `directioni = 1`, or shift the characters backward if `directioni = 0`.

Shifting a character **forward** means replacing it with the **next** letter in the alphabet (wrapping around so that `'z'` becomes `'a'`). Similarly, shifting a character **backward** means replacing it with the **previous** letter in the alphabet (wrapping around so that `'a'` becomes `'z'`).

Return *the final string after all such shifts to* `s` *are applied*.

 

**Example 1:**

```
Input: s = "abc", shifts = [[0,1,0],[1,2,1],[0,2,1]]
Output: "ace"
Explanation: Firstly, shift the characters from index 0 to index 1 backward. Now s = "zac".
Secondly, shift the characters from index 1 to index 2 forward. Now s = "zbd".
Finally, shift the characters from index 0 to index 2 forward. Now s = "ace".
```

**Example 2:**

```
Input: s = "dztz", shifts = [[0,0,0],[1,1,1]]
Output: "catz"
Explanation: Firstly, shift the characters from index 0 to index 0 backward. Now s = "cztz".
Finally, shift the characters from index 1 to index 1 forward. Now s = "catz".
```

 

**Constraints:**

- `1 <= s.length, shifts.length <= 5 * 10^4`
- `shifts[i].length == 3`
- `0 <= starti <= endi < s.length`
- `0 <= directioni <= 1`
- `s` consists of lowercase English letters.



如果直接模拟，问题的规模是5 * 10 ^ 4，最坏复杂度是`O(n^2)`，绝对会超时。

可以用一个数组统计每个位置需要移动的次数，类似前缀和，这样复杂度就降到了线性。



代码如下：

```java
class Solution {
    public String shiftingLetters(String s, int[][] shifts) {
        int n = s.length();
        int[] map = new int[n + 1];
        for(int[] arr: shifts) {
            if(arr[2] == 1) {
                map[arr[0]] ++;
                map[arr[1] + 1] --;
            } else {
                map[arr[0]] --;
                map[arr[1] + 1] ++;
            }
        }
        for(int i = 1; i < n; i ++) {
            map[i] += map[i - 1];
        }
        char[] str = s.toCharArray();
        for(int i = 0; i < n; i ++) {
            str[i] = (char) (((str[i] - 'a' + map[i]) % 26 + 26) % 26 + 'a');
        }
        return new String(str);
    }
}
```

时间复杂度：

- `O(n)`

空间复杂度：

- `O(n)`



# [983. Minimum Cost For Tickets](https://leetcode.com/problems/minimum-cost-for-tickets/)



You have planned some train traveling one year in advance. The days of the year in which you will travel are given as an integer array `days`. Each day is an integer from `1` to `365`.

Train tickets are sold in **three different ways**:

- a **1-day** pass is sold for `costs[0]` dollars,
- a **7-day** pass is sold for `costs[1]` dollars, and
- a **30-day** pass is sold for `costs[2]` dollars.

The passes allow that many days of consecutive travel.

- For example, if we get a **7-day** pass on day `2`, then we can travel for `7` days: `2`, `3`, `4`, `5`, `6`, `7`, and `8`.

Return *the minimum number of dollars you need to travel every day in the given list of days*.

 

**Example 1:**

```
Input: days = [1,4,6,7,8,20], costs = [2,7,15]
Output: 11
Explanation: For example, here is one way to buy passes that lets you travel your travel plan:
On day 1, you bought a 1-day pass for costs[0] = $2, which covered day 1.
On day 3, you bought a 7-day pass for costs[1] = $7, which covered days 3, 4, ..., 9.
On day 20, you bought a 1-day pass for costs[0] = $2, which covered day 20.
In total, you spent $11 and covered all the days of your travel.
```

**Example 2:**

```
Input: days = [1,2,3,4,5,6,7,8,9,10,30,31], costs = [2,7,15]
Output: 17
Explanation: For example, here is one way to buy passes that lets you travel your travel plan:
On day 1, you bought a 30-day pass for costs[2] = $15 which covered days 1, 2, ..., 30.
On day 31, you bought a 1-day pass for costs[0] = $2 which covered day 31.
In total, you spent $17 and covered all the days of your travel.
```

 

**Constraints:**

- `1 <= days.length <= 365`
- `1 <= days[i] <= 365`
- `days` is in strictly increasing order.
- `costs.length == 3`
- `1 <= costs[i] <= 1000`



线性规划

```java
class Solution {
    public int mincostTickets(int[] days, int[] costs) {
        int n = days.length;
        int[] dp = new int[days[n - 1] + 1 + 30];
        Arrays.fill(dp, Integer.MAX_VALUE);
        int last = -1;
        int[] len = new int[] {1, 7, 30};
        for(int i = days[n - 1] + 1; i < dp.length; i ++) {
            dp[i] = 0;
        }
        for(int i = n - 1; i >= 0; i --) {
            while(last != -1 && last > days[i]) {
                dp[last --] = dp[days[i + 1]];
            }
            last = days[i];
            for(int j = 0; j < costs.length; j ++) {
                dp[days[i]] = Math.min(costs[j] + dp[days[i] + len[j]], dp[days[i]]);
            }
        }
        return dp[days[0]];
    }
}
```

时间复杂度：

- `O(n)`

空间复杂度：

- `O(n)`








