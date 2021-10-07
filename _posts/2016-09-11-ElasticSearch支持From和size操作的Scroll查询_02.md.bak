---
layout: post
title: ElasticSearch 支持 From 和 size 操作的Scroll查询 (2)
---

{{ page.title }}
================

<p class="meta">11 September 2016 - Su Zhou</p>

# 前言
继上一篇 [ElasticSearch 支持 From 和 size 操作的Scroll查询(1)](http://karelpan.github.io/2016/09/09/ElasticSearch%E6%94%AF%E6%8C%81From%E5%92%8Csize%E6%93%8D%E4%BD%9C%E7%9A%84Scroll%E6%9F%A5%E8%AF%A2_01.html)

### 3.1 通用的处理Adaptor （有定制需求，可以实现一套自己的）
```java
import com.alibaba.fastjson.JSONArray;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.SearchHitField;
import org.json.JSONObject;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class CommonESFetchAdaptor implements IElasticsearchHitsAdaptor<JSONObject> {
    @Override
    public List<JSONObject> parse(SearchHit[] hits, Class<?> clazz) {
        final List<JSONObject> results = new ArrayList<>();
        try {
            for (SearchHit hit : hits) {
                final JSONObject item;
                if (hit.getSourceAsString() == null) {
                    item = new JSONObject();
                    for (Map.Entry<String, SearchHitField> entry : hit.getFields().entrySet()) {
                        int itemLen = entry.getValue().getValues().size();
                        if (itemLen > 1) {
                            final JSONArray child = new JSONArray();
                            for (int i = 0; i < itemLen; i++) {
                                child.add(entry.getValue().getValues().get(i));
                            }
                            item.put(entry.getKey(), child);
                        } else {
                            item.put(entry.getKey(), entry.getValue().getValue());
                        }
                    }
                } else {
                    item = new JSONObject(hit.getSourceAsString());
                }
                item.put("_index", hit.getIndex());
                item.put("_type", hit.getType());
                item.put("_id", hit.getId());
                if (hit.getScore() >= 0) {
                    item.put("_score", hit.getScore());
                }
                item.put("_version", hit.getVersion());
                results.add(item);
            }
            return results;
        } catch (Exception e) {
            return new ArrayList<>();
        }
    }
}
```


### 3.2 调用层，可以这么写
```java

<T, TFilterFields> List<T> fetchDataByQueryFromES(
    final SearchRequestBuilder searchRequestBuilder,
    final Integer scrollSize,
    final Integer from,
    final Integer size,
    final boolean isField,
    final Class<T> clazz,
    final String field,
    final TFilterFields filterFields
    ) throws Exception {

    // ...

    XXXX.searchWithHandlerAndAdapter(
        searchRequestBuilder,
        scrollSize,
        from,
        size,
        JSONObject.class,
        new ElasticsearchResultCallback<JSONObject>() {
            @Override
            public void doWithEachResult(List<JSONObject> results) {
                for (JSONObject result : results) {
                    try {
                        String key = CommonESResultUtils.buildMultiResultMapKey(result.getString("_index"), result.getString("_type"));
                        if (!map.containsKey(key)) map.put(key, new HashSet<T>());
                        if (isField) {
                            if (!result.has(field)) continue;
                            if (!JSONTypeMethodDictEnum.containsType(clazz.getName()))
                                throw new Exception("Not supported Type, @see JSONTypeMethodDictEnum");
                            map.get(key).add((T) MethodUtils.invokeMethod(result, JSONTypeMethodDictEnum.getMethod(clazz.getName()), field));
                        } else {
                            if (!clazz.getName().equals("org.json.JSONObject"))
                                throw new Exception("When it is not a field, only support Type org.json.JSONObject");
                            map.get(key).add((T) result);
                        }
                    } catch (Exception e) {
                        continue;
                    }
                }
            }
        },
        new CommonESFetchAdaptor());
    return new ArrayList<>(CommonESResultUtils.intersection(map)); 
    
}
```

在这里使用到了写的工具类 CommonESResultUtils 

```java
import com.alibaba.fastjson.JSON;
import com.google.common.collect.Sets;
import org.apache.commons.lang.StringUtils;

import java.util.*;

public class CommonESResultUtils {

    public static <T> Map<String, Set<T>> initMultiResultMap(String esIndices, String esTypes) throws Exception {
        final Map<String, Set<T>> map = new HashMap<>();
        for (String[] nexus : ESOrgJSONQueryBuilder.splitIndexTypeByESStructure(esIndices, esTypes))
            map.put((nexus[0] + "." + nexus[1]), new HashSet<T>());
        return map;
    }

    public static String buildMultiResultMapKey(String esIndex, String esType) {
        if(StringUtils.isEmpty(esIndex) || StringUtils.isEmpty(esType)) return null;

        return esIndex + "." + esType;
    }

    /**
     * Map 结果集取交集
     *
     *
     * @param map
     * @param <T>
     * @return
     * @throws Exception
     */
    public static <T> Set<T> intersection(final Map<String, Set<T>> map) throws Exception {
        Set<T> items = new HashSet<>();
        // 交集
        for (Set<T> set : map.values()) {
            if(items.size() == 0) {
                items = set;
                continue;
            }
            items = Sets.intersection(items, set);
            set.clear();
        }
        map.clear();
        return items;
    }

}


```

如 3.2， 这样就有了比较通用的处理基于Scroll查询的 from size 版本