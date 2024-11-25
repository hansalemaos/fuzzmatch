# Fast Batch String Matching in Python (Levenshtein, Jaro-Winkler, Hamming) with Zero Cache Misses

## Introduction
This library provides very fast implementations of Levenshtein, Jaro-Winkler, and Hamming distance algorithms for batch matching of two input lists with zero cache misses (L1, LLi, LLd, LL). It is written in C++ and exposed to Python via Cython.

## Installation
```bash
pip install fuzzmatch
```

**Note**: Cython and a C++20 compiler must be installed. The library will be compiled the first time you import it.

## What Does It Do?
The state-of-the-art for fuzzy matching in Python is, without a doubt, [maxbachmann's RapidFuzz](https://github.com/rapidfuzz/RapidFuzz). I have been using it for years because of its speed (written in [C++](https://isocpp.org/) ) and its permissive [MIT license](https://opensource.org/license/mit) ( unlike the [GPL license](https://www.gnu.org/licenses/gpl-3.0.html) of [fuzzywuzzy](https://github.com/seatgeek/fuzzywuzzy) ). However, there is a type of task that I couldn't accomplish effectively using [RapidFuzz](https://rapidfuzz.github.io/RapidFuzz/): matching two large input lists against each other.

## Example Use Case
Consider the scenario where you want to learn a new language (a real language, not a programming language), and you already speak a European language (e.g., English, Italian, German, Spanish). Almost all European languages have significant influences from Latin, which is a great advantage because you can leverage your existing knowledge to learn thousands of new words quickly in almost any European language.


#### Example 1

| English | German     | Portuguese | French    | Spanish  | Italian   |
|---------|------------|------------|-----------|----------|-----------|
| define  | definieren | definir    | définir   | definir  | definire  |

#### Example 2

| English | German    | Portuguese | French   | Spanish  | Italian |
|---------|-----------|------------|----------|----------|---------|
| quality | Qualität  | qualidade  | qualité  | cualidad | qualità |



Creating a vocabulary list with all these words seems like a perfect job for fuzzy matching, doesn't it? The words are not identical but very similar. It should be an easy task using [Levenshtein](https://en.wikipedia.org/wiki/Levenshtein_distance) or the [Jaro-Winkler similarity](https://en.wikipedia.org/wiki/Jaro%E2%80%93Winkler_distance), right?   However, why didn't I succeed using [RapidFuzz](https://github.com/rapidfuzz/RapidFuzz)?

Let's take two languages that are very similar, so we can expect some 100% matches and many near matches (70–99%): [Spanish and Italian](https://www.quora.com/How-similar-are-Spanish-and-Italian-languages).

I used the word lists from this repository for my tests: [kkrypt0nn/wordlists](https://github.com/kkrypt0nn/wordlists/tree/main/wordlists/languages)

**The Italian word list contains 95,152 words.**

**The Spanish word list contains 70,157 words.**

Comparing each word from one list with every word from the other would require **6,675,578,864** comparisons.

RapidFuzz has a function called [**extractOne**](https://rapidfuzz.github.io/RapidFuzz/Usage/process.html#rapidfuzz.process.extractOne), which extracts the best match given a single word and a collection of words.

**Let's try extractOne from RapidFuzz with the classic "foo" example:**

```python
>>> rapidfuzz.process.extractOne("foo", ["f", "foos"])
('f', 90.0, 0)

>>> rapidfuzz.process.extractOne("foo", ["foos", "f"])
('f', 90.0, 1)
```

**Can you see the problem?**

The issue here is that the length of the string is ignored. While the algorithm might be correct according to its design, from a human perspective, **"foos"** is a better match to **"foo"** than **"f"**. Ignoring the string length might not be a problem in other cases, but in our scenario, it is significant. It means that single-character words (e.g., **"y", "o", "a"** in Spanish) match any word starting with the same letter.

For example, in Spanish, the word **"o"** means **"or"**, and **"otro"** means **"(an)other"**. **"Otra"** is the feminine version of **"otro"** ("otro hombre" / "otra mujer"), which means **"otra"** is much closer to **"otro"** than **"o"**. However, RapidFuzz gives us this:

```python
>>> rapidfuzz.process.extractOne("otro", ["o", "otra"])
('o', 90.0, 0)
```

Which is unfortunately useless in our case.
When changing scorers, we get other interesting results:

```python
# Looks much better, but...
>>> rapidfuzz.process.extractOne("foo", ["foos", "f"], scorer=rapidfuzz.fuzz.partial_ratio)
('foos', 100.0, 0)

# Different word order, different results, useless again:
>>> rapidfuzz.process.extractOne("foo", ["f", "foos"], scorer=rapidfuzz.fuzz.partial_ratio)
('f', 100.0, 0)
```

## What about [rapidfuzz.process.cdist](https://rapidfuzz.github.io/RapidFuzz/Usage/process.html#rapidfuzz.process.cdist)?

When I use RapidFuzz, I mostly use rapidfuzz.process.cdist because it returns a [NumPy](https://numpy.org/) array (len(array1) * len(array2)), which allows applying filters afterward using [Cython](https://cython.readthedocs.io/en/stable/src/userguide/source_files_and_compilation.html), [Numexpr](https://github.com/pydata/numexpr), or [NumPy](https://github.com/numpy/numpy), and it is [really fast](https://raw.githubusercontent.com/rapidfuzz/RapidFuzz/main/docs/img/cdist.svg?sanitize=true). It works perfectly as long as the input arrays are small. Unfortunately, there is no way to use it with an array that has almost 7 billion cells, like in our case (out of memory).

## How to Match Huge Lists
The library that I wrote doesn't aim to replace RapidFuzz but to solve the above-described problem. It is written in C++ and uses Cython to expose its functionality to Python, just like RapidFuzz. It has no dependencies except Cython. It always considers the longer match with more caracters, e.g. 

[**The Levenshtein distance of foo/fo is 1**](https://planetcalc.com/1721/?source=foo&target=foos)

[**The Levenshtein distance of foo/foos is 1**](https://planetcalc.com/1721/?source=foo&target=foos)

But since the matching string in the second example is longer, it will be chosen as the best match. 

**Here are the results (sorted by distance - best first) using the damerau_levenshtein_distance_2ways algorithm**:

[Spanish - Italian](https://github.com/hansalemaos/fuzzmatch/blob/main/spanish_italian/spanish_italian.xlsx)

[Italian - Spanish](https://github.com/hansalemaos/fuzzmatch/blob/main/spanish_italian/italian_spanish.xlsx)


### Performance and Benchmarks

The speed and quality of the results differ.

In general, you get the best quality using:

```python
pysm.ba_map_damerau_levenshtein_distance_2ways()
```
It's also the slowest algorithm. It calculates the [Damerau-Levenshtein](https://en.wikipedia.org/wiki/Damerau%E2%80%93Levenshtein_distance) distance twice (left to right and right to left) and returns the sum of them(that means foo/foos will return 2 instead of 1)

The substring/subsequence/Hamming algorithms are usually the fastest but don't match as well as the Levenshtein/Jaro-Winkler algorithms. However, there might be situations where they are enough to get the desired result.

In general, all algorithms are very fast compared to other libs (maybe the fastest you'll find right now) because the memory usage of each algorithm is very, very low ([The Levenshtein matrix](http://www.levenshtein.net/images/levenshtein_meilenstein_matrix.gif) needs only a single malloc for all calculations **(len(longest string in A+1)*len(longest string in B+1)))**, and because of that, cache misses are at 0.0% on all cache levels.

![](https://github.com/hansalemaos/fuzzmatch/blob/main/cachemisses.png?raw=true)

This is how I ran the benchmarks:

```bash
sudo valgrind --tool=callgrind --cache-sim=yes --branch-sim=yes --dump-instr=yes --collect-jumps=yes python3 fastmatch3.py
```

**If you don't understand what that means, here is an analysis of [ChatGPT](https://chatgpt.com/):**

The provided [Valgrind](https://valgrind.org/) output for your application reveals detailed statistics on memory access, cache performance, and branch prediction efficiency. Here's an in-depth analysis of the data:

### Cache Miss Rates

#### Instruction Cache (I1)

Total Instruction References (Ir): 8,172,620,951
L1 Instruction Cache Misses (I1mr): 2,603
Last Level Instruction Cache Misses (LLi): 2,382
Instruction Cache Miss Rate: Nearly 0.00% for both L1 and Last Level cache, indicating exceptional utilization of the cache for instructions.

#### Data Cache (D1)

Total Data References (Dr + Dw): 1,784,646,050 (1,410,101,732 reads + 374,544,318 writes)
L1 Data Cache Misses (D1mr + D1mw): 219,707 (206,611 reads + 13,096 writes)
Last Level Data Cache Misses (DLmr + DLmw): 14,611 (7,645 reads + 6,966 writes)
Data Cache Miss Rate: Both L1 and Last Level show miss rates of nearly 0.00%, suggesting highly efficient data cache management.

#### Overall Last Level Cache (LL) Miss Rate

Total LL Cache References: 222,310 (209,214 reads + 13,096 writes)
Total LL Cache Misses: 16,993 (10,027 reads + 6,966 writes)
Miss Rate: Still nearly 0.00%, underlining superb cache efficiency across the board.


#### Branch Prediction
Total Branches: 1,347,266,153 (1,347,205,192 conditional + 60,961 indirect)
Mispredicts: 25,329,233 (25,324,540 conditional + 4,693 indirect)
Misprediction Rate: 1.9% for conditional branches and 7.7% for indirect branches.

#### Analysis

#### Cache Performance
The cache performance metrics are extremely impressive, with miss rates at or near zero across all cache levels. This performance suggests that the program's instruction fetching and data access patterns are highly optimized for the caching mechanisms of the hardware, ensuring that data and instructions are readily available, minimizing the need to access slower main memory.

#### Branch Prediction

**Conditional Branches:** The misprediction rate of 1.9% indicates that the branch predictor handles most scenarios efficiently, though there may be complex or less predictable branching logic that could potentially be streamlined or optimized.
**Indirect Branches:** The higher misprediction rate of 7.7% for indirect branches suggests that dynamic branching (e.g., through function pointers or virtual method calls) might benefit from further optimization. This could include refactoring to reduce reliance on such branches where possible or optimizing the structures influencing these branches to enhance predictability. (if/else stuff, if you have good ideas to improve that - contribs are welcome)

Obs: this test was made using pure C++, to get only the data for the algorithms. If you repeat the test with Python, the results still will be at 0.0% for all cache levels, but the absolut numbers will be a little bit higher (Due to Python imports, conversion to dicts, etc..)

## How to use the library 

Here is a quick test that you can do to check if everything is working fine. All algorithms compare the top [500 songs listed by Rolling Stone magazine in 2004 and 2021](https://www.rollingstone.com/) to see which songs are listed in both years. There is not a single line that matches 100% a line in the other file. I was only able to get good results with RapidFuzz after applying different scorers and filtering the cdist NumPy arrays it returned. It was a whole bunch of work. The algorithms in this library should make it a lot easier.

The song lists and results are available [on the GitHub repo](https://github.com/hansalemaos/fuzzmatch/tree/main/rollingstone500songs)

I found both lists in this post: [Rolling Stone's 500 Greatest Songs of All Time](https://forums.stevehoffman.tv/threads/rolling-stones-500-greatest-songs-of-all-time-song-by-song-thread.1108916/)

### Usage Examples

#### For POSIX Systems

```python
from fuzzmatch import PyStringMatcher

pysm = PyStringMatcher( # input might be a list of strings/a single string/or a file path
    "/path/to/rollingstone2004.txt",
    "/path/to/rollingstone2021.txt",
)

# ab means: match first list to second
# ba means: match second list to first

_ = pysm.ab_map_damerau_levenshtein_distance_1way()
_ = pysm.ab_map_damerau_levenshtein_distance_2ways()
_ = pysm.ab_map_hamming_distance_1way()
_ = pysm.ab_map_hamming_distance_2ways()
_ = pysm.ab_map_jaro_2ways()
_ = pysm.ab_map_jaro_distance_1way()
_ = pysm.ab_map_jaro_winkler_distance_1way()
_ = pysm.ab_map_jaro_winkler_distance_2ways()
_ = pysm.ab_map_levenshtein_distance_1way()
_ = pysm.ab_map_levenshtein_distance_2ways()
_ = pysm.ab_map_longest_common_subsequence_v0()
_ = pysm.ab_map_longest_common_substring_v0()
_ = pysm.ab_map_longest_common_substring_v1()
_ = pysm.ab_map_subsequence_v1()
_ = pysm.ab_map_subsequence_v2()
_ = pysm.ba_map_damerau_levenshtein_distance_1way()
_ = pysm.ba_map_damerau_levenshtein_distance_2ways()
_ = pysm.ba_map_hamming_distance_1way()
_ = pysm.ba_map_hamming_distance_2ways()
_ = pysm.ba_map_jaro_2ways()
_ = pysm.ba_map_jaro_distance_1way()
_ = pysm.ba_map_jaro_winkler_distance_1way()
_ = pysm.ba_map_jaro_winkler_distance_2ways()
_ = pysm.ba_map_levenshtein_distance_1way()
_ = pysm.ba_map_levenshtein_distance_2ways()
_ = pysm.ba_map_longest_common_subsequence_v0()
_ = pysm.ba_map_longest_common_substring_v0()
_ = pysm.ba_map_longest_common_substring_v1()
_ = pysm.ba_map_subsequence_v1()
_ = pysm.ba_map_subsequence_v2()
```

#### For Windows Systems (Converted to Pandas/Excel and Sorted by Best Match)
```python
from fuzzmatch import PyStringMatcher
import pandas as pd

# also possible
# reading the data first, instead of passing a file path 

with open(
    r"C:\Users\hansc\source\repos\decltypestuff\rollingstone2004.txt",
    mode="r",
    encoding="utf-8",
) as f:
    data1 = f.read().splitlines()
with open(
    r"C:\Users\hansc\source\repos\decltypestuff\rollingstone2021.txt",
    mode="r",
    encoding="utf-8",
) as f:
    data2 = f.read().splitlines()

pysm = PyStringMatcher(
    data1,
    data2,
)

# pysm = PyStringMatcher(
#     r"C:\Users\hansc\source\repos\decltypestuff\rollingstone2004.txt",
#     r"C:\Users\hansc\source\repos\decltypestuff\rollingstone2021.txt",
# )

pd.DataFrame(pysm.ab_map_damerau_levenshtein_distance_1way()).T.to_excel(
    r"c:\ab_map_damerau_levenshtein_distance_1way.xlsx"
)
pd.DataFrame(pysm.ab_map_damerau_levenshtein_distance_2ways()).T.to_excel(
    r"c:\ab_map_damerau_levenshtein_distance_2ways.xlsx"
)
pd.DataFrame(pysm.ab_map_hemming_distance_1way()).T.to_excel(
    r"c:\ab_map_hemming_distance_1way.xlsx"
)
pd.DataFrame(pysm.ab_map_hemming_distance_2ways()).T.to_excel(
    r"c:\ab_map_hemming_distance_2ways.xlsx"
)
pd.DataFrame(pysm.ab_map_jaro_2ways()).T.to_excel(r"c:\ab_map_jaro_2ways.xlsx")
pd.DataFrame(pysm.ab_map_jaro_distance_1way()).T.to_excel(
    r"c:\ab_map_jaro_distance_1way.xlsx"
)
pd.DataFrame(pysm.ab_map_jaro_winkler_distance_1way()).T.to_excel(
    r"c:\ab_map_jaro_winkler_distance_1way.xlsx"
)
pd.DataFrame(pysm.ab_map_jaro_winkler_distance_2ways()).T.to_excel(
    r"c:\ab_map_jaro_winkler_distance_2ways.xlsx"
)
pd.DataFrame(pysm.ab_map_levenshtein_distance_1way()).T.to_excel(
    r"c:\ab_map_levenshtein_distance_1way.xlsx"
)
pd.DataFrame(pysm.ab_map_levenshtein_distance_2ways()).T.to_excel(
    r"c:\ab_map_levenshtein_distance_2ways.xlsx"
)
pd.DataFrame(pysm.ab_map_longest_common_subsequence_v0()).T.to_excel(
    r"c:\ab_map_longest_common_subsequence_v0.xlsx"
)
pd.DataFrame(pysm.ab_map_longest_common_substring_v0()).T.to_excel(
    r"c:\ab_map_longest_common_substring_v0.xlsx"
)
pd.DataFrame(pysm.ab_map_longest_common_substring_v1()).T.to_excel(
    r"c:\ab_map_longest_common_substring_v1.xlsx"
)
pd.DataFrame(pysm.ab_map_subsequence_v1()).T.to_excel(r"c:\ab_map_subsequence_v1.xlsx")
pd.DataFrame(pysm.ab_map_subsequence_v2()).T.to_excel(r"c:\ab_map_subsequence_v2.xlsx")
pd.DataFrame(pysm.ba_map_damerau_levenshtein_distance_1way()).T.to_excel(
    r"c:\ba_map_damerau_levenshtein_distance_1way.xlsx"
)
pd.DataFrame(pysm.ba_map_damerau_levenshtein_distance_2ways()).T.to_excel(
    r"c:\ba_map_damerau_levenshtein_distance_2ways.xlsx"
)
pd.DataFrame(pysm.ba_map_hemming_distance_1way()).T.to_excel(
    r"c:\ba_map_hemming_distance_1way.xlsx"
)
pd.DataFrame(pysm.ba_map_hemming_distance_2ways()).T.to_excel(
    r"c:\ba_map_hemming_distance_2ways.xlsx"
)
pd.DataFrame(pysm.ba_map_jaro_2ways()).T.to_excel(r"c:\ba_map_jaro_2ways.xlsx")
pd.DataFrame(pysm.ba_map_jaro_distance_1way()).T.to_excel(
    r"c:\ba_map_jaro_distance_1way.xlsx"
)
pd.DataFrame(pysm.ba_map_jaro_winkler_distance_1way()).T.to_excel(
    r"c:\ba_map_jaro_winkler_distance_1way.xlsx"
)
pd.DataFrame(pysm.ba_map_jaro_winkler_distance_2ways()).T.to_excel(
    r"c:\ba_map_jaro_winkler_distance_2ways.xlsx"
)
pd.DataFrame(pysm.ba_map_levenshtein_distance_1way()).T.to_excel(
    r"c:\ba_map_levenshtein_distance_1way.xlsx"
)
pd.DataFrame(pysm.ba_map_levenshtein_distance_2ways()).T.to_excel(
    r"c:\ba_map_levenshtein_distance_2ways.xlsx"
)
pd.DataFrame(pysm.ba_map_longest_common_subsequence_v0()).T.to_excel(
    r"c:\ba_map_longest_common_subsequence_v0.xlsx"
)
pd.DataFrame(pysm.ba_map_longest_common_substring_v0()).T.to_excel(
    r"c:\ba_map_longest_common_substring_v0.xlsx"
)
pd.DataFrame(pysm.ba_map_longest_common_substring_v1()).T.to_excel(
    r"c:\ba_map_longest_common_substring_v1.xlsx"
)
pd.DataFrame(pysm.ba_map_subsequence_v1()).T.to_excel(r"c:\ba_map_subsequence_v1.xlsx")
pd.DataFrame(pysm.ba_map_subsequence_v2()).T.to_excel(r"c:\ba_map_subsequence_v2.xlsx")

```