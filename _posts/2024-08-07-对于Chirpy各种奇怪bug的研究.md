---
title: 对于Chirpy各种奇怪bug的研究
date: 2024-08-07 22:55:00 +0800
categories: [网络]
tags: [chirpy, jekyll]
---

> READ THE DOCS!
>
> READ THE DOCS!
>
> READ THE DOCS!
>
> 这篇文章中的问题很大概率是因为你没有阅读官方的[说明](https://github.com/cotes2020/jekyll-theme-chirpy/wiki/Upgrade-Guide#upgrade-the-fork)导致的！
{: .prompt-warning }

## 可能包含的bugs

- 无法切换light/dark模式
- toc目录无法正常显示，即使已经全局启用
- 无法进行搜索
- 图片的加载动画（shimmer）永远不会消失

这些问题从建站起就已经困扰我非常久了，最开始我以为是因为`assets/lib`下的submodules没有clone，于是`git submodules update --init`，但是没有任何改变，而且苦于编译没有任何报错，因此一直搁置了这个问题（事实上，如果没有启动static assets功能，这些文件会自动从CDN获取，相关设置位于`_data/origin/cors.yml`）。直到今天，我在开发者工具中发现了这样的报错：

![image-20240807225004940](/posts/image-20240807225004940.png)

然而，如果你定位到编译好的网站目录下，会发现根本没有`assets/js/dist`这个文件夹，更别提对应的文件了。于是顺藤摸瓜找到了这个[discussion](https://github.com/cotes2020/jekyll-theme-chirpy/discussions/1257)：

> JS distribution files have been removed since [`v5.6.0`](https://github.com/cotes2020/jekyll-theme-chirpy/releases/tag/v5.6.0), and Bootstrap CSS has been lean since [`v7.0.0`](https://github.com/cotes2020/jekyll-theme-chirpy/releases/tag/v7.0.0). Therefore, for all future upgrades, you should compile the CSS/JS files yourself.

从5.6.0版本开始，Chirpy已经移除了js文件的分发，如果你的网站是以直接fork官方仓库的形式创建的，那么这些文件需要你自行编译，所以才会出现js文件缺失的问题（或许这也是为什么官方不推荐以这种方式创建）。

另外，这些也可能是jekyll框架的通病，参考[这里](https://blog.csdn.net/EliasChang/article/details/136921882)。

## 解决方法

1. [安装Node.js](https://nodejs.org)，这会同时为你的电脑配置npm环境；
2. cd到网站的工作目录，运行`npm install`，npm会自动读取packages.json内的文件并安装好需要的依赖（你可能需要先运行`npm install rollup`；如果出现冲突，你可能需要使用`--force`强制安装）；
3. 运行`npm run build`，npm会自动将相关文件编译好后拷贝到对应的文件夹内，此时`assets/js/dist`目录下应当以及有以下文件：
   - app.min.js
   - categories.min.js
   - commons.min.js
   - home.min.js
   - misc.min.js
   - page.min.js
   - post.min.js
   - sw.min.js
4. 检查根目录下的`.gitignore`内是否有`_sass/dist`或`assets/dist`，如果有，记得删除（这两条目录应当在你创建网站、运行`tools/init.sh`时就已经被删除；如果仍然存在，说明网站在部署时没有按照[标准流程](https://chirpy.cotes.page/posts/getting-started/)）；
5. 运行`jekyll build`，如果没有问题的话就可以正常push了。