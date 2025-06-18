---
theme: seriph
title: 'Accelerating Online Multiple-Choice Scoring: A SIMD Optimization on
  x86_64 Architectures'
info: Demonstrate ways to improve performance of MCQs scoring systems using
  x86_64 SIMD
author: Hoàng Minh Thiên <hoangminhthien05022009@gmail.com>
lineNumbers: true
colorSchema: dark
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# fonts
fonts:
  serif: Lora
  mono: JetBrains Mono
class: text-left
hideInToc: true
---

<img src="/img/vnin-logo.jpg" alt="vnin logo" class="h-22 absolute top-0 left-0" />
<img src="/img/conference.png" alt="conference logo" class="h-22 absolute top-0 right-18" />

# Accelerating Online Multiple-Choice Scoring: A SIMD Optimization on x86_64 Architectures

Hoàng Minh Thiên

---
hideInToc: true
---

# Table of Contents

<Toc :maxDepth="1" />

---

# Scope of this research

- SIMD on Intel x86_64 CPUs with AVX2 and AVX512BW, AVX512VL, AVX512F, AVX512DQ
- GNU Compiler Collection (GCC) >= 10.0
- C++20
- Single-threaded

---

# The problem

- Online tests are becoming more and more popular
- Servers usually bottleneck at peak exam hours (due to an enormous number of incoming requests)
- One element in the critical path is the scoring logic

---

# The solutions

- Modern CPUs provide single-instruction, multiple-data (SIMD) for speeding up repetitive workload

→ There is room for improvements here!

- Using automatic parallelization libraries (OpenMP, OpenCL, Intel's oneMKL, AMD's AOCL)
- Relying on compiler's auto-vectorization functionality (with `-O2`, `-O3`, `-mavx2`, etc. flags)
- DIY: handwritten SIMD intrinsics (easier) or inline assembly (more complex)

In this research, we focus on SIMD intrinsics

---

# GCC (sometimes) doesn't auto-vectorize

## The naive code

```cpp
#include <immintrin.h>
#include <stdint.h>

#include <array>
#include <iostream>
#include <vector>

using ByteArray = std::vector<char>;

std::vector<int32_t> score(const std::vector<ByteArray> &exams,
                           const ByteArray &correct_answers,
                           const ByteArray &points) {
    std::vector<int32_t> scored_exams_points(exams.size());
    for (size_t i = 0; i < exams.size(); ++i) {
        for (size_t j = 0; j < exams[i].size(); ++j) {
            if (exams[i][j] == correct_answers[j]) {
                scored_exams_points[i] += static_cast<int32_t>(points[j]);
            }
        }
    }
    return scored_exams_points;
}
```

---
layout: iframe
url: https://godbolt.org/e?readOnly=true#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIM6SuADJ4DJgAcj4ARpjE/gEADqgKhE4MHt6%2B/qRJKY4CIWGRLDFxZgF2mA5pQgRMxAQZPn5ctpj2%2BQy19QSFEdGxegp1DU1ZrcM9fcWlEgCUtqhexMjsHOYAzKHI3lgA1CYbbngsLKEExKEAdAiH2CYaAIKb27uYB0fD%2BII3dw/PTxeDB2Xn2hzc9WITAAnn9AWYtsC3h9jskLphWHDngjXqD3uCAG5VIjELH/LwpIx7CzQgiYR7EKHQj4AET2XxAICJDhI4OQCHqfw2VkBTw5XOJvKO5w2ZgA%2BgQ7uy0MRMBA0AxhuyCOhOdySeCaXSGUyleYAGyYVSsBSkf57B2Op3Ol0ujVao30xkwg5mc0q1UOOWGBQAd1itvtrujMYd7oI1NpXqZvvNSXOCjmBwA7CLHk7xfqpcdBLKFUqFCrMOg5VabXL04IFBA6ywFFcUgAvNVzOaHPNO/jEPYQLuYBV7PCsvYafuTlF7VvtscQPvC31WSx4LMmXNRl1Dkdjifaaez9en8GL61tkwAVgseHvLI7eG7q7n1ms2h3e6esYdPAqBHJd70fZ8wO0Z9WUONkA2JYNNXDYgFEg59fwHACCyrGslwbVAMzAp87zZL8NjZSZHGQOVRGGcEZXlRUNmwCBGwIVCHygki10w2NdxZfdnX4wThP/R1VQIZYGGVEhq1rG8FHwwjhX%2BfiOAWWhODvXg/A4LRSFQTg3C/Sx2SWFZ8QRHhSAITR1IWABrEA7w0fROEkHS7IMzheAUEBXNsvT1NIOBYCQNAWASOhYnISgIqi%2Bg4mALg71aGhaDpFDKCiLyolCepoU4ay8uYYhoQAeSibRiSK3gIrYQRyoYWhCqC0gsCiLxgAhWhaD87heCwFhDGAcQ2vwQNHCJfr9KtKovDpWryEEdovNoPAoihMqPCwLyLhOJaiWIKJkkwFlMGGox1qMOyFioAxgAUAA1PBMFDcqEkYJb%2BEEEQxHYKQZEERQVHUNrdFaAwbtMSxrH0Da/MgBZUASTp%2BoAWnKjY9nRqgqCYYZ0eGggEBx4aCVUMwcfRr5YJhzcLDMfTUCOy4sERqBmDYEAviO0gCTELx2Hp6wmd7NoOjSFwGHcTxmj0YJQn6EpBlaXJUgEMYWhyZINYYaYBjiCZ2mJGoRkaOXxgl02BG6BoDZVo3bHNrWhnNh3Zi4BYFHM1Z5jcjhtNIXTmc4PZVAADnNdHzUkPZgGQZA9hSq4uBHXBCBIX0Ni93hAq0cWnJcgOPODrzDI4Xz/Js26NM4MxPLaiu89r/mIylyQgA%3D%3D%3D
---

