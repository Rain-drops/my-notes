[参考](https://www.infoq.cn/article/database-timestamp-02/?utm_source=infoq&utm_medium=related_content_link&utm_campaign=relatedContent_articles_clk)

[转载](https://www.jianshu.com/p/4aea8af7a9ea?utm_campaign=haruki&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

## 一、概述

在**搜索引擎**中每个文件都对应一个文件 ID，文件内容被表示为一系列关键词的集合（实际上在搜索引擎索引库中，关键词也已经转换为关键词 ID）。例如“文档1”经过分词，提取了 20 个关键词，每个关键词都会记录它在文档中的出现次数和出现位置。

得到 **正向索引** 的结构如下：

​    “文档1”的ID > 单词1：出现次数，出现位置列表；单词2：出现次数，出现位置列表；…………。

​    “文档2”的ID > 此文档出现的关键词列表。

 ![img](https://images2015.cnblogs.com/blog/855959/201707/855959-20170706154309815-1724421988.png)

　　一般是通过key，去找value。

得到 **倒排索引** 的结构如下：

​    “关键词1”：“文档1”的ID，“文档2”的ID，…………。

​    “关键词2”：带有此关键词的文档ID列表。

 ![img](https://images2015.cnblogs.com/blog/855959/201707/855959-20170706154505378-610589524.png)

　　从词的关键字，去找文档。