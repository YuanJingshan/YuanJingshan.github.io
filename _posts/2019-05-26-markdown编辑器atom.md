---
layout:     post
title:      python提取文档高频词
subtitle:   python提取文档高频词
date:       2019-05-26
author:     景山
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - python
    - 词云
---

# 问题分析
&emsp;最近项目上需要分析用户的留言，提用户的关注信息，进行舆论分析，以供决策人进行参考。实际上，这个任务就是提取留言高频词，再筛掉一下不是必要的词。  
# 实现思路
- 1）获取数据库留言信息保存文件。
- 2）读取留言内容，使用jieba进行分词，可根据任务需要可添加停用词、自定义词库。
- 3）collections词频统计。


&emsp;闲话不多说，直接上代码
```
#获取留言内容，保存文件
def get_letter_data():
    letter = pd.read_sql_query('select content from gzszf_app_letter_log', db.engine)
    letter.to_csv(LETTER_DICT, index=False)

#从文件提取高频词
letter_words = [x for x in jieba.cut(letter_content) if len(x) >= 2]  
words = list(map(lambda x: x[0], Counter(letter_words).most_common(10)))
```