---

# Giving the compiler some hints

## The boolean multiplication code

````md magic-move
```cpp{14-16}
#include <stdint.h>

#include <array>
#include <vector>

using ByteArray = std::vector<char>;

std::vector<int32_t> score(const std::vector<ByteArray> &exams,
                           const ByteArray &correct_answers,
                           const ByteArray &points) {
    std::vector<int32_t> scored_exams_points(exams.size());
    for (size_t i = 0; i < exams.size(); ++i) {
        for (size_t j = 0; j < exams[i].size(); ++j) {
            if (exams[i][j] == correct_answers[j]) {
                scored_exams_points[i] += static_cast<int32_t>(points[j]);
            }
        }
    }
    return scored_exams_points;
}
```

```cpp{14-16}
#include <stdint.h>

#include <array>
#include <vector>

using ByteArray = std::vector<char>;

std::vector<int32_t> score(const std::vector<ByteArray> &exams,
                           const ByteArray &correct_answers,
                           const ByteArray &points) {
    std::vector<int32_t> scored_exams_points(exams.size());
    for (size_t i = 0; i < exams.size(); ++i) {
        for (size_t j = 0; j < exams[i].size(); ++j) {
            // Idea: to reduce false branch predictions
            scored_exams_points[i] += (exams[i][j] == correct_answers[j]) *
                                      static_cast<int32_t>(points[j]);
        }
    }
    return scored_exams_points;
}
```
````

