# search a 2d matrix: the O(log(m×n)) binary search walkthrough, the index-mapping formula, and the bugs that break most submissions

LeetCode 74 looks like a freebie. The data is sorted, you already know binary search, so what's the catch?

The catch is that "sorted" here is doing two different jobs, and most first attempts only use one of them. You end up with code that returns the right answer on the samples and still isn't the solution the problem statement asked for.

Here's what trips people up: the problem says "you must write a solution in O(log(m * n)) time complexity." That's not a hint. It's a requirement, and a row-by-row binary search doesn't satisfy it the way most people assume.

## What the problem actually guarantees

The matrix is `m x n`, and it comes with two properties:

1. Each row is sorted in non-decreasing order.
2. The first integer of each row is greater than the last integer of the previous row.

You're given a `target` and asked to return `true` if it exists, `false` otherwise. Constraints are small: `1 <= m, n <= 100`, and values sit between `-10^4` and `10^4`.

Property 2 is the one that matters. Property 1 alone gives you a bunch of separately sorted rows. Property 2 welds those rows together so the whole matrix reads as one ascending sequence when you go left-to-right, top-to-bottom. That's the observation the entire optimal solution rests on.

Which means: the moment you see a matrix with sorted rows and columns but no ordering guarantee *between* rows, this problem's trick stops working. That's Search a 2D Matrix II, and we'll get to it.

## Why the two obvious answers don't pass

**Scanning everything.** `O(m * n)` time, `O(1)` space. Correct, and completely misses the point. With `m` and `n` capped at 100 you'll never time out on LeetCode, but you'll fail the stated complexity requirement, and an interviewer will stop you before you finish typing.

**Binary search on every row.** `O(m * log n)`. Better, and a lot of candidates stop here. It uses property 1 and throws away property 2 entirely. On a 100×100 matrix that's about 100 × 7 = 700 comparisons instead of roughly 14 for the intended solution. The gap gets worse as rows multiply.

The intended answer treats the whole thing as one sorted array of length `m * n` and runs a single binary search over it.

## Treating the matrix as a virtual 1D array

You don't need to build the flat array. You need a way to go from a 1D index to a `(row, col)` pair, and that's two operations:


row = mid // cols
col = mid % cols


`cols` here is the number of columns, and it is *not* interchangeable with the number of rows. Mixing them up is the single most common bug in this problem, and it's a sneaky one because it works fine on square matrices.

The reverse mapping, if you ever need it, is `index = row * cols + col`.

### The code

A clean version in Python:

python
class Solution:
    def searchMatrix(self, matrix: List[List[int]], target: int) -> bool:
        if not matrix or not matrix[0]:
            return False

        rows, cols = len(matrix), len(matrix[0])
        left, right = 0, rows * cols - 1

        while left <= right:
            mid = left + (right - left) // 2
            row, col = divmod(mid, cols)
            value = matrix[row][col]

            if value == target:
                return True
            elif value < target:
                left = mid + 1
            else:
                right = mid - 1

        return False


`O(log(m * n))` time, `O(1)` space. Two index variables and a couple of locals, regardless of how big the matrix gets.

### A template variant worth knowing

If you've internalized the "find the first index where the condition becomes true" binary search template, the same solution looks like this:

python
        left, right = 0, rows * cols - 1
        first_true = -1

        while left <= right:
            mid = left + (right - left) // 2
            row, col = divmod(mid, cols)
            if matrix[row][col] >= target:
                first_true = mid
                right = mid - 1
            else:
                left = mid + 1

        if first_true == -1:
            return False
        row, col = divmod(first_true, cols)
        return matrix[row][col] == target


It costs you an extra bounds check at the end, and it generalizes better to the "first element >= target" family of problems. If you're drilling binary search patterns rather than memorizing one-off solutions, this shape is the more useful thing to carry around.

### Tracing it by hand

Take `target = 11` and this 3×3 matrix:


[ 1,  4,  7]
[10, 11, 13]
[15, 18, 21]


`rows * cols = 9`, so `left = 0`, `right = 8`.

