---
title: 第一篇博客
date: 2022-12-15 09:46:57
tags:
  - 乐子
  - 还是乐子
categories:
  - essay
comments:
  - true
---

## 芝士我的第一篇博客

很好，现在开始发电

芜湖呼呼呼呼呼呼呼呼

我精神状态挺好的呀，我神状好挺态精的呀，精我态神的呀状好挺，好态我的精神呀挺状，状的我神呀精好态挺，挺我状精好态神的呀。对啊对啊，我没疯啊，我疯没啊，我啊疯没。上勾拳！下勾拳！左勾拳！扫堂腿！回旋踢！龙卷风摧毁停车场！羚羊蹬，山羊跳！乌鸦坐飞机！老鼠走迷宫！大象踢腿！愤怒的章鱼！

我精神状态挺好的呀，我神状好挺态精的呀，精我态神的呀状好挺，好态我的精神呀挺状，状的我神呀精好态挺，挺我状精好态神的呀。对啊对啊，我没疯啊，我疯没啊，我啊疯没。上勾拳！下勾拳！左勾拳！扫堂腿！回旋踢！龙卷风摧毁停车场！羚羊蹬，山羊跳！乌鸦坐飞机！老鼠走迷宫！大象踢腿！愤怒的章鱼！

我精神状态挺好的呀，我神状好挺态精的呀，精我态神的呀状好挺，好态我的精神呀挺状，状的我神呀精好态挺，挺我状精好态神的呀。对啊对啊，我没疯啊，我疯没啊，我啊疯没。上勾拳！下勾拳！左勾拳！扫堂腿！回旋踢！龙卷风摧毁停车场！羚羊蹬，山羊跳！乌鸦坐飞机！老鼠走迷宫！大象踢腿！愤怒的章鱼！

骗哥们可以，别把你自己也骗到了就行。哥们被你骗了真无所谓的，打个哈哈就过了。但希望你打完这段话后擦一下眼角，别让眼泪掉在手机屏幕上了就行。你说的这些话，哥们信一下也是没什么的。还能让你有个心里安慰，但这种话说出来骗骗兄弟就差不多得了，哥们信你一下也不会少块肉，但是你别搞得自己也当真了就行。哥们被你骗一下是真无所谓的，兄弟笑笑也就过去了。真不是哥们想要破你防，你擦擦眼泪好好想想，除了兄弟谁还会信你这些话？

骗哥们可以，别把你自己也骗到了就行。哥们被你骗了真无所谓的，打个哈哈就过了。但希望你打完这段话后擦一下眼角，别让眼泪掉在手机屏幕上了就行。你说的这些话，哥们信一下也是没什么的。还能让你有个心里安慰，但这种话说出来骗骗兄弟就差不多得了，哥们信你一下也不会少块肉，但是你别搞得自己也当真了就行。哥们被你骗一下是真无所谓的，兄弟笑笑也就过去了。真不是哥们想要破你防，你擦擦眼泪好好想想，除了兄弟谁还会信你这些话？

骗哥们可以，别把你自己也骗到了就行。哥们被你骗了真无所谓的，打个哈哈就过了。但希望你打完这段话后擦一下眼角，别让眼泪掉在手机屏幕上了就行。你说的这些话，哥们信一下也是没什么的。还能让你有个心里安慰，但这种话说出来骗骗兄弟就差不多得了，哥们信你一下也不会少块肉，但是你别搞得自己也当真了就行。哥们被你骗一下是真无所谓的，兄弟笑笑也就过去了。真不是哥们想要破你防，你擦擦眼泪好好想想，除了兄弟谁还会信你这些话？

骗哥们可以，别把你自己也骗到了就行。哥们被你骗了真无所谓的，打个哈哈就过了。但希望你打完这段话后擦一下眼角，别让眼泪掉在手机屏幕上了就行。你说的这些话，哥们信一下也是没什么的。还能让你有个心里安慰，但这种话说出来骗骗兄弟就差不多得了，哥们信你一下也不会少块肉，但是你别搞得自己也当真了就行。哥们被你骗一下是真无所谓的，兄弟笑笑也就过去了。真不是哥们想要破你防，你擦擦眼泪好好想想，除了兄弟谁还会信你这些话？

```cpp
#include <chrono>
#include <iostream>
#include <string>
#include <thread>
#include <vector>

using namespace std;

class Timer {
   public:
    Timer(const std::string &name) : timer_name_(name), start_(std::chrono::steady_clock::now()) {}

    ~Timer() {
        auto end = std::chrono::steady_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start_);
        std::cout << ">>> " << timer_name_ << ": " << duration.count() << "ms" << std::endl;
    }

   private:
    std::string timer_name_{};
    std::chrono::time_point<std::chrono::steady_clock> start_{};
};

#define CACHE_LINE_SIZE 8
#define COUNTERS_NUM 4
#define MAX_COUNTER_CNT (1073741824 / 8)
#define NO_FALSE_SHARING_IDX(idx) ((idx) * (CACHE_LINE_SIZE))

int counters[COUNTERS_NUM * CACHE_LINE_SIZE];

int main() {
    std::thread th[COUNTERS_NUM];
    for (int i = 0; i < COUNTERS_NUM; ++i) {
        th[i] = std::thread([=]() {
            // cout << "thread idx:" << i << endl;
            for (int j = 0; j < MAX_COUNTER_CNT; ++j) {
                ++counters[NO_FALSE_SHARING_IDX(i)];
                --counters[NO_FALSE_SHARING_IDX(i)];
                ++counters[NO_FALSE_SHARING_IDX(i)];
            }
        });
    }
    {
        Timer timer("counters timer");
        for (int i = 0; i < COUNTERS_NUM; ++i) {
            th[i].join();
        }
    }
    for (int i = 0; i < COUNTERS_NUM; ++i) {
        if (counters[NO_FALSE_SHARING_IDX(i)] != MAX_COUNTER_CNT) {
            cout << "thread " << i << " false!" << endl;
        }
    }

    return 0;
}
```

![](image/円.png)
