公链实战系列18-IPFS从零到一搭建个人博客系统

简介
环境准备
博客系统搭建
clone代码
运行
添加文章
发布
总结
项目地址
参考
简介
近年来，技术呈现出了百花齐放的态势。其创新与发展速度让人心潮澎湃。当下各种新技术层出不穷，IPFS(星际文件系统）作为这众多新技术当中的一员，非常值得我们期待。 关于IPFS的介绍今天就不赘述了，今天主要是关于IPFS的应用。对于IPFS概念不清楚的同学可以查看我先前的文章下一代 Web 协议——IPFS (星际文件系统)

今天我们利用IPFS开发一个博客系统，最终效果如下图所示：


博客地址

环境准备
既然是开发，那么环境准备相当重要，我们需要安装并配置以下软件

git
go
ipfs
对于以上三者有问题的同学可以看我之前的文章Go语言环境准备和下一代 Web 协议——IPFS (星际文件系统)， 相信前面两片文章足以解决你在环境准备中遇到的问题。

博客系统搭建
clone代码
git clone git@github.com:csunny/ipfs_blog.git
运行
ipfs daemon

# in another terminal
ipfs add -r ipfs_blog
打开浏览器

http://localhost:8080/ipfs/QmbstVVrYr6baRjuKQzJqx34GPjNagrSXCM9HDTRvZSS7g
就可以看到我们开发的博客系统了。

添加文章
可以首先写好文章，保存为txt或者其他你想要的格式，然后添加文章到ipfs

ipfs add my_article.txt
将文章添加到代码中

 <div>
    <h3><a href="/ipfs/hash_value">你的文章标题</a></h3>
    <div>
        你的文章内容
        <br/>
        <span class="postmeta_author" style="border:blue" >Magic</span>
        <span class="postmeta_category"><a href="" rel="category" style="border:blue" >区块链</a></span>
        <span class="postmeta_time" style="border:blue" >2018-8-7</span>
    </div>
</div>
完成以上步骤，你的文章也就添加进去了。刷新浏览器页面查看结果。

发布
修改域名，将域名改为以下域名，然后访问，查看结果

http://gateway.ipfs.io/ipfs/QmbstVVrYr6baRjuKQzJqx34GPjNagrSXCM9HDTRvZSS7g
总结
利用ipfs开发一个博客系统非常简单，发布部署也非常简单。当然，这里面前提是我已经写好了一个blog的模版，大家只需要复用这个模版就可以有自己的博客系统了。 另外，我相信大家肯定不会满足，那能不能利用ipfs建立网站或者企业站点呢？ 答案当然是肯定的，另外你也可以开发动态网站，关于动态网站的开发，如果不想写后端服务，利用公链来开发就是很好的选择。这里可以有无限的拓展。最近在读ipfs底层代码，同时也在写底层相关，兴趣使然，在这里拍砖引玉，希望大家能够有更好的作品出来。 整个项目代码都已经开源，开源地址嫌弃模版丑陋，想贡献代码的同学，欢迎提pull request.


