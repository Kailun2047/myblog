---
title: "Quick Review of KMP Algorithm"
date: 2020-05-02T23:40:40-07:00
draft: false
---

Published in 1977, the Knuth-Morris-Pratt algorithm can search substring pattern in time linear to the total string length. Recently I encountered 2 Leetcode questions (1367, 1392) that can be effectively solved by KMP, and therefore want to use this post as a quick refresher and go through the basic mechanism and some concrete examples of this algorithm. (A nice video explanation of KMP can be found [here](https://www.youtube.com/watch?v=uKr9qIZMtzw&t=764s)

# Intuition
In a naive substring search approach, when we find a mismatch between the pattern and the string, we shift the pattern by 1 character to start a new round of comparison (in the example below, we shift the starting `B` to index 1 after we detect the mismatch at index 4). But we already know the portion before `C` in the pattern, and we know that we're gonna compare `BAAAA` against `AAACB...`. This means we are doing some kind of "backup" after a mismatch. KMP avoids this backup by using deterministic finite state machine, which will be shown in detail in next section.

![alt test][naive]

[naive]: /images/kmp/substring_search_naive.png "naive search"

# The Algorithm
There are two major parts of the algorithm: 
1. given the target pattern, construct the state machine table
2. use the table constructed to iterate through the text to find matches

Since the table construction is the trickier part, we'll put it aside and take a look at the second part first. That being said, it's necessary to understand the meaning of the table.

![alt test][sample_table]

[sample_table]: /images/kmp/kmp_table.png "sample tables"

In the above examples, `p[j]` is the `j`th character of the pattern, and `next[j]` is the length of the substring of `p[0:j]` that satisfies: (1) this substring is a prefix of `p[0:j]`; (2) this substring is also a suffix of `p[0:j]`; (3) this substring is not `p[0:j]` itself. For example, when the pattern is `ABCDABD`, `next[5]` is 1 because the longest substring of `ABCDA` that is both its prefix and its suffix is `A`, which is of length 1. As for the last entry of the `next` array, it is appended there solely for computation (so that we know there's a match when we find one). When we use this table to traverse the input `text`, we can start with `j = 0`, and then use the table to find the next character in the pattern to compare with when there is a **mismatch**.

Say the target pattern is `p` and our goal is to find out all the starting index of substrings that matches `p` in the input `text`. Then we can use the following snippet to achieve this:

```go
func search(p, text string, next []int) []int {
    ans := make([]int, 0)
    for i, j := 0, 0; i < len(text); i++ {
        for j > 0 && text[i] != p[j] {
            j = next[j]
        }
        if text[i] == p[j] {
            j++
        }
        if j == len(p) {
            ans = append(ans, i - len(p) + 1)
            // Transition j when there is a match so that we don't get
            // index out of bound when we do p[j] in next iteration.
            j = next[j]
        }
    }
    return ans
}
```

Let's see how the program execute in the example where `text="ABCABCDAC"` and `p="ABCDABD"`. As we can see from the illustration below, when there's a mismatch in `(a)` we are at `p[3]` in the table, and `next[3]` is 0, meaning we should align `p[0]` to the current mismatched character 'A' in `text`, and this moves us to state `(b)`. Similarly, when there's a mismatch is `(c)`, we look it up and find we should align `p[1]` to the mismatched character 'C' in `text`, which moves us to state `(d)`. Notice that this time since we know that `text[7]` matches `p[0]` (both are 'A'), we don't need compare them again. This is how we can use the knowledge of previous portion of `p` in KMP (instead of using the previous portion of `text`, which leads to backup).

![alt test][kmp_example]

[kmp_example]: /images/kmp/kmp_examplle.png "KMP example"

Now it's time to move back to state machine construction. According to the meaning of `next` array, we can see that the first 2 entries of `next` are gonna be 0. We can use the snippet below to construct the rest of the table:

```go
func construct(p string) []int {
    next := []int{0, 0}
    for i, j := 1, 0; i < len(p); i++ {
        for j > 0 && p[j] != p[i] {
            j = next[j]
        }
        if p[i] == p[j] {
            j++
        }
        next = append(next, j)
    }
    return next
}
```

The construction code looks simple because it's quite similar to the iteration code. But it's trickier than it looks (at least to me). Let's use an example again to understand it.

![alt test][table_construction]

[table_construction]: /images/kmp/table_construction.png "table construction"

First of all, we can be sure that `next[0]` and `next[1]` are 0 according to the meaning of `next` array. Then when we start the for loop we need to set the initial value of `i` to 1 because the frist two entries are already filled. `j` starts from 0 because we are trying to compare the suffix to the prefix of `p[0:i]`, and the shortest prefix would be `p[0]`. In `(a)` we find that `p[1] == p[0] == 'A'`, so we append `next[0] + 1` to `next`; in `(b)` `p[2]` and `p[1]` mismatches, so we set `j` to `next[j]` (which is 0) and append it; in `(3)` we find that `p[3]` and `p[0]` matches, so we increment `j` to 1 and append it.

The construction process always uses the previous entries to compute the new entry (i.e. when we compares `p[j]` with `p[i]`, the entry `next[j]` is already computed), which is kind of like a dynamic programming approach.

# Analysis
Now that we've gone through the code of the algorithm, we can quickly prove why the algorithm is of linear time. Looking at the first code snippet, we can observe that the number of increments of `j` is bounded by the iteration variable `i`, meaning `j` can be incremented at most `n` times (where `n` is the length of `text`). Since `j` is always gonna be smaller after `j = next[j]` operation and it's bounded by 0, we can conclude that the number of decrements of `j` is also at most `n` (i.e. the inner for loop can execute at most `n` times). Therefore, the whole program is `O(n)` time. Similarly, the second code snippet will have time complexity of `O(m)`, making the overall time complexity `O(m+n)`.
