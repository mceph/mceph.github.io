# mceph.github.io

云计算知识分享

# 发布文章方法如下：

1. 在 `./_posts`目录下新建一个.md文件  
    文件的命名方式一般为：YYYY-MM-DD-ZZZ.md,其中YYYY表示年份，MM表示月份，DD表示天，ZZZ为文章的一个说明。  
    文件名称最好是全英文，以防引起不必要的错误。  
    例如：2017-06-15-ceph-installation.md

2. .md文件书写规范  
    .md文件一般分为head与body部分，都要满足markdown语法。head部分一般关系到文章的显示、分类等，这里简要  
   说明一下。

例如：

> \---<br>
> layout: post  
> title: RxBinding 学习笔记  
> category: android<br>
> tags: \[android, jni\]<br>
> \---

每篇文章的最顶部开头部分都固定采用如上格式，各字段解释如下。  
`layout`：这里固定为`post`;   
`title`：为文章的标题；   
`category`：为文章所属分类；  
`tags`：为文章的标签，可以支持多个标签，标签之间用逗号分割。

3. 图片存放位置  
   图片一般存在`assets/images`目录下

4. pdf等文件存放位置  
   pdf文件一般存放在`assets/files`目录下

5. 其他

注意: 文章必须保存为`utf-8 - NO BOM`格式，否则可能会导致文章显示不出来


