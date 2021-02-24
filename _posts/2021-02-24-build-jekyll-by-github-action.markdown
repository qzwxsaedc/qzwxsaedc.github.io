---
layout: post
title: 利用GitHub Action将Jekyll部署在GitHub Pages上
categories: Program
tags: 
  - Github Action
  - Github Pages
  - Jekyll
---
*<a href="#start">直接前往正文</a>*  
博客写好了，就该挂在网站上了。最经济实惠的应该就是直接把网页挂在GitHub Pages上了。既然都上GitHub Pages了，不如直接用GitHub Action来实现自动化网站部署。

*然后GitHub Action我折腾了一下午……*

先说明一下，如果不需要GitHub Action的话，整个操作就非常简单:   
  1. 用`jekyll build`编译网页  
  2. 把`./_site`目录下的所有文件和文件夹上传到服务器或 GitHub/Gitee(如果要用GitHub Pages或Gitee Pages的话)。

或者更偷懒一点:
  直接在服务器上面运行`jekyll serve`。这是Jekyll内置的一个调试用服务器。

作为懒人加穷学生，把博客挂在GitHub Pages上面自然是最好的选择，但是手动编译再部署就很烦。正好GitHub Action免费，直接拿GitHub Action自动部署就完事了。

---

Jekyll的安装、网站项目的创建和GitHub Pages的创建就不再赘述，直接进入主题。

GitHub Action的自动化部署有两个思路，一是把项目和网站放在同一个仓库的不同分支里面，二是把项目和网站放在不同的仓库里面。第二种方案其实蛮麻烦的，这个在后面细说。

先稍微讲一下GitHub Action。这玩意儿可玩性蛮高的，而且服务器配置挺高，比国内阿某云某讯云啥的大方多了。GitHub Action是一类CI\CD工具，配置好了可以体验编译打包发布一条龙服务。具体怎么配置就不展开细讲了(其实就是我自己都没完全玩明白)，百度一搜一大堆，或者自己去看GitHub Action的官方文档也行。GitHub Action和其他CI工具比起来更大的一个优点是，GitHub本身就是一个代码共享的一个平台，也就是说，你可以直接用别人写好的东西，而不需要自己过多的去操心。

---

## <a name="start">正文从这里开始</a>

##### <s>没错，上面都是废话</s>

首先将网站项目push到GitHub Pages的仓库中。**据说**GitHub Pages只能指定master分支，但据我观察，好像可以选定其他的分支。我选择将项目传到build分支中。接下来就会遇到第一个小坑。

![img](/assets/img/2021-02-24-build-jekyll-by-github-action/9900BCF8F0F30CA4.png)
*<center style="font-size:70%">上面是一个标准的GitHub项目的标签栏</center>*

**按百度上你能搜到的大部分步骤**，你只需要点上图中的Action，然后创建一个workflow就行了。就像下图这样:  
![img](/assets/img/2021-02-24-build-jekyll-by-github-action/097CE708D4118C6C.png)
*<center style="font-size:70%">如果你的项目没啥问题的话，应该就是一样的</center>*

可以看到，GitHub其实是默认提供了一套CI模板的，这里是第二个坑，接下来会细讲。

点击`Set up this workflow`后，GitHub会自动把模板填充上，就像这样: 
![img](/assets/img/2021-02-24-build-jekyll-by-github-action/2C982A81BCEA3CA3.png)
*<center style="font-size:70%">仔细看这玩意儿到底是创建在哪个分支的</center>*

这里同时存在两个坑。上文所说的第一个坑是：这个文件其实是创建在master分支的。如果你的项目就在master分支的话，其实也没啥问题。但是如果你的项目不在master分支的话，直接创建配置文件的话，GitHub Action就没法正常运行，毕竟master分支现在啥都没有。

正确的操作是：在本地的项目目录中创建`./.github/workflows/WHATEVER_YOU_LIKE.yml`，然后在里面写CI的配置文件。
![img](/assets/img/2021-02-24-build-jekyll-by-github-action/0BF3EE28D3CA076D.png)
*<center style="font-size:70%">目录长这样</center>*

这里就出现了第二个坑：如果你把GitHub自动创建的模板复制进去的话，也没法自动构建。原因有三。
其一，触发器监听的是master分支，即只有当master分支有push和pr操作时才会触发构建。正确的配置是
{%highlight yaml%}
on:
  push:
    branches: [ build ] # 这里其实可以填写多个监听的分支，也可以不使用这个参数，监听所有push操作。
                        # 因为我需要自动构建的分支是build分支，所以这里填写build分支。
  pull_request:         # 这个可以不要。
    branches: [ build ] # 同on.push.branches。
{%endhighlight%}