---
layout: iframe
url: https://godbolt.org/e?readOnly=true#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIM6SuADJ4DJgAcj4ARpjE/gEADqgKhE4MHt6%2B/qRJKY4CIWGRLDFxZgF2mA5pQgRMxAQZPn4VmPb5DLX1BIUR0bHxtnUNTVkVwz2hfSUD5QCUtqhexMjsHOYAzKHI3lgA1CYbbngsLKEExKEAdAiH2CYaAIKb27uYB0cKBPiCN3cPzyeLwYOy8%2B0ObnqxCYAE9/kCzFsQW8PsdkhdMKx4c9Ea8we8IQA3KpEYjYgFeFJGPYWGEETCPYjQmEfAAiey%2B6BAIGJDhIEOQCHq/w2ViBT053N5pIh5w2ZgA%2BgQ7hy0MRMBA0AwvhzvlKSfyjrT6YzmSrzAA2TCqVgKUgAvaOp3Ol2u11anXGhlM2EHMwWtXqhwKwwKADusTtDrdMdjjo9BBpdO9zL9FqS5wUcwOAHYxY9nZKeQayUc5YrlRtsKqSJh0ArrbaFRnBAoII2WAorikAF4auZzQ75538Yh7CC9zBKvZ4Nl7DRDmeovYdruTiCD0V%2BqyWPDZkx56Ou0fjyfT7Rzhdbi8Qlc2zsmACsFjwT9Z3bwfY3i%2Bs1m0%2B8PJ440dAB6EC9gASSwJgQD2Ig9nVdAvFWPYqDEJQ9iiaEQQQPYEkQvBqgEBQj1jBQ1TrBt7wUZtUEzJ8XzfbdDnZdtqIY19H1ZBjtCYliWL2QMSRDbUI2IEjn14rjswAKlI4CFOAr4mEcZAFVEL5ZUEeUlTuCAWwICSLCk1lN2HN0D24oCnUs0j1QIZYGBrRCqKbAySNFAFLI4BZaE4R9eD8DgtFIVBODcX9LA5JYVgJREeFIAhNB8hYAGsQEfDR9E4SRAuS0LOF4BQQCypLgp80g4FgJA0BYBI6FichKFq%2Br6DiYAuEfLg%2BDoelxMoKJ8qiUJ6hhTgEuG5hiBhAB5KJtBJcbeFqthBBmhhaDG8rSCwKIvGASFaFoYruF4LAWEMYBxG2/Ag0cYkTpC60qi8eklvIQQ2ny2g8Cw0aPCwfKLhOd7iWIKJkkwVlMAuowfqMZKFioAxgAUAA1PBMDDGaEkYd7%2BEEEQxHYKQZEERQVHUbbdG6gwEdMSxrH0X7isgBZUASDoToAWhmjY9m5qg0K%2BbmLoIXDRaYQlVDMAXuc5FiGZ3CwzBC1AwcuLBWagZg2BATkwdIQkxC8dglesVWB1sNoSTSFwGHcTxmj0YIpmKUo9FyVIBFGPxuq9jpendgZusqIjOgmX29DDjougaIP%2BjiUPI6drJk%2B6BOZiThYFBi1YJF8/y8u2sKOD2VQAA4LW5i1JD2YBkGQPZOquLhx1wQgSD9DYuDmXgyq0K30sy7KOFy0ggrVwrbBKxLEcLjgzGLqeOH7%2BejcjO3JCAA%3D%3D%3D
---

---
layout: intro
---

# Optimizing with handwritten SIMD intrinsics

---

## SIMD in x86 CPUs

- Normal assembly instructions are **scalar** ones: takes one `A` and `B` and produce a result `C`. For example:
  `1 + 2 = 3`
- Single-instruction, multiple-data (SIMD) means that the instructions take multiple `A` and `B` and produce multiple
  result `C`. For example: `[1, 2, 3] + [4, 5, 6] = [5, 7, 9]`

---

## SIMD **intrinsics** introduction

- Intrinsics are compiler-generated functions for writing code closer to assembly.
- They provide us `__m128i` (SSE family), `__m256i` (AVX/AVX2 family), and `__m512i` (AVX512 family) as integer
  data types.
- We can pack many smaller integer types in those data types, as demonstrated later → This means that we can process
  multiple data at once.
- We will focus on AVX2 and AVX512 as they're widely available on modern CPUs.
- For more details, please see Intel® Intrinsics Guide
  at https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html.

---

## `_mm256_sad_epu8` intrinsic

As defined by Intel® Intrinsics Guide:

> Compute the absolute differences of packed **unsigned** 8-bit integers in `a` and `b`, then horizontally sum each
> consecutive 8 differences to produce four **unsigned** 16-bit integers, and pack these **unsigned** 16-bit integers in the
> low 16 bits of 64-bit elements in `dst`.

Symbol:

```cpp
__m256i _mm256_sad_epu8 (__m256i a, __m256i b)
```

→ This highlights the complex nature of x86_64 architecture: one instruction does multiple operations

---

## `_mm256_sad_epu8` visualized

<div class="font-mono">

| Vector/INT8  | 0    | 1    | 2    | 3    | 4    | 5    | 6    | 7    | ... | 31   |
|--------------|------|------|------|------|------|------|------|------|-----|------|
| A            | 0x01 | 0x02 | 0x03 | 0x04 | 0x05 | 0x06 | 0x07 | 0x08 |     | 0x20 |
| B            | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 |     | 0x00 |
| TMP          | 0x01 | 0x02 | 0x03 | 0x04 | 0x05 | 0x06 | 0x07 | 0x08 |     | 0x20 |
| Result (DST) | 0x24 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 |     | 0x00 |

