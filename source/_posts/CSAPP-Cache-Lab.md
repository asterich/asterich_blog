---
layout: posts
title: CSAPP-Cache-Lab
tags:
  - CSAPP
  - 计算机体系结构
  - cache
  - C
category:
  - 编程
date: 2023-01-10 23:55:57
mathjax: true
---

这个 Lab 对应的是 CSAPP 第 6 章，首先是手写一个缓存模拟器，然后去优化一个矩阵转置。

<!--more-->

## 第一部分 缓存模拟器

我们首先看一下缓存结构：
![](/2023/01/10/CSAPP-Cache-Lab/image/cache_struct.png)
可以看到，一个缓存是由 $S \times E$ 个缓存行(cache line)组成的，其中 $S$ 是组(set)的个数，$E$ 是每组行数(在一些地方被叫做 associativity)；一个缓存行由有效位(valid bit)、标签(tag)、数据块(block)组成，其中有效位和 tag 共同确定一个缓存行对应的地址，而数据块是真正存放数据的地方。

再看一下内存地址与缓存的映射关系：
![](/2023/01/10/CSAPP-Cache-Lab/image/addr_mapping.png)
可以看到，内存地址对应的组由中间几位决定，而具体到哪个缓存行，则是随便你怎么搞。因此，每次找缓存行，都会遍历整个组，找符合的 tag 和 block offset。当然，缓存模拟器不考虑 block offset。注意，我们这个缓存模拟器，无论是读缓存还是写缓存，都应该是遍历整个组的。

下面说 eviction。每当一个组被填满时，就需要 evict 一个缓存行；而 evict 的策略是组内的 LRU(least-recently used)，就是找组内最早访问过的 缓存行，然后把它给丢掉。要想实现 LRU，可以用优先队列或者时间戳。优先队列比较难写，没有必要，所以这里采用时间戳。

Lab 要求我们读入 valgrind 生成的 memory trace。数据的访存总共有 3 种：加载、存储和修改，分别用字符'L'、'S'和'M'表示。L 和 S 被看作是一样的；M 是一个 S 后面接一个 L，后面的 L 一定是命中的。还有一种是指令访存('I')，这种是要忽略的。
话不多说，上代码：

