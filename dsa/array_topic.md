# Arrays, Strings & Hashing — Interview Questions + LeetCode Links

> A curated, topic-by-topic problem list mapped to the theory guide.
> Difficulty legend: 🟢 Easy · 🟡 Medium · 🔴 Hard
> Suggested order: do the ⭐ **must-do** ones first (they are the classic interview questions), then the rest to build pattern fluency.

---

## How to use this list
For each topic: solve the ⭐ core problem first (it *is* the pattern), then the variants right below it. If you can solve the whole block without hints, you've mastered that pattern. Aim for **2–4 problems per topic** before moving on.

---

## 1. Maximum / Minimum Subarray

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Maximum Subarray (Kadane) | 53 | 🟢/🟡 | https://leetcode.com/problems/maximum-subarray/ |
|    | Maximum Product Subarray | 152 | 🟡 | https://leetcode.com/problems/maximum-product-subarray/ |
|    | Maximum Sum Circular Subarray | 918 | 🟡 | https://leetcode.com/problems/maximum-sum-circular-subarray/ |
|    | Maximum Absolute Sum of Any Subarray | 1749 | 🟡 | https://leetcode.com/problems/maximum-absolute-sum-of-any-subarray/ |
|    | Best Time to Buy and Sell Stock (Kadane in disguise) | 121 | 🟢 | https://leetcode.com/problems/best-time-to-buy-and-sell-stock/ |
|    | Maximum Subarray Sum After One Operation | 1746 | 🟡 | https://leetcode.com/problems/maximum-subarray-sum-after-one-operation/ |

---

## 2. Kadane's Algorithm (variants & extensions)

Kadane is the engine behind topic 1. These push the idea further.

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Maximum Subarray | 53 | 🟢/🟡 | https://leetcode.com/problems/maximum-subarray/ |
|    | Maximum Product Subarray (track max & min) | 152 | 🟡 | https://leetcode.com/problems/maximum-product-subarray/ |
|    | Maximum Sum Circular Subarray | 918 | 🟡 | https://leetcode.com/problems/maximum-sum-circular-subarray/ |
|    | House Robber (DP cousin of Kadane) | 198 | 🟡 | https://leetcode.com/problems/house-robber/ |
|    | K-Concatenation Maximum Sum | 1191 | 🟡 | https://leetcode.com/problems/k-concatenation-maximum-sum/ |

---

## 3. Prefix & Suffix Sums

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Range Sum Query - Immutable | 303 | 🟢 | https://leetcode.com/problems/range-sum-query-immutable/ |
|    | Range Sum Query 2D - Immutable | 304 | 🟡 | https://leetcode.com/problems/range-sum-query-2d-immutable/ |
|    | Find Pivot Index | 724 | 🟢 | https://leetcode.com/problems/find-pivot-index/ |
|    | Running Sum of 1d Array | 1480 | 🟢 | https://leetcode.com/problems/running-sum-of-1d-array/ |
|    | Corporate Flight Bookings (difference array) | 1109 | 🟡 | https://leetcode.com/problems/corporate-flight-bookings/ |
|    | Subarray Sums Divisible by K | 974 | 🟡 | https://leetcode.com/problems/subarray-sums-divisible-by-k/ |

---

## 4. Subarray Sum Equals K (Prefix Sum + Hashmap)

The single most tested prefix-sum-with-hashmap family.

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Subarray Sum Equals K | 560 | 🟡 | https://leetcode.com/problems/subarray-sum-equals-k/ |
|    | Contiguous Array (equal 0s and 1s) | 525 | 🟡 | https://leetcode.com/problems/contiguous-array/ |
|    | Subarray Sums Divisible by K | 974 | 🟡 | https://leetcode.com/problems/subarray-sums-divisible-by-k/ |
|    | Continuous Subarray Sum | 523 | 🟡 | https://leetcode.com/problems/continuous-subarray-sum/ |
|    | Binary Subarrays With Sum | 930 | 🟡 | https://leetcode.com/problems/binary-subarrays-with-sum/ |
|    | Count Number of Nice Subarrays | 1248 | 🟡 | https://leetcode.com/problems/count-number-of-nice-subarrays/ |
|    | Maximum Size Subarray Sum Equals k (premium) | 325 | 🟡 | https://leetcode.com/problems/maximum-size-subarray-sum-equals-k/ |