</div>

→ By `_mm256_sad_epu8(A, _mm256_setzero_si256())`, we've calculated sums of four groups of eight 8-bit integers.

Note:

- `_mm256_setzero_si256` is an intrinsic to create a new, zeroed `__m256i`.
- `_mm512_sad_epu8` and `_mm512_setzero_si512` has the same functionality to this 256-bit variant.

---

## AVX2

```cpp {monaco}{height:'450px'}
#include <immintrin.h>
#include <stdint.h>

#include <array>
#include <vector>

using ByteArray = std::vector<char>;

std::vector<int32_t> score(const std::vector<ByteArray> &exams,
                           const ByteArray &correct_answers,
                           const std::vector<int8_t> &points) {
    std::vector<int32_t> scored_exams_points(exams.size(), 0);
    // We are targeting AVX2, so we have at most 256-bit registers,
    // which means that we can pack 32 MCQs to score at a time.
    constexpr int32_t BATCH_SIZE = 32;

    // Process each exam
    for (size_t i = 0; i < exams.size(); ++i) {
        auto &exam = exams[i];

        // Prefetch the next exam
        // https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html
        if (i + 1 < exams.size()) {
            __builtin_prefetch(&exams[i + 1]);
        }

        size_t j = 0;

        for (; j + BATCH_SIZE < correct_answers.size(); j += BATCH_SIZE) {
            // Vectorize exam's MCQs
            __m256i v1 = _mm256_set_epi8(
                exam[j + 31], exam[j + 30], exam[j + 29], exam[j + 28],
                exam[j + 27], exam[j + 26], exam[j + 25], exam[j + 24],
                exam[j + 23], exam[j + 22], exam[j + 21], exam[j + 20],
                exam[j + 19], exam[j + 18], exam[j + 17], exam[j + 16],
                exam[j + 15], exam[j + 14], exam[j + 13], exam[j + 12],
                exam[j + 11], exam[j + 10], exam[j + 9], exam[j + 8],
                exam[j + 7], exam[j + 6], exam[j + 5], exam[j + 4], exam[j + 3],
                exam[j + 2], exam[j + 1], exam[j]);
            // Vectorize correct MCQs
            __m256i v2 = _mm256_set_epi8(
                correct_answers[j + 31], correct_answers[j + 30],
                correct_answers[j + 29], correct_answers[j + 28],
                correct_answers[j + 27], correct_answers[j + 26],
                correct_answers[j + 25], correct_answers[j + 24],
                correct_answers[j + 23], correct_answers[j + 22],
                correct_answers[j + 21], correct_answers[j + 20],
                correct_answers[j + 19], correct_answers[j + 18],
                correct_answers[j + 17], correct_answers[j + 16],
                correct_answers[j + 15], correct_answers[j + 14],
                correct_answers[j + 13], correct_answers[j + 12],
                correct_answers[j + 11], correct_answers[j + 10],
                correct_answers[j + 9], correct_answers[j + 8],
                correct_answers[j + 7], correct_answers[j + 6],
                correct_answers[j + 5], correct_answers[j + 4],
                correct_answers[j + 3], correct_answers[j + 2],
                correct_answers[j + 1], correct_answers[j]);
            // Mark the correct answers with 0xff, otherwise 0
            v1 = _mm256_cmpeq_epi8(v1, v2);
            // Vectorize points
            v2 = _mm256_set_epi8(
                points[j + 31], points[j + 30], points[j + 29], points[j + 28],
                points[j + 27], points[j + 26], points[j + 25], points[j + 24],
                points[j + 23], points[j + 22], points[j + 21], points[j + 20],
                points[j + 19], points[j + 18], points[j + 17], points[j + 16],
                points[j + 15], points[j + 14], points[j + 13], points[j + 12],
                points[j + 11], points[j + 10], points[j + 9], points[j + 8],
                points[j + 7], points[j + 6], points[j + 5], points[j + 4],
                points[j + 3], points[j + 2], points[j + 1], points[j]);

            // Imagine that we have a mark vector of 3 elements 0xff00ff, and
            // the points are 0x010101, 0xff00ff & 0x010101 = 0x010001, which is
            // correct. Hence, the use of bitwise AND.
            v1 = _mm256_and_si256(v1, v2);
            
            // Reference:
            // https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm256_sad_epu8&ig_expand=5674
            // This will produce 4 sums, stored in the first 16 bits of every
            // 64-bit block inside the 256-bit vector
            v1 = _mm256_sad_epu8(v1, _mm256_setzero_si256());
            scored_exams_points[i] +=
                _mm256_extract_epi16(v1, 0) + _mm256_extract_epi16(v1, 4) +
                _mm256_extract_epi16(v1, 8) + _mm256_extract_epi16(v1, 12);
        }

        // Process the remaining batch (if any)
        for (; j < correct_answers.size(); ++j) {
            scored_exams_points[i] +=
                (exam[j] == correct_answers[j]) * points[j];
        }
    }

    return scored_exams_points;
}
```

