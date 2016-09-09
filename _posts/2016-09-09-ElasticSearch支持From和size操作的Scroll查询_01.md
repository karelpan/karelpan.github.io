---
layout: post
title: ElasticSearch 支持 From 和 size 操作的Scroll查询 (1)
---

{{ page.title }}
================

<p class="meta">09 September 2016 - Su Zhou</p>

# 前言
from 和 size 查询对于非scroll 查询， elasticseach默认支持的非常好。
使用 Java Client 也可以非常方便的进行查询， 这里就不做冗余介绍了。

我们这里要探讨的是 基于 Scroll 的 from - size 查询。

## Scroll 查询
```
Scroll 是滚动查询，比如有的查询需要翻页，那么这个时候滚动查询就上场了。 
可以设置每次滚动查询的数量 scrollSize 比如 10000
```
* 除了翻页，为了保证Client的处理不会因以此查询太多，处理数据时内存溢出， 我们会采用scroll的方式查询大数据集，比如一次查询 3000000 的数据，并从查询的每条数据中提取处某几个字段， 这个时候 scroll 可以很大程度上在保证查询的稳定性（大大减小内存溢出的可能性）

## 主要处理点
```
from 和 size 的区间  和  每次 scroll 区间的 判断和控制
```

### 1. 获取实际需要 Scroll 的size， 最后一次Scroll 根据 from-size 和 scrollSize 的不同，实际需要查询的 realScrollSize 往往会小于设定的 scrollSize
```java
private static int generateRealScrollSize(int reScrollCount, int scrollSize, int from, int size) {
    int start       = scrollSize * reScrollCount;
    int end         = scrollSize * (reScrollCount + 1);
    int expectedEnd = from + size;

    if(size == 0) return scrollSize;

    if (expectedEnd >= end) {
        return scrollSize;
    } else {
        return expectedEnd - start;
    }
}
```

### 2. 使用 scroll 进行查询的主方法
```java
/**
 * 非阻塞读取ES， 结果处理方法 和 ES 返回解析方法， 使用模板归一化处理流程
 *
 * @param srb
 * @param scrollSize
 * @param from
 * @param size
 * @param clazz
 * @param handler
 * @param adaptor
 * @throws InterruptedException
 * @throws ExecutionException
 */
public static <E> void searchWithHandlerAndAdapter(final SearchRequestBuilder srb, final int scrollSize,
                                                   final int from, final int size,
                                                   Class<?> clazz,
                                                   final ElasticsearchResultCallback<E> handler,
                                                   final IElasticsearchHitsAdaptor<E> adaptor)
        throws InterruptedException, ExecutionException {
    final LinkedBlockingQueue<E> queue = new LinkedBlockingQueue<E>();

    // register
    final int maxIndex      = from + size - 1;
    int       reScrollCount = 0; // re-scroll count
    boolean   isFinished    = false;

    SearchResponse response = srb.setScroll(TimeValue.timeValueMinutes(Const.MAX_SCROLL_KEEPALIVE_MINUTES))
                                 .setSize(generateRealScrollSize(reScrollCount,
                                                                 scrollSize,
                                                                 from,
                                                                 size))
                                 .execute().actionGet();

    ExecutorService singleThread = Executors.newSingleThreadExecutor();
    try {
        while (true) {
            FutureTask<Void> future = null;
            SearchHit[] hits = response.getHits().getHits();


            // register {scroll.firstIndex, scroll.lastIndex}
            int[] scroll = new int[2];
            scroll[0] = reScrollCount * scrollSize;
            scroll[1] = scroll[0] + hits.length - 1;
            if (hits.length > 0) {
                if (from <= scroll[1]) { // from <= 本次 scroll 的 last 元素的 Index， 执行数据收集
                    // 使用 parser 解析， 解藕
                    List<E> items = adaptor.parse(hits, clazz);
                    int start = Math.max((from - scroll[0]), 0);
                    int end = 0;
                    if (size == 0) { // size == 0 读取从 from 处开始的所有
                        end = hits.length;
                        if (hits.length != scrollSize) isFinished = true;
                    } else { // size ！= 0 ， 开始检测结尾
                        if (maxIndex > scroll[1]) {
                            end = hits.length;
                            // 本次 scroll 的 hits < scrollSize，说明已经全部收集，结束收集
                            if (hits.length != scrollSize) isFinished = true;
                        } else { // MaxIndex < scroll.lastIndex, 说明抵达 需要收集的数据集的 最大数量，结束收集
                            end = (maxIndex - scroll[0]) + 1;
                            isFinished = true;
                        }
                    }
                    for (; start < end; start++) queue.add((E) items.get(start));

                    // 另起线程（无限Integer.Max size的阻塞队列）,处理本次从ES中获取的结果集
                    // 不阻塞当前从Elasticsearch读取的线程
                    singleThread.submit(future = new FutureTask<Void>(new Runnable() {
                        @Override
                        public void run() {
                            List<E> results = new ArrayList<E>();
                            queue.drainTo(results, scrollSize);
                            handler.doWithEachResult((List<E>) results);
                        }
                    }, null));
                }

                final String scrollId = response.getScrollId();
                response = null; // 释放句柄
                if (!isFinished) {
                    // 继续scroll 从ES获取结果
                    response = getClient().prepareSearchScroll(scrollId)
                                          .setScroll(TimeValue.timeValueMinutes(3)).execute().actionGet();
                    reScrollCount++;
                }
            } else {
                isFinished = true;
            }

            if (isFinished) {
                if (future != null) {// 从开始就没hits的情况
                    future.get(); // ES结果集读取结束，等待最后一个结果处理线程结束
                    future = null;
                }
                break;
            }
        }
    } finally {
        singleThread.shutdown(); // destroy thread service
        singleThread = null;
    }
}
```

#### 2.1 handler 接口定义
```java
import java.util.List;

/**
 * 
 * elasticsearch 查询结果批次回调处理类 , (即 handler)
 * 
 */
public interface ElasticsearchResultCallback<T> {

	void doWithEachResult(List<T> results);

}
```

#### 2.2 adaptor 接口定义
```java
import org.elasticsearch.search.SearchHit;

import java.util.List;

/**
 * 查询结果的 解析器模板
 * @param <T>
 * @author zhen.pan
 */
public interface IElasticsearchHitsAdaptor<T> {
    List<T> parse(SearchHit[] hits, Class<?> clazz);
}
```

### 3. 我们如何使用呢？ 
我会在， "支持 From 和 size 操作的Scroll查询 (2)" 中进行讲述

