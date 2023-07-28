---
title: "GitHub Pages + Hugo搭建个人博客"
subtitle: ""
date: 2022-04-10T11:34:02+08:00
draft: false

tags:
- Github Pages
- Hugo
categories:
- tech
---

<!--more-->

## 0 前言

去年年底开始，就一直有写博客的想法，一方面来沉淀自己的技术，另一方面也希望能对他人有些小小的帮助。如今各种平台非常的多，但是不是广告铺天盖地，就是样式固定，不够个性化。最终决定借助Github Pages来搭建一个属于自己的个性化博客。

[Github Pages](https://pages.github.com/) 是由GitHub提供的一个免费构建网站的方式。通过创建名为`github用户名.github.io`的仓库，上传文件后，就可以通过 `http://github用户名.github.io`访问你的网站。

GIthub Pages默认使用[Jekyll](https://jekyllrb.com/)构建静态网站，由于Jekyll依赖于Ruby，并且个人对Ruby完全不熟悉，所以最终选择了基于Golang的静态网站生成器[Hugo](https://gohugo.io/)来生成静态网站，然后托管在Github上。



##  1 创建github仓库

创建一个 名为`github用户名.github.io`的仓库，`github用户名`填写自己的GIthub用户名。比如我，仓库名就是`dingyufan.github.io`。

![create-repository](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/create-repository.png)

创建之后，是一个空仓库，先放着暂时不管。



## 2 安装Golang

Windows系统安装Golang非常简单，[下载msi安装包](https://studygolang.com/dl)，直接安装即可。默认会安装到 `C:\Go` 目录下，同时`C:\Go\bin` 目录也会被添加到PATH 环境变量中。可以通过`go version`验证安装的版本。目前最新版本应该是go1.18，我是之前安装的老版本。

![微信截图_20220410185748](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20220410185748.png)



## 3 安装hugo

创建文件夹`C:\Hugo`、`C:\Hugo\bin`、`C:\Hugo\sites`。其中bin目录用来存放hugo可执行程序，sites目录则用来保存站点。

![微信截图_20220409125654](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20220409125654.png)



从Github下载[Hugo发布的安装包](https://github.com/gohugoio/hugo/releases)，注意要使用**extended版本**，我下载的就是 hugo_extended_0.96.0_Windows-64bit.zip

![微信截图_20220409130146](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20220409130146.png)



将下载的hugo解压，并把解压得到的内容放到 `C:\Hugo\bin`下。

![微信截图_20220409125710](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20220409125710.png)

为了方便使用hugo.exe，我们把`C:\Hugo\bin`添加到环境变量PATH中。

![image-20220410191852967](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/image-20220410191852967.png)





## 4 创建网站

准备完全后，就可以开始构建网站了。进入`C:\Hugo\sites`目录，使用`hugo new site 网站名`创建网站。

注意，为了方便关联Github仓库，建议网站名与Github仓库名一致，比如我站点名、Github仓库名都为 dingyufan.github.io。

![微信截图_20220409130406](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20220409130406.png)



## 5 修改主题

Hugo拥有非常丰富的主题，可以实现各种个性化需求。我这次选择的主题是 [FixIt](https://github.com/Lruihao/FixIt)。

安装主题的方式有很多

- 一：可以直接下载主题，放置在站点的 themes文件夹下。需要注意的是，下载的主题要放在themes下和主题同名的文件夹内。比如FixIt，放置在`C:\Hugo\sites\dingyufan.github.io\themes\FixIt`中，文件夹不存在则自行创建；

- 二：可以把主题克隆到 `themes` 目录。

  ```bash
  cd C:/Hugo/sites/dingyufan.github.io
  git clone https://github.com/Lruihao/FixIt.git themes/FixIt
  ```

- **三：初始化仓库，并将主题仓库作为你的网站目录的子模块**。

  ```bash
  cd C:/Hugo/sites/dingyufan.github.io
  git init
  git submodule add https://github.com/Lruihao/FixIt.git themes/FixIt
  ```

  

注意：**推荐使用第三种方式！推荐使用第三种方式！推荐使用第三种方式！**

否则之后Github Pages在构建时会有异常！<font color="red">`No url found for submodule path 'themes/FixIt' in .gitmodules`</font>



安装主题之后，如何选择使用该主题呢？修改站点下配置文件config.toml

![image-20220410195927557](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/image-20220410195927557.png)

修改其中的 theme 选项，同时可以根据[FitIt主题说明](https://fixit.lruihao.cn/zh-cn/theme-documentation-basics/)，设置更多个性化配置。

我的config.toml文件基础配置如下：

```toml
baseURL = "http://dingyufan.github.io/"
# [en, zh-cn, fr, ...] 设置默认的语言
defaultContentLanguage = "zh-cn"
# 网站语言，仅在这里 CN 大写
languageCode = "zh-CN"
# 是否包括中日韩文字
hasCJKLanguage = true
# 网站标题
title = "我的全新 Hugo 网站"
# 文章发布位置
publishDir = "docs"

# 更改使用 Hugo 构建网站时使用的默认主题
theme = "FixIt"

[params]
  # FixIt 主题版本
  version = "0.2.X"

[menu]
  [[menu.main]]
    identifier = "posts"
    # 你可以在名称（允许 HTML 格式）之前添加其他信息，例如图标
    pre = ""
    # 你可以在名称（允许 HTML 格式）之后添加其他信息，例如图标
    post = ""
    name = "文章"
    url = "/posts/"
    # 当你将鼠标悬停在此菜单链接上时，将显示的标题
    title = ""
    weight = 1
  [[menu.main]]
    identifier = "categories"
    pre = ""
    post = ""
    name = "分类"
    url = "/categories/"
    title = ""
    weight = 2
  [[menu.main]]
    identifier = "tags"
    pre = ""
    post = ""
    name = "标签"
    url = "/tags/"
    title = ""
    weight = 3

# Hugo 解析文档的配置
[markup]
  # 语法高亮设置 (https://gohugo.io/content-management/syntax-highlighting)
  [markup.highlight]
    # false 是必要的设置 (https://github.com/Lruihao/FixIt/issues/43)
    noClasses = false
```



## 6 写文章

创建文章。`hugo new posts/文件名.md`, 会在站点的 content/post路径下创建一个md文件。这就是要写的博文。

![微信截图_20220409191339](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20220409191339.png)

注意，默认创建的md文件是草稿。草稿在构建网站时是会被忽略的。所以**一定要在发布前，修改文件头中 draft为false**。

在里面写一行 hello github pages

![微信截图_20220409191751](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20220409191751.png)



可以通过`hugo serve`命令启动本地预览，通过localhost:1313即可访问预览。

![微信截图_20220409191826](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20220409191826.png)



看到预览效果

![微信截图_20220409191900](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20220409191900.png)



## 7 发布网站

Hugo默认会把构建后的网站放在 public 文件夹下，但是为了方用github统一管理 发布前、后的内容，我想要将构建后网站路径，修改为放在docs文件夹下。

可以通过`-d docs`指定构建后放在docs文件夹下。**也可以在config.toml中添加一行配置`publishDir = "docs"`，这样就不需要在构建时使用 -d ，直接执行 `hugo`就可以完成侯建并放入docs文件夹。**

![微信截图_20220409191950](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20220409191950.png)



构建网站之后，需要将网站托管到Github上，即 上传到刚开始创建的空仓库里。

```bash
git init
git add -A
git commit -m 'init'
git branch -M main
git remote add origin git@githubcom:dingyufan/dingyufan.github.io.git
git push -u origin main
```



## 8 Github Pages配置

提交代码之后，需要修改Github Pages配置。指定Github Pages发布docs文件夹下的内容。

![image-20220410202202488](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/image-20220410202202488.png)

这样，仓库里的内容就是hugo生成的site的全部内容，docs文件夹下则是构建后的静态页面。这样可以对这个site完整的进行版本控制。



## 9 绑定域名

在完成以上步骤后，已经可以通过 `http://dingyufan.github.io`访问到网站了。

Github Pages还支持自定义域名。我希望通能过自己的域名 `http://blog.dingyufan.cn` 访问

![image-20220410202804130](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/image-20220410202804130.png)

配置后，要在域名注册商那里修改 域名解析规则。添加CNAME记录类型。

![image-20220410203010269](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/image-20220410203010269.png)

待解析生效之后，即可通过自定义域名访问。
