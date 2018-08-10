+++
title = "Hello World Again And use gutenberg"
date = 2018-08-05

[taxonomies]
categories = ["blog"]
tags = ["rust", "javascript", "nodejs"]
+++

在使用hexo写了寥寥几篇文章后,我还是放弃了这个著名的静态博客程序.
<!-- more -->

# 原因

hexo作为一个nodejs项目,运行需要安装node,然后运行

```bash
npm install hexo-cli -g
hexo init <folder>
cd <folder>
npm install
```

然而问题来了,我运行Windows10和Manjaro Linux双系统,写文章可能会发生在两个系统,同时Linux有ntfs-3g,Windows有ext2fs,那么我就可以使用同一个文件夹了吗,结果是不行的.

不同系统的node_modules内容略有不同,在Windows下不能使用Linux下的node_modules,通用的方法就是删了再运行`npm install`,然后等.

但npm缓慢的安装速度让我失去了信心,我决定放弃hexo.

# 选择


[https://www.staticgen.com/](https://www.staticgen.com/) 这个网站列出大量的静态网站生成器(不都是博客),其中不需要运行时或者虚拟机就能运行的有两个

 * [hugo](https://gohugo.io/)
 * [gutenberg](https://www.getgutenberg.io)

Hugo是Go写的,Gutenberg是Rust写的,由于我对Rust的偏好,我选择后者.但从功能、主题数量、流行程度上讲，Hugo更适合大众需求。

下面是gutenberg文档的简单翻译,原文[https://www.getgutenberg.io/documentation/](https://www.getgutenberg.io/documentation/)

# 部署

## 安装

从[GitHub Release页面](https://github.com/Keats/gutenberg/releases)下载二进制文件并解压,将文件夹添加到`PATH`环境变量.只需要一个文件即可运行.[官网](https://www.getgutenberg.io/documentation/getting-started/installation/)提供了各种包管理器的安装方法.

## 使用

```bash
$ gutenberg --help
gutenberg 0.4.1
Vincent Prouillet <prouillet.vincent@gmail.com>
A fast static site generator with everything built-in

USAGE:
    gutenberg [OPTIONS] <SUBCOMMAND>

FLAGS:
    -h, --help       Prints help information    
    -V, --version    Prints version information
OPTIONS:
    -c, --config <config>    Path to a config file other than config.toml [default: config.toml]

SUBCOMMANDS:
    build    Builds the site
    help     Prints this message or the help of the given subcommand(s)
    init     Create a new Gutenberg project
    serve    Serve the site. Rebuild and reload on change automatically
```

执行`gutenberg init blog`来初始化.执行`gutenberg serve`来预览,默认地址为[http://127.0.0.1:1111/](http://127.0.0.1:1111/).执行`gutenberg build`生成静态文件,输出到`public`文件夹.

## 文件夹结构

* config.toml 配置文件,
* content markdown文件夹
* sass 样式表文件夹
* static JavaScript与其他文件 
* templates 渲染模板文件夹
* themes 主题文件夹

[主题列表](https://www.getgutenberg.io/themes/)

主题文件夹下存放主题文件夹,配置文件中的主题名即此文件夹名,一个主题的文件夹结构与上面相同,多一个theme.toml.生成网站时,gutenberg同时使用网站和theme的`sass`、`static`和`templates`文件夹中的数据,但网站的优先级更高.

static文件夹中的文件会复制到输出文件夹的根目录,使用其中的文件需要使用绝对路径,例如`/static/logo.png`的正确链接是`/logo.png`

## config.toml

配置文件格式为[TOML](https://github.com/toml-lang/toml/blob/master/README.md),以下为建议配置,所有选项见[官方文档](https://www.getgutenberg.io/documentation/getting-started/configuration/)

```toml
# 网站域名,必选,以下均为可选
base_url = "https://xxx.github.io"

# 标题,用于RSS而不是网页
title = ""

# 介绍,用于RSS而不是网页
description = ""

# 主题名
theme = ""

# 使用代码高亮
highlight_code = true

# 代码高亮主题,列表见上面的官方文档
highlight_theme = "solarized-light"

# 生成RSS文件
generate_rss = false

# 是否编译 sass
compile_sass = true

# 生成搜索索引,支持搜索与否取决于主题
build_search_index = false

# 检查外部死链,极其耗时
check_external_links = false

# taxonomies是页面的分类方式,这些变量先在这里定义,随后会根据模板生成对应的分类页面
# 模板在theme或者templates下的name文件夹,需要
# `list.html`(分类列表)和`single.html`(每个分类下的文章列表)
# 定义格式:{name = "名称",paginate_by = 5, paginate_path = "page" rss = 是否生成RSS}
# paginate_by:文章列表一个页面的文章数量(可选,0为不分页),paginate_path含义见下文
# 建议如下
taxonomies = [
    {name = "tags"},
    {name = "categories"},
]

# 不对content文件夹中满足以下通配符的文件进行处理和渲染
ignored_content = ["*.{tmp,xlsx}", "*draft.md"]

# extra存放没有被程序定义的变量,可以被模板和主题使用
[extra]
```

## content文件夹

`content`文件夹有两个概念:`section`和`page`.`section`作为目录或者列表页,`page`则是作为文章页.

当一个文件夹下存在`_index.md`时会创建一个`section`.`section`文件夹下的md文件(除了`_index.md`)会成为`page`.

没有`_index.md`的文件夹不是`section`,但md文件仍会渲染成`page`,只能通过url访问.

主页显示的内容是content文件夹这个section,其渲染模板是`templates/index.html`.即使不存在`_index.md`,content文件夹也是section,但这时一些主题无法渲染,因此建议加入`/content/_index.md`

gutenberg要求所有markdown文件开头都有front-matter.front-matter为markdown开头使用+++包围的行,其中格式为TOML.`_index.md`的front-matter模板如下,其中的值为默认值,所有项目均可选.即使一个项目也没有,+++也不能省去.

```toml
+++
title = ""

description = ""

# 对 pages 按 日期"date" 或 优先度"weight" 排序,或不排序 "none" .weight越低越优先
sort_by = "none"

# 用于父section对子section排序
# 不像page,section的排序只能按weight,同weight的section顺序是随机的
# 越低越优先
weight = 0

# 渲染这个section使用的模板,注意content文件夹默认使用index.html
template = "section.html"

# 每页展示多少page
# 设为0标识不分页
paginate_by = 0

# 分页后在url中显示的表示页面的路径,页面数会显示在后面
# 例如默认的url是 page/1
paginate_path = "page"

# 是否在pages的标题上插入锚符号🔗,当鼠标悬浮在标题上时出现,便于复制锚链接
# 选项"left", "right" and "none"
insert_anchor_links = "none"

# 是否出现在搜索索引中
# config.toml中`build_search_index`需要设为true
in_search_index = true

# 是否渲染此section
# 当你只想在此section中存放page却不想显示列表时使用
render = true

# 在访问此section时重定向到另一url,默认为空不重定向.
# 原因同上,但你不想在访问此url时显示404
redirect_to = ""

# 额外的变量
[extra]
+++

# 通过section.content访问这里的内容
```

page也有Front-matter,格式如下.

```toml
+++
title = ""
description = ""

# 日期
# 支持2种格式:YYYY-MM-DD (2012-10-02) , RFC3339 (2002-10-02T15:00:00Z),注意没有括号引号
# 当 section 的 `sort_by` 为 `date`, `date`为空会导致不被渲染
date = 2018-08-05

# 优先度,越低越优先
# 当 section 的 `sort_by` 为 `weight`, `weight`为空会导致不被渲染
weight = 0

# 草稿
draft = false

# 当不为空时,不使用文件名作为url结尾,而是这里的值
# section仍然会出现在url中
slug = ""

# 设置url绝对路径
# 不以/开头
path = ""

# 当你想重定向其他url到此页面时,将那些url加入数组.格式同path,不以/开头
aliases = []

# 是否出现在搜索索引
in_search_index = true

# 渲染页面使用的模板,默认为page.html
template = "page.html"

# taxonomies,名称需要在`config.toml`中定义,值为字符串数组
# tags = ["rust", "web"]
[taxonomies]
categories = ["blog"]
tags = ["rust", "javascript", "nodejs"]

# 自定义变量
[extra]
+++

# 正文...
```

page的文件名(不含.md)和子section的文件夹名会成为url的一部分.但page的url可以通过Front-matter中的path设置改变

当一个文件夹下存在`index.md`时(且不能有`_index.md`)会创建一个与文件夹名相同的page,并属于上一级section.这常用于存放页面专用的资源文件,这些文件可以使用相对路径读取.出于维护性的考虑,不要在这种文件夹下放其他md文件或`_index.md`.

`section`可以无限嵌套,但实现效果取决于主题文件和模板.子`section`的渲染模板是`templates/section.html`,但部分主题没有此文件,也就不支持子`section`

gutenberg还支持在markdown中插入[shortcodes](https://www.getgutenberg.io/documentation/content/shortcodes/),例如输入以下内容会插入一个gist

{{ gist(url="https://gist.github.com/12101111/63b36d8bd9bfddbe72fc24dd4c1aaeff") }}

也支持[图像缩放](https://www.getgutenberg.io/documentation/content/image-processing/)

gutenberg使用[tera](https://tera.netlify.com/)处理模板.其语法类似Jinja2, Django templates, Liquid, Twig

## 自动部署

此博客使用CI自动部署,源代码存放在source分支.

```yml
before_script:
  # Download and unzip the gutenberg executable
  # Replace the version numbers in the URL by the version you want to use
  - curl -s -L https://github.com/Keats/gutenberg/releases/download/v0.4.1/gutenberg-v0.4.1-x86_64-unknown-linux-gnu.tar.gz | sudo tar xvzf - -C /usr/local/bin

script:
  - gutenberg build

# If you are using a different folder than `public` for the output directory, you will
# need to change the `gutenberg` command and the `ghp-import` path
after_success: |
  [ $TRAVIS_BRANCH = source ] &&
  [ $TRAVIS_PULL_REQUEST = false ] &&
  gutenberg build &&
  sudo pip install ghp-import &&
  ghp-import -b master -m "push by CI" -n public && 
  git push -fq https://${GH_TOKEN}@github.com/${TRAVIS_REPO_SLUG}.git master:master

branches:
  only:
    - source
```