- `mid = 4` → `row = 1, col = 1` → `matrix[1][1] = 11`. Match, return `true`.

Now try something that isn't there, `target = 12`, same matrix:

- `mid = 4` → `11 < 12` → `left = 5`
- `mid = 6` → `row = 2, col = 0` → `15 > 12` → `right = 5`
- `mid = 5` → `row = 1, col = 2` → `13 > 12` → `right = 4`
- Loop exits because `left > right`. Return `false`.

Three probes to eliminate nine cells. That's the whole pitch.

## Two binary searches instead of one

There's a second valid method, and plenty of people find it easier to reason about under pressure:

1. Binary search the *rows* to find the only row that could contain `target`. Compare against the row's first and last element: if `target` is smaller than the first, move up; larger than the last, move down; otherwise you've found your row.
2. Binary search *inside* that row.


O(log m + log n) = O(log(m * n))


They're the same complexity. `log(x*y) = log(x) + log(y)`, so the two-phase version isn't asymptotically worse, it's just two loops instead of one.

Which should you write? The single loop is shorter and harder to get subtly wrong. The two-phase version is easier to narrate out loud. If you're the kind of person who needs to explain the invariant at every step, two-phase might survive an interview better.

## The staircase works here too, and it's still not what they want

Start at the top-right corner. If the current cell is larger than `target`, move left. If it's smaller, move down. Repeat until you fall off the grid or hit the target.

That's `O(m + n)`, which is worse than `O(log(m * n))` but far better than a full scan. And it does work on LeetCode 74, because the global row ordering also forces each column to be sorted: since row `i`'s last element is strictly less than row `i+1`'s first element, every element in column `j` of row `i` is smaller than any element in column `j` of row `i+1`.

So you can submit it. It passes. It just isn't the answer the problem is testing for, and saying "I know a linear one but I can do better" is a stronger move than leading with it.

## Search a 2D Matrix vs Search a 2D Matrix II

This comparison shows up in interviews constantly, and getting it backwards is worse than not knowing it.

|  | Search a 2D Matrix (74) | Search a 2D Matrix II (240) |
| --- | --- | --- |
| Row ordering | Each row sorted ascending | Each row sorted ascending |
| Column ordering | Guaranteed, and rows are ordered relative to each other | Sorted ascending, but rows are independent |
| Global order | Yes, matrix reads as one sorted sequence | No |
| Flattening trick | Valid | **Invalid** |
| Best-known approach | Single binary search, `O(log(m*n))` | Staircase from a corner, `O(m + n)` |
| Also viable | Two-phase binary search | Per-row binary search, `O(m log n)` |
| Constraints | `m, n <= 100` | `m, n <= 300` |

The rule of thumb: if you can read the whole matrix in row-major order and get a sorted sequence, flatten. If you can only say "each row and each column is sorted on its own," the flatten trick produces wrong answers, and you need the staircase.

Plenty of candidates float "just treat it as a 1D array" on problem 240 out of habit. That's an instant red flag.

## The bugs that actually cost submissions

A short list, in rough order of how often they bite:

- **`row = mid // rows` instead of `mid // cols`.** Works on square matrices, breaks on `[[1,1]]` and anything rectangular. The corner case `[[1,1]]` with `target = 1` is a good smoke test.
- **Skipping the empty guard.** `if not matrix or not matrix[0]: return False`. Without it, `len(matrix[0])` throws on an empty input.
- **`while left < right`.** With inclusive bounds you want `left <= right`. Dropping the equality check means the final candidate never gets compared.
- **Off-by-one in `right`.** The search space is `[0, m*n - 1]`, not `[0, m*n]`.
- **`(left + right) // 2` in fixed-width languages.** In Python you're fine. In Java or C++ it's a habit worth dropping in favor of `left + (right - left) // 2`, since overflow on large indices is a real failure mode outside interview-sized inputs.
- **Calling the staircase on problem 74 in interview and then claiming `O(log(m*n))`.** The complexity claim has to match the code.

Test cases worth running before you submit: empty matrix, single row, single column, `target` smaller than `matrix[0][0]`, `target` larger than the last element, and `target` sitting exactly at the first or last virtual index.

