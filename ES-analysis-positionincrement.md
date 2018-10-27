# Elasticsearch分词中的PositionIncrement何解?

题目中的描述不是十分准确，更多应该是Lucene的分词。Lucene用了很久，其版本更新也很快。在ES出来之后，直接使用Lucene的时候就比较少了，更多的就在ES框架下一站式完成，ES目前在项目中几乎占据了半壁江山。

ES的功能很强大，使用过程中，有一个问题是绕不过的：就是中文分词。这是至关重要的一个问题，直接影响搜索结果的准确和召回。

一般来讲，分词的问题本身目前解决的已经相当不错了，大家用的比较多的是jieba分词，还有清华、斯坦福、复旦等开源的中文分词。如果要在ES中使用jieba分词，就需要定制一个ES的分词插件，将jieba分词load到ES中。

几年之前，因为项目需要，我撸过一个简单的ES插件，在github上开源: [jieba分词ES插件](https://github.com/sing1ee/elasticsearch-jieba-plugin)，也有一些用户在使用，期间也在断断续续的更新。

其中的关键，通过阅读代码就会发现，在处理token的过程中，有以下属性需要处理：
> 
- CharTermAttribute
- OffsetAttribute
- TypeAttribute
- PositionIncrementAttribute

分别代表了分词的结果的最小单元：term，分词的offset：`startOffset`和`endOffset`，以及词性，例如word、或者数字、字母等等。

最后一个属性`PositionIncrementAttribute`比较难以理解，在特定的场合下才需要特殊的处理，大部分情况下默认的结果就可以，但在特定的场合下，会丢掉部分的文档。下文我们就详细解释这个属性，通过例子来说明这个是如何产生影响的，以及该如何解决。

我们先解释一下分词的结果，使用到的ES，以及插件版本如下：

>
- elasticsearch-6.4.0
- elasticsearch-jieba-plugin-6.4.0

安装好插件，启动ES：

```java
./bin/elasticsearch
```
有如下输出，则说明插件加载成功：

```java
...
[2018-10-26T23:04:12,572][INFO ][o.e.p.PluginsService     ] [z7z-6dR] loaded plugin [analysis-jieba]
...
```

准备好示例文档：

```java
现在 高级产品经理\n2003。4-2003。11 产品副经理\n向产品群经理汇报工作\  负责产品为：得普利麻\n2002。5-2003。3 产品副经理\n向产品群经理汇报工作\n负责推广产品为：精分（思瑞康），麻醉（得普利麻）
```

jieba包括两种分词模式：

> 
- index模式，适用于索引的分词，可以分词更多的term，照顾召回。
- search模式，适用于查询的分词，分词结果没有交叉，更多考虑的是准确率的方面。

我们验证一下分词插件，以及两种模式的影响，通过如下命令，我们先看看`search`模式的分词效果：

```java
curl -X GET "localhost:9200/_analyze" -H 'Content-Type: application/json' -d' { "tokenizer" : "jieba_search", "text" : "现在 高级产品经理\n2003。4-2003。11 产品副经理\n向产品群经理汇报工作\  负责产品为：得普利麻\n2002。5-2003。3 产品副经理\n向产品群经理汇报工作\n负责推广产品为：精分（思瑞康），麻醉（得普利麻）" }‘
```

查看输出：

```json
{
  "tokens": [
    {
      "token": "现在",
      "start_offset": 0,
      "end_offset": 2,
      "type": "word",
      "position": 0
    },
    {
      "token": " ",
      "start_offset": 2,
      "end_offset": 3,
      "type": "word",
      "position": 1
    },
    {
      "token": "高级",
      "start_offset": 3,
      "end_offset": 5,
      "type": "word",
      "position": 2
    },
    {
      "token": "产品",
      "start_offset": 5,
      "end_offset": 7,
      "type": "word",
      "position": 3
    },
    {
      "token": "经理",
      "start_offset": 7,
      "end_offset": 9,
      "type": "word",
      "position": 4
    },
    {
      "token": "\n",
      "start_offset": 9,
      "end_offset": 10,
      "type": "word",
      "position": 5
    },
    {
      "token": "2003",
      "start_offset": 10,
      "end_offset": 14,
      "type": "word",
      "position": 6
    },
    {
      "token": "。",
      "start_offset": 14,
      "end_offset": 15,
      "type": "word",
      "position": 7
    },
    {
      "token": "4",
      "start_offset": 15,
      "end_offset": 16,
      "type": "word",
      "position": 8
    },
    {
      "token": "-",
      "start_offset": 16,
      "end_offset": 17,
      "type": "word",
      "position": 9
    },
    {
      "token": "2003",
      "start_offset": 17,
      "end_offset": 21,
      "type": "word",
      "position": 10
    },
    {
      "token": "。",
      "start_offset": 21,
      "end_offset": 22,
      "type": "word",
      "position": 11
    },
    {
      "token": "11",
      "start_offset": 22,
      "end_offset": 24,
      "type": "word",
      "position": 12
    },
    {
      "token": " ",
      "start_offset": 24,
      "end_offset": 25,
      "type": "word",
      "position": 13
    },
    {
      "token": "产品",
      "start_offset": 25,
      "end_offset": 27,
      "type": "word",
      "position": 14
    },
    {
      "token": "副经理",
      "start_offset": 27,
      "end_offset": 30,
      "type": "word",
      "position": 15
    },
    {
      "token": "\n",
      "start_offset": 30,
      "end_offset": 31,
      "type": "word",
      "position": 16
    },
    {
      "token": "向",
      "start_offset": 31,
      "end_offset": 32,
      "type": "word",
      "position": 17
    },
    {
      "token": "产品",
      "start_offset": 32,
      "end_offset": 34,
      "type": "word",
      "position": 18
    },
    {
      "token": "群",
      "start_offset": 34,
      "end_offset": 35,
      "type": "word",
      "position": 19
    },
    {
      "token": "经理",
      "start_offset": 35,
      "end_offset": 37,
      "type": "word",
      "position": 20
    },
    {
      "token": "汇报工作",
      "start_offset": 37,
      "end_offset": 41,
      "type": "word",
      "position": 21
    },
    {
      "token": "\n",
      "start_offset": 41,
      "end_offset": 42,
      "type": "word",
      "position": 22
    },
    {
      "token": "负责",
      "start_offset": 42,
      "end_offset": 44,
      "type": "word",
      "position": 23
    },
    {
      "token": "产品",
      "start_offset": 44,
      "end_offset": 46,
      "type": "word",
      "position": 24
    },
    {
      "token": "为",
      "start_offset": 46,
      "end_offset": 47,
      "type": "word",
      "position": 25
    },
    {
      "token": "：",
      "start_offset": 47,
      "end_offset": 48,
      "type": "word",
      "position": 26
    },
    {
      "token": "得",
      "start_offset": 48,
      "end_offset": 49,
      "type": "word",
      "position": 27
    },
    {
      "token": "普利",
      "start_offset": 49,
      "end_offset": 51,
      "type": "word",
      "position": 28
    },
    {
      "token": "麻",
      "start_offset": 51,
      "end_offset": 52,
      "type": "word",
      "position": 29
    },
    {
      "token": "\n",
      "start_offset": 52,
      "end_offset": 53,
      "type": "word",
      "position": 30
    },
    {
      "token": "2002",
      "start_offset": 53,
      "end_offset": 57,
      "type": "word",
      "position": 31
    },
    {
      "token": "。",
      "start_offset": 57,
      "end_offset": 58,
      "type": "word",
      "position": 32
    },
    {
      "token": "5",
      "start_offset": 58,
      "end_offset": 59,
      "type": "word",
      "position": 33
    },
    {
      "token": "-",
      "start_offset": 59,
      "end_offset": 60,
      "type": "word",
      "position": 34
    },
    {
      "token": "2003",
      "start_offset": 60,
      "end_offset": 64,
      "type": "word",
      "position": 35
    },
    {
      "token": "。",
      "start_offset": 64,
      "end_offset": 65,
      "type": "word",
      "position": 36
    },
    {
      "token": "3",
      "start_offset": 65,
      "end_offset": 66,
      "type": "word",
      "position": 37
    },
    {
      "token": " ",
      "start_offset": 66,
      "end_offset": 67,
      "type": "word",
      "position": 38
    },
    {
      "token": "产品",
      "start_offset": 67,
      "end_offset": 69,
      "type": "word",
      "position": 39
    },
    {
      "token": "副经理",
      "start_offset": 69,
      "end_offset": 72,
      "type": "word",
      "position": 40
    },
    {
      "token": "\n",
      "start_offset": 72,
      "end_offset": 73,
      "type": "word",
      "position": 41
    },
    {
      "token": "向",
      "start_offset": 73,
      "end_offset": 74,
      "type": "word",
      "position": 42
    },
    {
      "token": "产品",
      "start_offset": 74,
      "end_offset": 76,
      "type": "word",
      "position": 43
    },
    {
      "token": "群",
      "start_offset": 76,
      "end_offset": 77,
      "type": "word",
      "position": 44
    },
    {
      "token": "经理",
      "start_offset": 77,
      "end_offset": 79,
      "type": "word",
      "position": 45
    },
    {
      "token": "汇报工作",
      "start_offset": 79,
      "end_offset": 83,
      "type": "word",
      "position": 46
    },
    {
      "token": "\n",
      "start_offset": 83,
      "end_offset": 84,
      "type": "word",
      "position": 47
    },
    {
      "token": "负责",
      "start_offset": 84,
      "end_offset": 86,
      "type": "word",
      "position": 48
    },
    {
      "token": "推广",
      "start_offset": 86,
      "end_offset": 88,
      "type": "word",
      "position": 49
    },
    {
      "token": "产品",
      "start_offset": 88,
      "end_offset": 90,
      "type": "word",
      "position": 50
    },
    {
      "token": "为",
      "start_offset": 90,
      "end_offset": 91,
      "type": "word",
      "position": 51
    },
    {
      "token": "：",
      "start_offset": 91,
      "end_offset": 92,
      "type": "word",
      "position": 52
    },
    {
      "token": "精分",
      "start_offset": 92,
      "end_offset": 94,
      "type": "word",
      "position": 53
    },
    {
      "token": "（",
      "start_offset": 94,
      "end_offset": 95,
      "type": "word",
      "position": 54
    },
    {
      "token": "思",
      "start_offset": 95,
      "end_offset": 96,
      "type": "word",
      "position": 55
    },
    {
      "token": "瑞康",
      "start_offset": 96,
      "end_offset": 98,
      "type": "word",
      "position": 56
    },
    {
      "token": "）",
      "start_offset": 98,
      "end_offset": 99,
      "type": "word",
      "position": 57
    },
    {
      "token": "，",
      "start_offset": 99,
      "end_offset": 100,
      "type": "word",
      "position": 58
    },
    {
      "token": "麻醉",
      "start_offset": 100,
      "end_offset": 102,
      "type": "word",
      "position": 59
    },
    {
      "token": "（",
      "start_offset": 102,
      "end_offset": 103,
      "type": "word",
      "position": 60
    },
    {
      "token": "得",
      "start_offset": 103,
      "end_offset": 104,
      "type": "word",
      "position": 61
    },
    {
      "token": "普利",
      "start_offset": 104,
      "end_offset": 106,
      "type": "word",
      "position": 62
    },
    {
      "token": "麻",
      "start_offset": 106,
      "end_offset": 107,
      "type": "word",
      "position": 63
    },
    {
      "token": "）",
      "start_offset": 107,
      "end_offset": 108,
      "type": "word",
      "position": 64
    }
  ]
}
```
分词结果中，token对应的就是term属性，start_offset和end_offset对应的就是Offset属性，type类似于词性。这几个都是比较好理解的，那么`position`是什么含义呢？通过观察：

> `position`是分词之后term/token的先对位置，代表了顺序和距离。

这个例子中`产品`和`副经理`是紧挨着的，中间没有间隔。也就意味着如下的查询

```json
{
	"query": {
		"match_phrase":{
			"field1": {
				"query": "产品经理",
				"slop": 0 
			}
		}
	}
}
```
能够搜到我们的示例文档。这里要注意，`slop`默认是0，可以不写。当`slop`要求为0的时候，就要求搜索词组`产品经理`在文档中连起来的，这个时候命中的是`产品经理`，而不是`产品|群|经理`，`|`表示token分割。如果设置`slop`为1，则`产品|群|经理`也会命中。`slop`的大小，就是`position`的大小差异。

看下`index`模式，要更加复杂，`PositionIncrement`的作用也是在这里体现。同样是上面的文本：

```java
curl -X GET "localhost:9200/_analyze" -H 'Content-Type: application/json' -d' { "tokenizer" : "jieba_index", "text" : "现在 高级产品经理\n2003。4-2003。11 产品副经理\n向产品群经理汇报工作\  负责产品为：得普利麻\n2002。5-2003。3 产品副经理\n向产品群经理汇报工作\n负责推广产品为：精分（思瑞康），麻醉（得普利麻）" }‘
```

结果如下，需要仔细对比和`search`的差异。

```json
{
  "tokens": [
    {
      "token": "现在",
      "start_offset": 0,
      "end_offset": 2,
      "type": "word",
      "position": 0
    },
    {
      "token": " ",
      "start_offset": 2,
      "end_offset": 3,
      "type": "word",
      "position": 1
    },
    {
      "token": "高级",
      "start_offset": 3,
      "end_offset": 5,
      "type": "word",
      "position": 2
    },
    {
      "token": "产品",
      "start_offset": 5,
      "end_offset": 7,
      "type": "word",
      "position": 3
    },
    {
      "token": "经理",
      "start_offset": 7,
      "end_offset": 9,
      "type": "word",
      "position": 4
    },
    {
      "token": "\n",
      "start_offset": 9,
      "end_offset": 10,
      "type": "word",
      "position": 5
    },
    {
      "token": "2003",
      "start_offset": 10,
      "end_offset": 14,
      "type": "word",
      "position": 6
    },
    {
      "token": "。",
      "start_offset": 14,
      "end_offset": 15,
      "type": "word",
      "position": 7
    },
    {
      "token": "4",
      "start_offset": 15,
      "end_offset": 16,
      "type": "word",
      "position": 8
    },
    {
      "token": "-",
      "start_offset": 16,
      "end_offset": 17,
      "type": "word",
      "position": 9
    },
    {
      "token": "2003",
      "start_offset": 17,
      "end_offset": 21,
      "type": "word",
      "position": 10
    },
    {
      "token": "。",
      "start_offset": 21,
      "end_offset": 22,
      "type": "word",
      "position": 11
    },
    {
      "token": "11",
      "start_offset": 22,
      "end_offset": 24,
      "type": "word",
      "position": 12
    },
    {
      "token": " ",
      "start_offset": 24,
      "end_offset": 25,
      "type": "word",
      "position": 13
    },
    {
      "token": "产品",
      "start_offset": 25,
      "end_offset": 27,
      "type": "word",
      "position": 14
    },
    {
      "token": "副经理",
      "start_offset": 27,
      "end_offset": 30,
      "type": "word",
      "position": 15
    },
    {
      "token": "经理",
      "start_offset": 28,
      "end_offset": 30,
      "type": "word",
      "position": 16
    },
    {
      "token": "\n",
      "start_offset": 30,
      "end_offset": 31,
      "type": "word",
      "position": 17
    },
    {
      "token": "向",
      "start_offset": 31,
      "end_offset": 32,
      "type": "word",
      "position": 18
    },
    {
      "token": "产品",
      "start_offset": 32,
      "end_offset": 34,
      "type": "word",
      "position": 19
    },
    {
      "token": "群",
      "start_offset": 34,
      "end_offset": 35,
      "type": "word",
      "position": 20
    },
    {
      "token": "经理",
      "start_offset": 35,
      "end_offset": 37,
      "type": "word",
      "position": 21
    },
    {
      "token": "汇报",
      "start_offset": 37,
      "end_offset": 39,
      "type": "word",
      "position": 22
    },
    {
      "token": "汇报工作",
      "start_offset": 37,
      "end_offset": 41,
      "type": "word",
      "position": 22
    },
    {
      "token": "工作",
      "start_offset": 39,
      "end_offset": 41,
      "type": "word",
      "position": 23
    },
    {
      "token": "\n",
      "start_offset": 41,
      "end_offset": 42,
      "type": "word",
      "position": 24
    },
    {
      "token": "负责",
      "start_offset": 42,
      "end_offset": 44,
      "type": "word",
      "position": 25
    },
    {
      "token": "产品",
      "start_offset": 44,
      "end_offset": 46,
      "type": "word",
      "position": 26
    },
    {
      "token": "为",
      "start_offset": 46,
      "end_offset": 47,
      "type": "word",
      "position": 27
    },
    {
      "token": "：",
      "start_offset": 47,
      "end_offset": 48,
      "type": "word",
      "position": 28
    },
    {
      "token": "得",
      "start_offset": 48,
      "end_offset": 49,
      "type": "word",
      "position": 29
    },
    {
      "token": "普利",
      "start_offset": 49,
      "end_offset": 51,
      "type": "word",
      "position": 30
    },
    {
      "token": "麻",
      "start_offset": 51,
      "end_offset": 52,
      "type": "word",
      "position": 31
    },
    {
      "token": "\n",
      "start_offset": 52,
      "end_offset": 53,
      "type": "word",
      "position": 32
    },
    {
      "token": "2002",
      "start_offset": 53,
      "end_offset": 57,
      "type": "word",
      "position": 33
    },
    {
      "token": "。",
      "start_offset": 57,
      "end_offset": 58,
      "type": "word",
      "position": 34
    },
    {
      "token": "5",
      "start_offset": 58,
      "end_offset": 59,
      "type": "word",
      "position": 35
    },
    {
      "token": "-",
      "start_offset": 59,
      "end_offset": 60,
      "type": "word",
      "position": 36
    },
    {
      "token": "2003",
      "start_offset": 60,
      "end_offset": 64,
      "type": "word",
      "position": 37
    },
    {
      "token": "。",
      "start_offset": 64,
      "end_offset": 65,
      "type": "word",
      "position": 38
    },
    {
      "token": "3",
      "start_offset": 65,
      "end_offset": 66,
      "type": "word",
      "position": 39
    },
    {
      "token": " ",
      "start_offset": 66,
      "end_offset": 67,
      "type": "word",
      "position": 40
    },
    {
      "token": "产品",
      "start_offset": 67,
      "end_offset": 69,
      "type": "word",
      "position": 41
    },
    {
      "token": "副经理",
      "start_offset": 69,
      "end_offset": 72,
      "type": "word",
      "position": 42
    },
    {
      "token": "经理",
      "start_offset": 70,
      "end_offset": 72,
      "type": "word",
      "position": 43
    },
    {
      "token": "\n",
      "start_offset": 72,
      "end_offset": 73,
      "type": "word",
      "position": 44
    },
    {
      "token": "向",
      "start_offset": 73,
      "end_offset": 74,
      "type": "word",
      "position": 45
    },
    {
      "token": "产品",
      "start_offset": 74,
      "end_offset": 76,
      "type": "word",
      "position": 46
    },
    {
      "token": "群",
      "start_offset": 76,
      "end_offset": 77,
      "type": "word",
      "position": 47
    },
    {
      "token": "经理",
      "start_offset": 77,
      "end_offset": 79,
      "type": "word",
      "position": 48
    },
    {
      "token": "汇报",
      "start_offset": 79,
      "end_offset": 81,
      "type": "word",
      "position": 49
    },
    {
      "token": "汇报工作",
      "start_offset": 79,
      "end_offset": 83,
      "type": "word",
      "position": 49
    },
    {
      "token": "工作",
      "start_offset": 81,
      "end_offset": 83,
      "type": "word",
      "position": 50
    },
    {
      "token": "\n",
      "start_offset": 83,
      "end_offset": 84,
      "type": "word",
      "position": 51
    },
    {
      "token": "负责",
      "start_offset": 84,
      "end_offset": 86,
      "type": "word",
      "position": 52
    },
    {
      "token": "推广",
      "start_offset": 86,
      "end_offset": 88,
      "type": "word",
      "position": 53
    },
    {
      "token": "产品",
      "start_offset": 88,
      "end_offset": 90,
      "type": "word",
      "position": 54
    },
    {
      "token": "为",
      "start_offset": 90,
      "end_offset": 91,
      "type": "word",
      "position": 55
    },
    {
      "token": "：",
      "start_offset": 91,
      "end_offset": 92,
      "type": "word",
      "position": 56
    },
    {
      "token": "精分",
      "start_offset": 92,
      "end_offset": 94,
      "type": "word",
      "position": 57
    },
    {
      "token": "（",
      "start_offset": 94,
      "end_offset": 95,
      "type": "word",
      "position": 58
    },
    {
      "token": "思",
      "start_offset": 95,
      "end_offset": 96,
      "type": "word",
      "position": 59
    },
    {
      "token": "瑞康",
      "start_offset": 96,
      "end_offset": 98,
      "type": "word",
      "position": 60
    },
    {
      "token": "）",
      "start_offset": 98,
      "end_offset": 99,
      "type": "word",
      "position": 61
    },
    {
      "token": "，",
      "start_offset": 99,
      "end_offset": 100,
      "type": "word",
      "position": 62
    },
    {
      "token": "麻醉",
      "start_offset": 100,
      "end_offset": 102,
      "type": "word",
      "position": 63
    },
    {
      "token": "（",
      "start_offset": 102,
      "end_offset": 103,
      "type": "word",
      "position": 64
    },
    {
      "token": "得",
      "start_offset": 103,
      "end_offset": 104,
      "type": "word",
      "position": 65
    },
    {
      "token": "普利",
      "start_offset": 104,
      "end_offset": 106,
      "type": "word",
      "position": 66
    },
    {
      "token": "麻",
      "start_offset": 106,
      "end_offset": 107,
      "type": "word",
      "position": 67
    },
    {
      "token": "）",
      "start_offset": 107,
      "end_offset": 108,
      "type": "word",
      "position": 68
    }
  ]
}
```
因为`index`模式的原因，`产品副经理`分为了`产品|副经理|经理`。这个时候，合理的`position`就十分重要了。通过我最新的插件的实现，这里的`position`分别是14,15,16。这是正确的，因为要正确处理下面的结果。

当我们执行如下搜索：

```json
{
	"query": {
		"match_phrase":{
			"field1": {
				"query": "产品经理"
			}
		}
	},
	"highlight" : {
        "fields" : {
            "field1" : {}
        }
    }
}
```
命中我们的示例文本，无间隔的`产品经理`可以命中，并且可以高亮，但是`产品副经理`没有命中，也没有高亮。

再看这个例子：

```json
{
	"query": {
		"match_phrase":{
			"field1": {
				"query": "产品经理",
				"slop": 2
			}
		}
	},
	"highlight" : {
        "fields" : {
            "field1" : {}
        }
    }
}
```
则，无间隔的`产品经理`可以命中，并且可以高亮；同时，`产品副经理`有命中，`产品`和`经理`分别高亮。这两个例子的差别，大家要细细体会。

那么如何正确的处理`position`呢，关键就在于`PositionIncrementAttribute`属性的处理，通常我们使用`search`模式类似的分词是不会遇到问题的，即使使用默认的`PositionIncrementAttribute`的实现：根据分词得到的token，每次`+1`，从而得到`position`。

但默认的实现，遇到如下的情况，就会出现问题：

示例文本：

```java
中国人民解放军胜利了。
```

如果采用默认的实现，则输出：

```json
{
  "tokens": [
    {
      "token": "中国",
      "start_offset": 0,
      "end_offset": 2,
      "type": "word",
      "position": 0
    },
    {
      "token": "中国人",
      "start_offset": 0,
      "end_offset": 3,
      "type": "word",
      "position": 1
    },
    {
      "token": "中国人民解放军",
      "start_offset": 0,
      "end_offset": 7,
      "type": "word",
      "position": 2
    },
    {
      "token": "国人",
      "start_offset": 1,
      "end_offset": 3,
      "type": "word",
      "position": 4
    },
    {
      "token": "人民",
      "start_offset": 2,
      "end_offset": 4,
      "type": "word",
      "position": 5
    },
    {
      "token": "解放",
      "start_offset": 4,
      "end_offset": 6,
      "type": "word",
      "position": 6
    },
    {
      "token": "解放军",
      "start_offset": 4,
      "end_offset": 7,
      "type": "word",
      "position": 7
    },
    {
      "token": "胜利",
      "start_offset": 7,
      "end_offset": 9,
      "type": "word",
      "position": 8
    },
    {
      "token": "了",
      "start_offset": 9,
      "end_offset": 10,
      "type": "word",
      "position": 9
    }
  ]
}
```
根据这样的`position`，我们如下的查询，就找不到这个示例文档，从而产生丢数据的现象。

```json
{
	"query": {
		"match_phrase":{
			"field1": {
				"query": "中国人民"
			}
		}
	},
	"highlight" : {
        "fields" : {
            "field1" : {}
        }
    }
}
```

本来`中国人民`在示例中是无间隔紧邻的，但是由于`position`解析的问题，直接导致`slop`已经变成了4，所以必须制定查询中的`slop`比较大，才能够返回正确的文档，但这里Rank也会受到影响。

看一下正确`position`的结果。

```json
{
  "tokens": [
    {
      "token": "中国",
      "start_offset": 0,
      "end_offset": 2,
      "type": "word",
      "position": 0
    },
    {
      "token": "中国人",
      "start_offset": 0,
      "end_offset": 3,
      "type": "word",
      "position": 0
    },
    {
      "token": "中国人民解放军",
      "start_offset": 0,
      "end_offset": 7,
      "type": "word",
      "position": 0
    },
    {
      "token": "国人",
      "start_offset": 1,
      "end_offset": 3,
      "type": "word",
      "position": 0
    },
    {
      "token": "人民",
      "start_offset": 2,
      "end_offset": 4,
      "type": "word",
      "position": 1
    },
    {
      "token": "解放",
      "start_offset": 4,
      "end_offset": 6,
      "type": "word",
      "position": 2
    },
    {
      "token": "解放军",
      "start_offset": 4,
      "end_offset": 7,
      "type": "word",
      "position": 2
    },
    {
      "token": "胜利",
      "start_offset": 7,
      "end_offset": 9,
      "type": "word",
      "position": 3
    },
    {
      "token": "了",
      "start_offset": 9,
      "end_offset": 10,
      "type": "word",
      "position": 4
    }
  ]
}
```
其中，`中国`是0，`人民`是1，就可以命中了。

基本上，在处理token的时候，要判断``是1，还是0。这里的Lucene实现机制不好，对于分词的实现约束比较多，并且只考虑了英文。现在的实现，优先考虑了召回。极个别情况，还是会有些准确率的问题。

另外一个层面，要从词的切分的角度处理，分词的结果应该提供一个最细粒度的、无交叉的切分，这个方式用来做索引，会比较好一些。那这样，默认的`PositionIncrement`也是能够满足需求的。接下来看看，`jieba`是否可以改造一下，支持第三种分词的模式：最细粒度的、无交叉的切分。