---

## 5. Product of Array Except Self

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Product of Array Except Self | 238 | 🟡 | https://leetcode.com/problems/product-of-array-except-self/ |
|    | Trapping Rain Water (prefix-max / suffix-max) | 42 | 🔴 | https://leetcode.com/problems/trapping-rain-water/ |
|    | Maximum Product of Three Numbers | 628 | 🟢 | https://leetcode.com/problems/maximum-product-of-three-numbers/ |
|    | Product of the Last K Numbers | 1352 | 🟡 | https://leetcode.com/problems/product-of-the-last-k-numbers/ |

---

## 6. Merge Overlapping Intervals

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Merge Intervals | 56 | 🟡 | https://leetcode.com/problems/merge-intervals/ |
|    | Insert Interval | 57 | 🟡 | https://leetcode.com/problems/insert-interval/ |
|    | Non-overlapping Intervals | 435 | 🟡 | https://leetcode.com/problems/non-overlapping-intervals/ |
|    | Meeting Rooms (premium) | 252 | 🟢 | https://leetcode.com/problems/meeting-rooms/ |
|    | Meeting Rooms II (premium) | 253 | 🟡 | https://leetcode.com/problems/meeting-rooms-ii/ |
|    | Interval List Intersections | 986 | 🟡 | https://leetcode.com/problems/interval-list-intersections/ |
|    | Minimum Number of Arrows to Burst Balloons | 452 | 🟡 | https://leetcode.com/problems/minimum-number-of-arrows-to-burst-balloons/ |

---

## 7. Rotate Array

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Rotate Array (reversal trick) | 189 | 🟡 | https://leetcode.com/problems/rotate-array/ |
|    | Rotate List | 61 | 🟡 | https://leetcode.com/problems/rotate-list/ |
|    | Search in Rotated Sorted Array | 33 | 🟡 | https://leetcode.com/problems/search-in-rotated-sorted-array/ |
|    | Find Minimum in Rotated Sorted Array | 153 | 🟡 | https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/ |

---

## 8. Matrix Traversal & Rotation

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Rotate Image (90°) | 48 | 🟡 | https://leetcode.com/problems/rotate-image/ |
| ⭐ | Spiral Matrix | 54 | 🟡 | https://leetcode.com/problems/spiral-matrix/ |
|    | Spiral Matrix II | 59 | 🟡 | https://leetcode.com/problems/spiral-matrix-ii/ |
|    | Set Matrix Zeroes | 73 | 🟡 | https://leetcode.com/problems/set-matrix-zeroes/ |
|    | Transpose Matrix | 867 | 🟢 | https://leetcode.com/problems/transpose-matrix/ |
|    | Diagonal Traverse | 498 | 🟡 | https://leetcode.com/problems/diagonal-traverse/ |
|    | Search a 2D Matrix | 74 | 🟡 | https://leetcode.com/problems/search-a-2d-matrix/ |

---

## 9. Dutch National Flag (Sort 0s, 1s, 2s)

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Sort Colors | 75 | 🟡 | https://leetcode.com/problems/sort-colors/ |
|    | Move Zeroes | 283 | 🟢 | https://leetcode.com/problems/move-zeroes/ |
|    | Remove Duplicates from Sorted Array | 26 | 🟢 | https://leetcode.com/problems/remove-duplicates-from-sorted-array/ |
|    | Remove Element | 27 | 🟢 | https://leetcode.com/problems/remove-element/ |
|    | Partition Array (3-way / quickselect idea) | 905 | 🟢 | https://leetcode.com/problems/sort-array-by-parity/ |
|    | Wiggle Sort II | 324 | 🟡 | https://leetcode.com/problems/wiggle-sort-ii/ |