---

## AVX512

```cpp {monaco}{height:'450px'}
#include <immintrin.h>
#include <stdint.h>

#include <array>
#include <vector>

using ByteArray = std::vector<char>;

std::vector<int32_t> score(const std::vector<ByteArray> &exams,
                           const ByteArray &correct_answers,
                           const std::vector<int8_t> &points) {
    // The idea is the same as the AVX2 functions, the only difference is that
    // we have double the registers' size (512 bits instead of 256 bits)
    std::vector<int32_t> scored_exams_points(exams.size());
    constexpr int32_t BATCH_SIZE = 64;

    // Process each exam
    for (size_t i = 0; i < exams.size(); ++i) {
        auto &exam = exams[i];

        // Prefetch the next exam
        if (i + 1 < exams.size()) {
            __builtin_prefetch(&exams[i + 1]);
        }

        size_t j = 0;

        for (; j + BATCH_SIZE < correct_answers.size(); j += BATCH_SIZE) {
            __m512i v1 = _mm512_set_epi8(
                exam[j + 63], exam[j + 62], exam[j + 61], exam[j + 60],
                exam[j + 59], exam[j + 58], exam[j + 57], exam[j + 56],
                exam[j + 55], exam[j + 54], exam[j + 53], exam[j + 52],
                exam[j + 51], exam[j + 50], exam[j + 49], exam[j + 48],
                exam[j + 47], exam[j + 46], exam[j + 45], exam[j + 44],
                exam[j + 43], exam[j + 42], exam[j + 41], exam[j + 40],
                exam[j + 39], exam[j + 38], exam[j + 37], exam[j + 36],
                exam[j + 35], exam[j + 34], exam[j + 33], exam[j + 32],
                exam[j + 31], exam[j + 30], exam[j + 29], exam[j + 28],
                exam[j + 27], exam[j + 26], exam[j + 25], exam[j + 24],
                exam[j + 23], exam[j + 22], exam[j + 21], exam[j + 20],
                exam[j + 19], exam[j + 18], exam[j + 17], exam[j + 16],
                exam[j + 15], exam[j + 14], exam[j + 13], exam[j + 12],
                exam[j + 11], exam[j + 10], exam[j + 9], exam[j + 8],
                exam[j + 7], exam[j + 6], exam[j + 5], exam[j + 4], exam[j + 3],
                exam[j + 2], exam[j + 1], exam[j]);
            __m512i v2 = _mm512_set_epi8(
                correct_answers[j + 63], correct_answers[j + 62],
                correct_answers[j + 61], correct_answers[j + 60],
                correct_answers[j + 59], correct_answers[j + 58],
                correct_answers[j + 57], correct_answers[j + 56],
                correct_answers[j + 55], correct_answers[j + 54],
                correct_answers[j + 53], correct_answers[j + 52],
                correct_answers[j + 51], correct_answers[j + 50],
                correct_answers[j + 49], correct_answers[j + 48],
                correct_answers[j + 47], correct_answers[j + 46],
                correct_answers[j + 45], correct_answers[j + 44],
                correct_answers[j + 43], correct_answers[j + 42],
                correct_answers[j + 41], correct_answers[j + 40],
                correct_answers[j + 39], correct_answers[j + 38],
                correct_answers[j + 37], correct_answers[j + 36],
                correct_answers[j + 35], correct_answers[j + 34],
                correct_answers[j + 33], correct_answers[j + 32],
                correct_answers[j + 31], correct_answers[j + 30],
                correct_answers[j + 29], correct_answers[j + 28],
                correct_answers[j + 27], correct_answers[j + 26],
                correct_answers[j + 25], correct_answers[j + 24],
                correct_answers[j + 23], correct_answers[j + 22],
                correct_answers[j + 21], correct_answers[j + 20],
                correct_answers[j + 19], correct_answers[j + 18],
                correct_answers[j + 17], correct_answers[j + 16],
                correct_answers[j + 15], correct_answers[j + 14],
                correct_answers[j + 13], correct_answers[j + 12],
                correct_answers[j + 11], correct_answers[j + 10],
                correct_answers[j + 9], correct_answers[j + 8],
                correct_answers[j + 7], correct_answers[j + 6],
                correct_answers[j + 5], correct_answers[j + 4],
                correct_answers[j + 3], correct_answers[j + 2],
                correct_answers[j + 1], correct_answers[j]);
            
            __mmask64 mask = _mm512_cmpeq_epi8_mask(v1, v2);
            v2 = _mm512_set_epi8(
                points[j + 63], points[j + 62], points[j + 61], points[j + 60],
                points[j + 59], points[j + 58], points[j + 57], points[j + 56],
                points[j + 55], points[j + 54], points[j + 53], points[j + 52],
                points[j + 51], points[j + 50], points[j + 49], points[j + 48],
                points[j + 47], points[j + 46], points[j + 45], points[j + 44],
                points[j + 43], points[j + 42], points[j + 41], points[j + 40],
                points[j + 39], points[j + 38], points[j + 37], points[j + 36],
                points[j + 35], points[j + 34], points[j + 33], points[j + 32],
                points[j + 31], points[j + 30], points[j + 29], points[j + 28],
                points[j + 27], points[j + 26], points[j + 25], points[j + 24],
                points[j + 23], points[j + 22], points[j + 21], points[j + 20],
                points[j + 19], points[j + 18], points[j + 17], points[j + 16],
                points[j + 15], points[j + 14], points[j + 13], points[j + 12],
                points[j + 11], points[j + 10], points[j + 9], points[j + 8],
                points[j + 7], points[j + 6], points[j + 5], points[j + 4],
                points[j + 3], points[j + 2], points[j + 1], points[j]);
            v1 = _mm512_maskz_mov_epi8(mask, v2);

            v1 = _mm512_and_si512(v1, v2);
            v1 = _mm512_sad_epu8(v1, _mm512_setzero_si512());

            // Split the 512-bit vector into two 256-bit vectors and then sum
            // them up like we did in AVX2
            const auto lo = _mm512_extracti32x8_epi32(v1, 0);
            const auto hi = _mm512_extracti32x8_epi32(v1, 1);
            scored_exams_points[i] +=
                _mm256_extract_epi16(lo, 0) + _mm256_extract_epi16(lo, 4) +
                _mm256_extract_epi16(lo, 8) + _mm256_extract_epi16(lo, 12) +
                _mm256_extract_epi16(hi, 0) + _mm256_extract_epi16(hi, 4) +
                _mm256_extract_epi16(hi, 8) + _mm256_extract_epi16(hi, 12);
        }

        // Process the remaining batch (if any)
        for (; j < correct_answers.size(); ++j) {
            scored_exams_points[i] +=
                (exam[j] == correct_answers[j]) * points[j];
        }
    }

    return scored_exams_points;
}
```

