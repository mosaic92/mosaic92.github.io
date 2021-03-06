---
layout: post
title: "京东评论爬虫和词云展示"
date: 2016-04-16
categories: blog
tags: [JD,爬虫,词云]

---

## 写在前面

这本来是数据挖掘课上老师布置的作业，做完后便记录在这里。作业的要求是：利用爬虫抓取文本数据，进行文本分词，对分词后的数据进行高频词排名，并绘制词云。正巧自己最近在啃周志华老师的《机器学习》，所以，就决定爬京东上这本书的评论来做这个作业。   

## 爬虫

爬虫和清理数据用的的R中的RCurl包和XML包

``` R
library(RCurl)  
library(XML)
```



京东的评论网址一般是这样的：`http://club.jd.com/review/11867803-1-1-0.html`在抓取的时候，遇到的问题是：循环中每抓20页之后就报错，猜测应该是京东设置了什么反爬虫的机制吧。我的解决办法是暴力解决，不让抓就不停地抓，一直抓到为止，用了R中的try函数。

``` R
url <- paste0('http://club.jd.com/review/11867803-1-',1:26,'-0.html')
comment_all <- c()
date_all <- c()
nurl_all <- c()
for (i in 1:length(url)){
  htmldoc <- try(htmlParse(url[i],encoding='GBK'))
  while(inherits(htmldoc, 'try-error')){
    Sys.sleep(5)
    htmldoc <- try(htmlParse(url[i],encoding='GBK'))
  }
  rootdoc <- xmlRoot(htmldoc)
  comment <- xpathSApply(rootdoc, 
                   '//div[@class="item"]/div[@class="i-item"]/div[@class="comment-content"]', xmlValue)
  date <- xpathSApply(rootdoc,
                   '//div[@class="item"]/div[@class="i-item"]/div[@class="o-topic"]/span[@class="date-comment"]',xmlValue)
  nurl <- xpathSApply(rootdoc,
                   '//div[@class="item"]/div[@class="i-item"]/div[@class="btns"]/a[@class="btn-reply"]', xmlGetAttr,'href')
  
  comment_all <- c(comment_all,comment)
  date_all <- c(date_all,date)
  nurl_all <- c(nurl_all,nurl)
}  

commentf <- gsub('\\s','',comment_all)
datef <- as.POSIXct(date_all)
jd.df <- data.frame(nurl_all,datef,commentf)
```

## 文本分词

文本分词用的是R中的jibaR包

``` R
library(jiebaR)
```

首先，做分词，需要先用`worker()`建立分词引擎，这里设置为默认。然后，读入停止词，用`filter_segment()`过滤停止词。

``` R
cutter <- worker()
seg <- segment(commentf,jiebar=cutter)

stop_words2 <- read.table("~/stop_words.txt", encoding="UTF8", quote="\"", comment.char="")
stopwords <- as.vector(stop_words2$V1)
sgwd2 <- filter_segment(seg,stopwords)
```



## 词云

做词云图用的是R中的wordcloud包

``` R
library(wordcloud) 
```

首先把分词得到的结果中的数字和字母去掉，然后统计各词出现的频数，设置好调色盘，用`wordcloud()`绘制词云图（MAC用户注意绘图时中文字体的问题，自己设置指定一种字体，免得看到一大片框框）。

``` R
sgwd <- gsub("[0-9a-zA-Z]+?","",sgwd2) 
wdnum <- table(unlist(sgwd)) 
wdnum <- sort(wdnum,decreasing = T) 
wdfreq<- data.frame(words =names(wdnum), freq = wordsNum)
colors<-brewer.pal(9,"Set1") 
wordcloud(wdfreq[,1],wdfreq[,2],colors=colors,random.order=FALSE,family='STKaiti') 
```

最后，放一张词云图：
![](https://raw.githubusercontent.com/mosaic92/mosaic92.github.io/master/img/wordcloud.jpeg)