其二，这里其实并没有将构建后的内容提交到仓库里面。所以需要一个提交的操作。下文细说。
其三，这个配置文件其实是有问题的。我遇到的问题有两个，第一个是提示Gemfile.lock文件无权访问，第二个是在后续的提交操作中提示没有对应的目录(这个是我调试的时候莫名其妙出现的问题，不知道是怎么触发的)。第一个问题将`chmod 777 /srv/jekyll`改成`chmod -R 777 srv.jekyll/`可解。

**<p sytle="font-size:150%">但是!</p>**这里可以不用这么复杂。

首先我们来稍微复习一下GitHub Action的特点。GitHub Action最大的特点就是可以用别人写的脚本，所以我们理论上能找到别人写好的构建脚本。实际上我确实找到了一个，还解决了另外一个问题：提交。这个脚本的地址是[https://github.com/marketplace/actions/jekyll-actions](https://github.com/marketplace/actions/jekyll-actions)。所以我们的jobs可以这么写了:
{%highlight yaml%}
{%raw%}
jobs:
  build:
    runs-on: ubuntu-latest # GitHub Action可以在Linux、MacOS、Windows环境下构建项目。这里使用的Linux环境。
    steps:
    - uses: actions/checkout@v2

    # Use GitHub Actions' cache to shorten build times and decrease load on servers
    - uses: actions/cache@v2
      with:
        path: vendor/bundle
        key: ${{ runner.os }}-gems-${{ hashFiles('**/Gemfile') }}
        restore-keys: |
          ${{ runner.os }}-gems-

    # https://github.com/marketplace/actions/jekyll-actions
    - uses: helaili/jekyll-action@v2
      with:
        token: ${{ secrets.GITHUB_TOKEN }}
        target_branch: 'master'
{%endraw%}
{%endhighlight%}

完整的配置文件:
{%highlight yaml%}
{%raw%}
name: Jekyll Site CI
on:
  push:
    branches: [ build ] # 这里其实可以填写多个监听的分支，也可以不使用这个参数，监听所有push操作。
                        # 因为我需要自动构建的分支是build分支，所以这里填写build分支。
  pull_request:         # 这个可以不要。
    branches: [ build ] # 同on.push.branches。

jobs:
  build:
    runs-on: ubuntu-latest # GitHub Action可以在Linux、MacOS、Windows环境下构建项目。这里使用的Linux环境。
    steps:
    - uses: actions/checkout@v2

    # Use GitHub Actions' cache to shorten build times and decrease load on servers
    - uses: actions/cache@v2
      with:
        path: vendor/bundle
        key: ${{ runner.os }}-gems-${{ hashFiles('**/Gemfile') }}
        restore-keys: |
          ${{ runner.os }}-gems-

    # https://github.com/marketplace/actions/jekyll-actions
    - uses: helaili/jekyll-action@v2
      with:
        token: ${{ secrets.GITHUB_TOKEN }}
        target_branch: 'master'
{%endraw%}
{%endhighlight%}

push后就会自动构建，稍等一会儿，当看到commit id旁边出现绿勾时就说明任务完成了。

---

## 还没完全结束！

我选择GitHub Pages还有另外一个理由：GitHub Pages可以自定义域名，Gitee Pages就不行。虽然Gitee Pages Pro可以设置，但是那玩意儿个人账户甚至买不了。

![img](/assets/img/2021-02-24-build-jekyll-by-github-action/9900BCF8F0F30CA4.png)
*<center style="font-size:70%">没错，还是这张图</center>*
步骤很简单。点上图中的Setting，然后往下翻到底，就能看到这个：
![img](/assets/img/2021-02-24-build-jekyll-by-github-action/4F7B2413E33E3434.png)
*<center style="font-size:70%">翻到底就能看到，一定要翻到底</center>*

我这里已经设置过了，所以url是自定义的域名。默认域名是`<username>.github.io`。在Custom domain栏填写你要设置的域名，然后点save，然后网站所在的分支就会创建一个CNAME文件。所以其实可以直接在本地创建一个，然后push上去。然后你就能用自己的域名了。

如果你要用自己的域名，你还需要把CNAME放在项目的根目录，要不然下次提交时会被移除。