---

## Benchmarking system

- Ubuntu 24.04.2 LTS x86_64 6.8.0-60-generic
- 11th Gen Intel i5-1135G7 @ 2.419 GHz
- 4 GB RAM
- Enabled compiler flags:

```txt {line:false}
-O3: release build
-Wall: show all warnings
-mavx2 -mavx512bw -mavx512vl -mavx512f -mavx512dq: enable AVX2, AVX512{BW, VL, F, DQ}
-fno-omit-frame-pointer -fsanitize=address -fno-sanitize-recover=all: AddressSanitizer (to detect memory issues)
```

- Benchmarked using Google's benchmark library, with `BENCHMARK_LTO=ON`.
- We will use **real time** (wall time), not CPU time.

---
layout: intro
---

## Benchmarks

---
layout: image
image: /img/benchmark_results_avx512.json_100000.png
---

---
layout: image
image: /img/benchmark_results_avx512.json_5000000.png
---

---
layout: image
image: /img/benchmark_results_avx512.json_10000000.png
---

---
layout: intro
---

# Going further: optimizing the structure itself

---

## Data structure changes

```cpp {monaco}{maxHeight: '450px'}
#include <vector>

using ByteArray = std::vector<char>;
```

<v-clicks>

Problem: `std::vector` access time is slower than C arrays. How can we achieve:

- The ease of use of `std::vector`?
- The performance of C arrays?

→ Create a new data structure ourselves!

</v-clicks>

---

```cpp {monaco}{height: '450px'}
#include <cstring>
#include <stdexcept>

class ByteArray {
   private:
    size_t _size;
    size_t _capacity;
    size_t _block_count;
    int8_t *_values;

#if defined(__AVX512BW__) && defined(__AVX512VL__) && defined(__AVX512F__) && \
    defined(__AVX512DQ__)
    bool _avx512 = true;
#else
    bool _avx512 = false;
#endif

    void construct(const size_t &size) {
        _size = size;
#if defined(__AVX2__) && !(defined(__AVX512BW__) && defined(__AVX512VL__) && \
                           defined(__AVX512F__) && defined(__AVX512DQ__))
        _block_count = (_size >> 5) + ((_size & 31) != 0);
        _capacity = _block_count << 5;
#else
        _block_count = (_size >> 6) + ((_size & 63) != 0);
        _capacity = _block_count << 6;
#endif
        _values = static_cast<int8_t *>(calloc(_capacity, sizeof(int8_t)));
        if (!_values) {
            throw std::runtime_error("Failed to allocate memory for ByteArray");
        }
    }

   public:
    // Initialize an empty ByteArray
    ByteArray() : _size(0), _capacity(0), _block_count(0), _values(nullptr) {}
    // Initialize a ByteArray with `size`
    explicit ByteArray(const size_t &size) { construct(size); }
    
    // Some more constructors for copy/copy-move/intializer list constructions...

    // Operators
    int8_t &operator[](const size_t &index) const { return _values[index]; }

    // Getters
    [[nodiscard]] size_t capacity() const { return _capacity; }
    [[nodiscard]] size_t size() const { return _size; }
    [[nodiscard]] size_t block_count_avx2() const {
        return _avx512 ? _block_count << 1 : _block_count;
    }
    [[nodiscard]] size_t block_count_avx512() const {
        if (!_avx512) {
            throw std::runtime_error("AVX512 is not supported");
        }
        return _block_count;
    }

    // Get the `_values` array for direct access
    [[nodiscard]] int8_t *data() const { return _values; }

    // Iterators (for range-based for loops)
    [[nodiscard]] int8_t *begin() const { return _values; }
    [[nodiscard]] int8_t *end() const { return _values + _size; }

    ~ByteArray() {
        // Rule of thumb: free after use
        if (_values) free(_values);
    }
};
```

---
layout: intro
---

# Updated implementations

---

## AVX2

```cpp {monaco}{height: '450px'}
#include <immintrin.h>
#include <stdint.h>

#include <vector>

std::vector<int32_t> score(const std::vector<ByteArray> &exams,
                           const ByteArray &correct_answers,
                           const ByteArray &points) {
    std::vector<int32_t> scored_exams_points(exams.size(), 0);

    // Process each exam
    for (size_t i = 0; i < exams.size(); ++i) {
        auto &exam = exams[i];

        // Prefetch the next exam
        // https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html
        if (i + 1 < exams.size()) {
            __builtin_prefetch(&exams[i + 1]);
        }

        for (size_t j = 0, _j = 0; j < correct_answers.block_count_avx2();
             ++j, _j = j << 5) {
            // Vectorize exam's MCQs
            __m256i v1 = _mm256_loadu_si256(
                reinterpret_cast<const __m256i *>(exam.data() + _j));
            // Vectorize correct MCQs
            __m256i v2 = _mm256_loadu_si256(
                reinterpret_cast<const __m256i *>(correct_answers.data() + _j));

            // Mark the correct answers with 0xff, otherwise 0
            v1 = _mm256_cmpeq_epi8(v1, v2);
            // Vectorize points
            v2 = _mm256_loadu_si256(
                reinterpret_cast<const __m256i *>(points.data() + _j));

            // Imagine that we have a mark vector of 3 elements 0xff00ff, and
            // the points are 0x010101, 0xff00ff & 0x010101 = 0x010001, which is
            // correct. Hence, the use of bitwise AND.
            v1 = _mm256_and_si256(v1, v2);
            
            // Reference:
            // https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm256_sad_epu8&ig_expand=5674
            // This will produce 4 sums, stored in the first 16 bits of every
            // 64-bit block inside the 256-bit vector
            v1 = _mm256_sad_epu8(v1, _mm256_setzero_si256());
            scored_exams_points[i] +=
                _mm256_extract_epi16(v1, 0) + _mm256_extract_epi16(v1, 4) +
                _mm256_extract_epi16(v1, 8) + _mm256_extract_epi16(v1, 12);
        }
    }

    return scored_exams_points;
}
```