```c
#include <getopt.h>
#include <inttypes.h>
#include <limits.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "cachelab.h"

int s = 0, E = 0, b = 0;
int hits = 0, misses = 0, evictions = 0;
int verbose = 0;
char tracefile[256];

typedef enum _bool {
  FALSE,
  TRUE,
} _bool_t;

// 缓存行
typedef struct cache_line {
  _bool_t valid_bit;
  int tag;
  int timestamp;
} cache_line_t;

// cache[S][E]
cache_line_t *cache = NULL;

#define SET(address, s, b) (((address) & (((1 << (s)) - 1) << (b))) >> (b))
#define TAG(address, s, b) ((address) >> ((s) + (b)))

int min_int(int a, int b) { return ((a) < (b) ? (a) : (b)); }

void init_cache(cache_line_t **c, int s, int E) {
  *c = (cache_line_t *)calloc((1 << s) * E, sizeof(cache_line_t));
  if (*c == NULL) {
    perror("init cache failed");
  }
  return;
}

void free_cache(cache_line_t **c) {
  free(*c);
  *c = NULL;
  return;
}

// 判断是否命中
_bool_t is_hit(cache_line_t *c, uint64_t address, int timestamp) {
  int set = SET(address, s, b);
  int tag = TAG(address, s, b);
  for (int curr_line = set * E;
       curr_line < min_int((set + 1) * E, (1 << s) * E); ++curr_line) {
    if (c[curr_line].valid_bit == TRUE && tag == c[curr_line].tag) {
      c[curr_line].timestamp = timestamp;
      return TRUE;
    }
  }
  return FALSE;
}

// 判断缓存行是否满了，需要evict
_bool_t is_full(cache_line_t *c, uint64_t address) {
  int set = SET(address, s, b);
  for (int curr_line = set * E;
       curr_line < min_int((set + 1) * E, (1 << s) * E); ++curr_line) {
    if (c[curr_line].valid_bit == FALSE) {
      return FALSE;
    }
  }
  return TRUE;
}

// 写入缓存当中
void write_cache(cache_line_t *c, uint64_t address, int timestamp) {
  int set = SET(address, s, b);
  int tag = TAG(address, s, b);
  for (int curr_line = set * E;
       curr_line < min_int((set + 1) * E, (1 << s) * E); ++curr_line) {
    if (c[curr_line].valid_bit == FALSE) {
      c[curr_line].valid_bit = TRUE;
      c[curr_line].tag = tag;
      c[curr_line].timestamp = timestamp;
      break;
    }
  }
  return;
}

// evict
void evict(cache_line_t *c, uint64_t address, int timestamp) {
  int set = SET(address, s, b);
  int lru_cacheline_idx = set * E;
  int min_timestamp = INT_MAX;
  for (int curr_line = set * E;
       curr_line < min_int((set + 1) * E, (1 << s) * E); ++curr_line) {
    if (c[curr_line].timestamp < min_timestamp) {
      lru_cacheline_idx = curr_line;
      min_timestamp = c[curr_line].timestamp;
    }
  }
  c[lru_cacheline_idx].valid_bit = FALSE;
  c[lru_cacheline_idx].tag = 0;
  return;
}

// 每一行valgrind都做此处理
void determine(cache_line_t *c, uint64_t address, char op, int timestamp,
               char verbose_str[]) {
  switch (op) {
  case 'S':
  case 'L': {
    if (is_hit(c, address, timestamp) == TRUE) {
      ++hits;
      strcat(verbose_str, " hit");
    } else {
      ++misses;
      strcat(verbose_str, " miss");

      if (is_full(c, address)) {
        evict(c, address, timestamp);
        ++evictions;
        strcat(verbose_str, " eviction");
      }

      write_cache(c, address, timestamp);
    }
    break;
  }

  case 'M': {
    if (is_hit(c, address, timestamp) == TRUE) {
      ++hits;
      strcat(verbose_str, " hit");
    } else {
      ++misses;
      strcat(verbose_str, " miss");

      if (is_full(c, address)) {
        evict(c, address, timestamp);
        ++evictions;
        strcat(verbose_str, " eviction");
      }

      write_cache(c, address, timestamp);
    }

    is_hit(c, address, timestamp); // just update timestamp here
    ++hits;
    strcat(verbose_str, " hit");

    break;
  }

  default: {
    break;
  }
  }
}

int main(int argc, char **argv) {
  int opt;
  /* looping over arguments */
  while (-1 != (opt = getopt(argc, argv, "vs:E:b:t:"))) {
    /* determine which argument it’s processing */
    switch (opt) {
    case 'v':
      verbose = 1;
      break;
    case 's':
      s = atoi(optarg);
      break;
    case 'E':
      E = atoi(optarg);
      break;
    case 'b':
      b = atoi(optarg);
      break;
    case 't':
      strncpy(tracefile, optarg, 255);
      break;
    default:
      printf("wrong argument\n");
      break;
    }
  }

  init_cache(&cache, s, E);

  FILE *p_file = fopen(tracefile, "r");

  char op;
  uint64_t address;
  int size;
  int timestamp = 0;
  while (fscanf(p_file, " %c %" PRIx64 ",%d", &op, &address, &size) > 0) {
    if (op == 'L' || op == 'S' || op == 'M') {
      char verbose_str[128];
      memset(verbose_str, 0, sizeof(verbose_str));
      snprintf(verbose_str, 127, "%c %" PRIx64 ",%d", op, address, size);
      determine(cache, address, op, timestamp, verbose_str);
      ++timestamp;
      if (verbose) {
        printf("%s\n", verbose_str);
      }
    }
  }

  printSummary(hits, misses, evictions);

  fclose(p_file);
  free_cache(&cache);

  return 0;
}

```

## 第二部分 矩阵转置

这个第二部分是优化矩阵转置的，比第一部分要捏吗难很多。我们要尽可能降低矩阵转置的 cache miss。缓存有 32 组，直接映射，每行的块大小是 32 字节。Lab 要求测试三种尺寸：32x32、64x64、61x67，并且转置函数里面不能有超过 12 个自由变量。A 和 B 地址相差 0x40000。

要想降低 cache miss，我们一般有两种方法：分块、改变访存模式。我们先从最简单的分块着手。

每行 32 字节，能放得下 8 个 int，所以我们先分 8x8 的块：

```c
#define CEIL_DIV(i, block_size_per_dim)                                        \
  (((i) + (block_size_per_dim)-1) / block_size_per_dim)
#define MIN(a, b) ((a) > (b) ? (b) : (a))

int tmp;

for (int jj = 0; jj < CEIL_DIV(M, block_size_per_dim); ++jj)
    for (int ii = 0; ii < CEIL_DIV(N, block_size_per_dim); ++ii) {
        for (int i = ii * block_size_per_dim;
                i < MIN((ii + 1) * block_size_per_dim, N); i++) {
            for (int j = jj * block_size_per_dim;
                j < MIN((jj + 1) * block_size_per_dim, M); j++) {
                tmp = A[i][j];
                B[j][i] = tmp;
            }
        }
    }
```