## How to explain it out loud

Sixty seconds, in this order:

> "The matrix is globally sorted. Each row is sorted, and every row starts above where the previous row ended, so reading row-major gives one ascending array. I'll binary search that array logically, without building it. Index `mid` maps to `matrix[mid / cols][mid % cols]`. If it's smaller than the target, I move `left`; larger, I move `right`; equal, return true. If the pointers cross, the target isn't there. `O(log(m*n))` time, `O(1)` space."

Then code it. If they push on why you didn't search rows first, you have the answer ready: it's `O(m log n)`, which uses one property and ignores the other.

## Where AlgoMonster fits into this

The reason this problem feels easy to read and annoying to get exactly right is that it's a *template* problem, not a trick problem. The index mapping, the inclusive bounds, the first-true-index variant, the edge cases: those are the same pieces that show up again in rotated-array search, "find first element >= target," and search-in-2D variations.

AlgoMonster is built around exactly that framing. It's a text-and-diagram based interview prep site, with content written by Google engineers and competitive programmers, and it organizes material by pattern rather than by problem number. Its write-up of LeetCode 74 [uses the "find the first index where the condition holds" binary search template](https://algo.monster?fpr=li69) and then verifies the match at the end, which is the generalized version of the solution above. It also points out the thing worth internalizing: the elegance is in treating the 2D matrix as a 1D sorted array without actually reshaping it.

### What's actually in Pro

Per the official plan page, a Pro subscription unlocks:

- All patterns, lessons, and problems
- Unlimited AI assistant usage
- Unlimited Speedrun sessions
- Real questions from top tech companies, refreshed monthly
- 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials
- Member discount on live bootcamps
- Bonus courses included: a CS fundamentals course covering data structures and algorithms from scratch, and a "Build an LLM" course covering tokenization, attention, and training

Worth flagging what isn't there: AlgoMonster is text and diagram based, with no video walkthroughs. A third-party breakdown puts the curriculum at 48 patterns and around 231 problems, versus LeetCode's several thousand. So if you want a giant problem bank, this isn't it. If you want the patterns that recur across those thousands of problems, the ratio flips in its favor.

### Current plans and pricing

Pricing on the official plan page as of this writing, and it does move around with promotions:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems | $0 | No card needed | [Try the free tier first](https://algo.monster?fpr=li69) |
| Pro - Yearly | Everything unlocked: all patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun, 500+ lessons across 6 courses, monthly-updated company questions, bonus courses, bootcamp discount, 12 months of access | $5.84/month, billed as $69.99/year | Annual, cancel or upgrade anytime | [Get Pro yearly access](https://algo.monster?fpr=li69) |
| Pro - Lifetime | Everything in the annual plan, plus lifetime updates on all future content | $89 | One-time | [Get Pro lifetime access](https://algo.monster?fpr=li69) |

There's no monthly plan and no team or enterprise tier: it's annual or lifetime, individual accounts only.

Two things stand out in those numbers. First, lifetime at $89 costs barely more than a single year at $69.99, which makes the annual plan a hard sell unless you're sure you'll be done with interview prep in under twelve months. Second, a lot of older blog posts quote $119, $189, or even $459 for these same plans. Those numbers don't match the current page, so treat any review with a fixed price in the headline as stale.

### Is it the right call for this problem?

If you're on LeetCode 74 and already comfortable with binary search, you don't need to pay anyone. The article you just read covers it.

Where a structured course earns its money is the part after that: knowing that the first-true-index template in problem 74 is the same one you'll reach for in "find minimum in rotated sorted array" or "koko eating bananas." Pattern recognition is what interviews actually measure, and it's the slowest thing to build from random problem grinding. AlgoMonster's bet is that a smaller set of problems, organized by pattern, gets you there faster than a bigger set organized by difficulty. That's a reasonable bet for someone with four to eight weeks and no idea where to start, and a worse one for someone who wants three thousand problems to grind through.

The free tier is enough to see whether the text-and-diagram format clicks for you. That's the honest test, and it's a cheaper one than any refund policy.