---

## 10. Majority Element (Boyer–Moore Voting)

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Majority Element (> n/2) | 169 | 🟢 | https://leetcode.com/problems/majority-element/ |
|    | Majority Element II (> n/3, two candidates) | 229 | 🟡 | https://leetcode.com/problems/majority-element-ii/ |
|    | Check If a Number Is Majority Element | 1150 | 🟢 | https://leetcode.com/problems/check-if-a-number-is-majority-element-in-a-sorted-array/ |

---

# Bonus — closely related Array / String / Hashing patterns

These aren't in your 10 topics but appear constantly in the same interviews. Add them once the core 10 feel comfortable.

## Hashing (seen-before / counts / pairs)

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Two Sum | 1 | 🟢 | https://leetcode.com/problems/two-sum/ |
|    | Contains Duplicate | 217 | 🟢 | https://leetcode.com/problems/contains-duplicate/ |
|    | Group Anagrams | 49 | 🟡 | https://leetcode.com/problems/group-anagrams/ |
|    | Valid Anagram | 242 | 🟢 | https://leetcode.com/problems/valid-anagram/ |
|    | Top K Frequent Elements | 347 | 🟡 | https://leetcode.com/problems/top-k-frequent-elements/ |
|    | Longest Consecutive Sequence | 128 | 🟡 | https://leetcode.com/problems/longest-consecutive-sequence/ |
|    | First Missing Positive | 41 | 🔴 | https://leetcode.com/problems/first-missing-positive/ |

## Two Pointers

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Two Sum II - Input Array Is Sorted | 167 | 🟡 | https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/ |
|    | 3Sum | 15 | 🟡 | https://leetcode.com/problems/3sum/ |
|    | Container With Most Water | 11 | 🟡 | https://leetcode.com/problems/container-with-most-water/ |
|    | Trapping Rain Water | 42 | 🔴 | https://leetcode.com/problems/trapping-rain-water/ |
|    | Valid Palindrome | 125 | 🟢 | https://leetcode.com/problems/valid-palindrome/ |
|    | Merge Sorted Array | 88 | 🟢 | https://leetcode.com/problems/merge-sorted-array/ |

## Sliding Window

| ⭐ | Problem | # | Difficulty | Link |
|----|---------|---|-----------|------|
| ⭐ | Longest Substring Without Repeating Characters | 3 | 🟡 | https://leetcode.com/problems/longest-substring-without-repeating-characters/ |
|    | Minimum Size Subarray Sum | 209 | 🟡 | https://leetcode.com/problems/minimum-size-subarray-sum/ |
|    | Longest Repeating Character Replacement | 424 | 🟡 | https://leetcode.com/problems/longest-repeating-character-replacement/ |
|    | Minimum Window Substring | 76 | 🔴 | https://leetcode.com/problems/minimum-window-substring/ |
|    | Sliding Window Maximum | 239 | 🔴 | https://leetcode.com/problems/sliding-window-maximum/ |
|    | Permutation in String | 567 | 🟡 | https://leetcode.com/problems/permutation-in-string/ |

---

## Suggested study plan (2–3 weeks)

| Week | Focus | Problems |
|------|-------|----------|
| Week 1 | Core patterns 1–5 | 53, 152, 303, 560, 525, 238 |
| Week 2 | Core patterns 6–10 | 56, 57, 189, 48, 54, 75, 169, 229 |
| Week 3 | Bonus + mixed revision | 1, 49, 347, 15, 11, 42, 3, 209, 76 |

**Interview tip:** in a real interview, first *state which pattern you recognize* and why ("this is contiguous + target sum with possible negatives → prefix sum + hashmap"), then code. Naming the pattern out loud is half the score.

---

*Total: ~60 problems. If you can confidently solve the ⭐ starred ones (about 15), you cover the vast majority of array interview rounds. Links point to leetcode.com; a few (Meeting Rooms 252/253, Max Size Subarray 325) require LeetCode Premium.*
