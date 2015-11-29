---
layout: post
title: SearchPrompt:基于redis的搜索提示 
---

搜索提示对于当代的搜索引擎已经是标配了。好的搜索提示可以让用户更懒，增加用户对搜索的依赖。对于搜索提示，会有一些特殊的设计需求:

* 速度足够快， 一般要小于用户输入一个字的间隔，大概需要在100ms以内给出结果
* 搜索结果能够自定义排序
* 对于中文搜索，提供拼音搜索可以使用户减少输入法的切换
* 简单的使用场景下提供前缀命中即可，但某些情况下需要提供分词命中(比如quora)
* 搜索的结果数量不需要太多
* 方便扩展

由于速度的要求，必然需要将搜索的term放到内存中，Trie树看上去是最适用的数据结构。然而Trie树单独并不能很好地处理好第二点排序的需求，而且在单独在内存中维护一个Trie树并不是一个快速的解决方案。


选择redis作为数据容器。首先redis作为一个内存数据库能很好地满足我们速度上的需求，其次利用sort-set结构能有效地完成数据的自定义排序、自动去重，而且redis来可以设置过期时间，设置持久化等功能，可以有效地控制内存的大小。

## 主要功能
* 支持前缀命中
* 支持拼音搜索
* 支持分词命中
* 自定义排序
* 支持自定义数据结果

## 使用方式 
### 创建SearchPrompt对象
{% highlight python %}
from search_prompt import SearchPrompt 
sp = SearchPrompt(scope='', redis_addr='localhost')

{% endhighlight %}

### 添加数据 
普通的搜索建议，关键词本身就是需要查询的结果，比如搜索 `游` 的时候，给出 `游品位` 这个搜索提示即可，真实的搜索行为发生在用户选择了关键词之后。这种情况下只需要一个包含了以term为key的字典对象里，我们叫做`item`好了，当`item`被添加到SearchPrompt中时，它会自动根据`term`的值生成一个`uid`，并将`item`整个数据保存在redis中。

{% highlight python %}
item = {'term': '游品位'} 
sp.add(item) 
sp.add({'term': '游戏'}, pinyin=True) 
sp.add({'term': '好品位'}, pinyin=True, seg=True) 
{% endhighlight %}

对搜索结果排序, 只需要在item中添加score, 
{% highlight python %}
item = {'term': '游品位', 'score':0}
{% endhighlight %}

有时候，当我们给出搜索建议的时候，并不仅仅希望只给出关键词本身。例如在应用商店里，搜索梦幻西游的时候，自动给出游戏的icon，简介和下载链接等等。这个时候我们只需要这样:
{% highlight python %}
item = {'term': '游品位 ', 'icon': 'http://test.com/test.png', 'brief':'', 'download_url': 'http://test.com/test.apk'}
sp.add(item)
{% endhighlight %}

如果想使用拼音搜索，只需要:
{% highlight python %}
sp.add(item, pinyin=True)
{% endhighlight %}
还可以分词, 如果你希望在搜索西游的时候也能返回 梦幻西游:
{% highlight python %}
sp.add(item,seg=True) 
{% endhighlight %}

### 搜索 
{% highlight python %}
print sp.search('游')  
print sp.search('you')
print sp.search('好品位', fuzzy=True, limit=1) 
{% endhighlight %}
输出结果
{% highlight python %}
[{u'term': u'游戏'}, {u'term': u'游品位'}]
[{u'term': u'游戏'}]
[{u'term': u'好品位'}]
{% endhighlight %}



### 删除关键词:
{% highlight python %}
sp.delete({'term': '游品位'})
{% endhighlight %}


## 原理
### 保存数据:
  默认情况下, SearchPrompt会根据term的值生成一个uid, 将item保存在以uid为key的hash结构中
  当item中包含uid字段时, 会以uid为key保存数据
    
{% highlight python %}
def add(self, item, pinyin=False, seg=False):
    self.item_check(item)

    term = item['term']
    score = item.get('score', 0)
    uid = item.get('uid')
    if not uid:
        uid = hashlib.md5(item['term'].encode('utf8')).hexdigest()
    self.redis.hset(self.db, uid, json.dumps(item))
{% endhighlight %}


### 分解添加的关键词term

 为一个关键词建立所有的搜索索引，比如关键词`游品位`，当我们要支持前缀匹配的适合，需要将关键词分解为 
> `游`
>
> `游品`
> 
> `游品位` 

{% highlight python %}
def prefixs_for_term(self, term, seg=False):
    term = term.lower()
    prefixs = []

    for index, word in enumerate(term):
        prefixs.append(term[:index+1])
    if seg: 
        words = jieba.cut(term)
        for word in words:
            prefixs.append(word)

    return prefixs
{% endhighlight %}

### 将关键词转成拼音:
 利用pypinyin 将`游品位` 转成拼音 `youpinwei`, 把`youpinwei`当作一个新的关键词分解

### 切词:
  利用jieba分词, 将关键词 `游品位` 切分成`游`, `品位`
        
{% highlight python %}
for prefix in self.prefixs_for_term(term, seg):
    self._index_prefix(prefix, uid, score=score)
    if pinyin:
        prefix_pinyin = ''.join(lazy_pinyin(prefix))
        self._index_prefix(prefix_pinyin, uid, score=score)
{% endhighlight %}


### 创建索引:
  为每一个分解结果创建一个sorted-set, 将uid根据score添加到有序队列中 
{% highlight python %}
def _index_prefix(self, prefix, uid, score=0):
    self.redis.sadd(self.index, prefix)
    self.redis.zadd(self._get_index_key(prefix), uid, score)
{% endhighlight %}


### 搜索:
  根据search的参数query找到对应的sorted-set。对于query还可以进行分词和转成拼音依次搜索, 然后将搜索结果集合并。

{% highlight python %}
def search(self, query, limit=5, fuzzy=False):
    if not query: return []
    if fuzzy:
        search_querys = self.normalize(query) 
    else:
        search_querys = [query]
    cache_key = self._get_index_key(('|').join(search_querys)) 
    if not self.redis.exists(cache_key):
        self.redis.zinterstore(cache_key, 
                map(lambda x:self._get_index_key(x), search_querys))
    ids = self.redis.zrevrange(cache_key, 0, limit)
    if not ids: return ids
    return map(lambda x:json.loads(x), self.redis.hmget(self.db, *ids))
{% endhighlight %}
        