这样已经足以解决 61x67 的矩阵了，但是在 32x32 上仍然有 344 个 miss，达不到满分要求。这是因为 A 和 B 的地址隔了 0x40000，在对角线上 A 和 B 会映射到同一组，发生缓存冲突，互相刷出缓存。对角线处在如图所示的 8x8 分块上：
![](/2023/01/10/CSAPP-Cache-Lab/image/cache_conflict_1.jpg)
这需要我们改变访存模式，我们可以用 8 个自由变量一次读完一行 A，减少 A 的缓存冲突：

```c
for (int i = 0; i < N; i += 8)
    for (int j = 0; j < M; j += 8) {
      for (int k = 0; k < 8; ++k) {
        a0 = A[i + k][j + 0];
        a1 = A[i + k][j + 1];
        a2 = A[i + k][j + 2];
        a3 = A[i + k][j + 3];
        a4 = A[i + k][j + 4];
        a5 = A[i + k][j + 5];
        a6 = A[i + k][j + 6];
        a7 = A[i + k][j + 7];

        B[j + 0][i + k] = a0;
        B[j + 1][i + k] = a1;
        B[j + 2][i + k] = a2;
        B[j + 3][i + k] = a3;
        B[j + 4][i + k] = a4;
        B[j + 5][i + k] = a5;
        B[j + 6][i + k] = a6;
        B[j + 7][i + k] = a7;
      }
    }
```

miss 数降到了 288，够满分了。

那我们再看一下 64x64 的：
![](/2023/01/10/CSAPP-Cache-Lab/image/cache_miss_1.png)
赫赫，4000 多个。究其原因在于随着矩阵的扩大，一个 cache 只能容下 4 行矩阵了，这就造成 8x8 分块的上半部分和下半部分也是冲突的，如图：
![](/2023/01/10/CSAPP-Cache-Lab/image/cache_conflict_2.jpg)
这就要求我们按照每次 4 个的方式处理矩阵。我直接画图吧：
![](/2023/01/10/CSAPP-Cache-Lab/image/transpose.jpg)

1. 把上半部分一行 A 切成两半，转移到 B 的左上和右上。第一步做完后，B 的右上角刚好是转置完成后左下角应该的样子。
2. 把 A 左下和右下的两个小竖排(分别标了'1'和'5')分别赋给自由变量 a0..3，a4..7。a0..3 与 B 右上角的一个小横排(标了数字'5')交换。
3. 把 a0..3 和 a4..7 全部赋给 B 的一整个横排。

在第 2、3 步，上下互不干扰，有效消除了 miss。这样子总 miss 数到了 1108，能够满分。

贴个代码：

```c
int tmp;
for (int i = 0; i < N; i += 8)
  for (int j = 0; j < M; j += 8) {
    for (int k = 0; k < 4; ++k) {
      a0 = A[i + k][j + 0];
      a1 = A[i + k][j + 1];
      a2 = A[i + k][j + 2];
      a3 = A[i + k][j + 3];
      a4 = A[i + k][j + 4];
      a5 = A[i + k][j + 5];
      a6 = A[i + k][j + 6];
      a7 = A[i + k][j + 7];

      B[j + 0][i + k] = a0;
      B[j + 0][i + k + 4] = a4;
      B[j + 1][i + k] = a1;
      B[j + 1][i + k + 4] = a5;
      B[j + 2][i + k] = a2;
      B[j + 2][i + k + 4] = a6;
      B[j + 3][i + k] = a3;
      B[j + 3][i + k + 4] = a7;
    }
    for (int k = 0; k < 4; ++k) {
      a0 = A[i + 4][j + k];
      a4 = A[i + 4][j + k + 4];
      a1 = A[i + 5][j + k];
      a5 = A[i + 5][j + k + 4];
      a2 = A[i + 6][j + k];
      a6 = A[i + 6][j + k + 4];
      a3 = A[i + 7][j + k];
      a7 = A[i + 7][j + k + 4];

      tmp = a0;
      a0 = B[j + k][i + 4 + 0];
      B[j + k][i + 4 + 0] = tmp;
      tmp = a1;
      a1 = B[j + k][i + 4 + 1];
      B[j + k][i + 4 + 1] = tmp;
      tmp = a2;
      a2 = B[j + k][i + 4 + 2];
      B[j + k][i + 4 + 2] = tmp;
      tmp = a3;
      a3 = B[j + k][i + 4 + 3];
      B[j + k][i + 4 + 3] = tmp;

      B[j + 4 + k][i + 0] = a0;
      B[j + 4 + k][i + 1] = a1;
      B[j + 4 + k][i + 2] = a2;
      B[j + 4 + k][i + 3] = a3;
      B[j + 4 + k][i + 4] = a4;
      B[j + 4 + k][i + 5] = a5;
      B[j + 4 + k][i + 6] = a6;
      B[j + 4 + k][i + 7] = a7;
    }
  }
```

## 总结

cache 这方面的内容是真的很有难度，很多时候 cache conflict 只能手动分析、手动解决，非常令人头疼。