---

## AVX512

```cpp {monaco}{height: '450px'}
#include <immintrin.h>
#include <stdint.h>

#include <vector>

std::vector<int32_t> score(const std::vector<ByteArray> &exams,
                           const ByteArray &correct_answers,
                           const ByteArray &points) {
    std::vector<int32_t> scored_exams_points(exams.size(), 0);

    // Process each exam
    for (size_t i = 0; i < exams.size(); ++i) {
      auto &exam = exams[i];

      // Prefetch the next exam
      if (i + 1 < exams.size()) {
        __builtin_prefetch(&exams[i + 1]);
      }

      for (size_t j = 0, _j = 0; j < correct_answers.block_count_avx512();
           ++j, _j = j << 6) {
        // Load the exam and the correct answers
        __m512i v1 = _mm512_loadu_si512(exam.data() + _j);
        const __m512i v2 = _mm512_loadu_si512(correct_answers.data() + _j);

        // Compute the mask
        const __mmask64 mask = _mm512_cmpeq_epi8_mask(v1, v2);

        // Load the points
        v1 = _mm512_loadu_si512(points.data() + _j);

        // The masked points
        v1 = _mm512_maskz_mov_epi8(mask, v1);

        // Final sum calculation
        v1 = _mm512_sad_epu8(v1, _mm512_setzero_si512());
        uint64_t sum[8];
        _mm512_storeu_si512(sum, v1);

        for (const auto &k : sum) {
          // In fact, the results are stored at the lower 16-bit of each 64-bit
          // group, so we can safely cast here
          scored_exams_points[i] += static_cast<int32_t>(k);
        }
      }
    }

    return scored_exams_points;
}
```

---
layout: intro
---

## Benchmarks

---
layout: image
image: /img/benchmark_results_avx512_new_struct.json_100000.png
---

---
layout: image
image: /img/benchmark_results_avx512_new_struct.json_5000000.png
---

---
layout: image
image: /img/benchmark_results_avx512_new_struct.json_10000000.png
---

---

## Drawbacks

- More complex to set up.
- We're using more memory than required (if the number of MCQs is not a multiple of 32). E.g. when the number of
  questions is 200, the capacity will be raised to 224 (AVX2) or 256 (AVX512).
- Need to reimplement many operator overloading (unlike prebuilt ones from `std`)

---

# Conclusions

- SIMD utilization, if done correctly, can bring up to 3-7x performance improvement.
- Higher-memory, higher-performance (with custom, correctly aligned data structures).
- We can't entirely depend on GCC's auto-vectorization, yet.
- AVX512 doesn't provide a big boost of performance like advertised (because of the loading overhead).

---

# Notes

- The code repository is at https://github.com/hmthien050209/simd-research (with the older `std::vector<char>`-based version
  located on branch `avx512_vec`)
- For our operation, we don't need to care about number signedness, because our points are positive
- There are also `__m128d`, `__m256d`, and `__m512d`, which are double-precision floating point data types. For now,
  we focus on working with integer data types.
- x86_64 CPUs from AMD also provide 3DNow!, 3DNow! Professional, but they're deprecated in 2010. [^1]

[^1]: M. Yam, “AMD drops 3DNow! support from future CPUs,” Tom’s Hardware, Aug. 24, 2010. [Online].
Available: https://www.tomshardware.com/news/3dnow-simd-extensions-phenom-sse,11128.html

---

# Acknowledgements

- Đoàn Ngọc Bình Minh for providing me a VM on his laptop for the above benchmarks.
- Intel&reg; Intrinsics Guide.
- Intel&reg; 64 and IA-32 Architectures Software Developer's Manual Volume 1: Basic Architecture.

---
layout: end
hideInToc: true
---

# Thank you!