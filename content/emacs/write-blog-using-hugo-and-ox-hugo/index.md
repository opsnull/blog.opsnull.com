---
title: "使用 hugo 和 ox-hugo 写博客"
author: ["opsnull"]
date: 2023-07-22T00:00:00+08:00
lastmod: 2023-08-20T17:51:25+08:00
tags: ["hugo", "org-mode", "blog"]
categories: ["emacs"]
draft: false
---

这篇文章总结下我使用 hugo 和 blowfish 主题搭建博客的配置参数，同时也记录了使用 org-mode 和 ox-hugo
来写博客的过程。

<!--more-->


## <span class="section-num">1</span> blowfish 主题 {#blowfish-主题}


### <span class="section-num">1.1</span> 安装和更新主题 {#安装和更新主题}

<https://blowfish.page/docs/installation/#install-hugo>

```shell
# 安装
hell new site mywebsite
cd mywebsite
git init
git submodule add -b main https://github.com/nunocoracao/blowfish.git themes/blowfish

# 更新
git submodule update --remote --merge
```


### <span class="section-num">1.2</span> 配置主题 {#配置主题}

从 theme 复制配置文件：

```shell
$ tree ~/blog/blog.opsnull.com/themes/blowfish/config
/Users/zhangjun/blog/blog.opsnull.com/themes/blowfish/config
└── _default
    ├── config.toml
    ├── languages.en.toml
    ├── markup.toml
    ├── menus.en.toml
    ├── module.toml
    └── params.toml

1 directory, 6 files
$
```

配置 config.toml: languageCode = "zh-CN"：与 blowfish/i18n 目录下的文件名前缀一致。

官方自带的 i18n/zh-CN.yaml 中的部分配置被注释了, 导致不能正确显示文件更新时间等内容, 需要修正。


### <span class="section-num">1.3</span> browser icon {#browser-icon}

自定义浏览器角标：在 favicon.io 将自己的图片生成为各种尺寸的 icon，直接解压在favicon.io下载好的icon压缩包，并放在/static目录下即可。

```shell
$ ls -l ~/blog/blog.opsnull.com/static/
total 332K
-rw-rw-r--  1 zhangjun  40K  5 17 13:42 android-chrome-192x192.png
-rw-rw-r--  1 zhangjun 227K  5 17 13:42 android-chrome-512x512.png
-rw-rw-r--  1 zhangjun  36K  5 17 13:42 apple-touch-icon.png
drwxr-xr-x  2 zhangjun   64  5 13 17:07 css/
-rw-rw-r--  1 zhangjun  771  5 17 13:42 favicon-16x16.png
-rw-rw-r--  1 zhangjun 2.1K  5 17 13:42 favicon-32x32.png
-rw-rw-r--  1 zhangjun  16K  5 17 13:42 favicon.ico
drwxr-xr-x  3 zhangjun   96  5 13 17:23 js/
drwxr-xr-x 85 zhangjun 2.7K  5 13 21:16 ox-hugo/
-rw-rw-r--  1 zhangjun  263  5 17 13:42 site.webmanifest
$
```


### <span class="section-num">1.4</span> custom icon {#custom-icon}

自定义 icon： 将自定义的 svg 文件放在 /asserts/icons 目录下，为了使 icon 和主题自适应，需要在 svg 文件中添加属性fill=“currentColor” 如下：

```yaml
<svg>
    <<path fill="currentColor" d="xxx"/>
</svg>
```

```shell
$ mkdir assets/icons
$ ls ~/Downloads/*.svg
/Users/zhangjun/Downloads/wechat.svg  /Users/zhangjun/Downloads/weibo.svg
$ mv ~/Downloads/*.svg  assets/icons/
$
```


### <span class="section-num">1.5</span> 内置 icon {#内置-icon}

<https://blowfish.page/samples/icons/>

Blowfish has built-in support for a number of FontAwesome 6 icons. These can be included in your website
through either the `icon partial` or `icon shortcode`.


### <span class="section-num">1.6</span> 调大文档内容宽度 {#调大文档内容宽度}

创建 assets/css/custom.css 文件，内容如下：

```css
.max-w-prose {
  max-width: 100%;
}
```


### <span class="section-num">1.7</span> firebase {#firebase}

<https://blowfish.page/docs/firebase-views/>

配置后，可以显示页面访问计数和 like 统计，但是国内被墙。


### <span class="section-num">1.8</span> baidu analytics {#baidu-analytics}

创建目录和文件：~/blog/blog.opsnull.com/layouts/partials/analytics/main.html

-   官方文档不对： layouts/partials/analytics.html，没有效果。

在百度搜索官网注册网站，获得分析代码，例如：

```html
<script>
var _hmt = _hmt || [];
(function() {
  var hm = document.createElement("script");
  hm.src = "https://hm.baidu.com/hm.js?XX";
  var s = document.getElementsByTagName("script")[0];
  s.parentNode.insertBefore(hm, s);
})();
</script>
```


### <span class="section-num">1.9</span> comment {#comment}

1.  utterances： <https://utteranc.es/>
2.  gisscus： 在 layouts/partials/comments.html 创建一个页面。在 <https://giscus.app/> 配置基于 github repo 的
    comment 评论系统，然后将自动生成的代码贴入上面的 comments.html页面。
3.  cusdis：

<!--listend-->

```html
<div id="cusdis_thread"
  data-host="https://cusdis.com"
  data-app-id="XXX"
  data-page-id="{{ .File.UniqueID  }}"
  data-page-url="{{ .Permalink }}"
  data-page-title="{{ .Title  }}"
></div>
<script async defer src="https://cusdis.com/js/cusdis.es.js"></script>
```


### <span class="section-num">1.10</span> 相关文章 {#相关文章}

在 config/_default/config.toml 文件中设置，主要是根据文件的 tags、categories、series 等标准来判断：

```toml
# 相关文档
[related]
  threshold = 0
  toLower = false

    [[related.indices]]
        name = "tags"
        weight = 100

    [[related.indices]]
        name = "categories"
        weight = 100

    [[related.indices]]
        name = "series"
        weight = 50

    [[related.indices]]
        name = "authors"
        weight = 20

    [[related.indices]]
        name = "date"
        weight = 10

    [[related.indices]]
      applyFilter = false
      name = 'fragmentrefs'
      type = 'fragments'
      weight = 10
```


### <span class="section-num">1.11</span> rss {#rss}

访问地址：<http://blog.opsnull.com/index.xml>


### <span class="section-num">1.12</span> TOC {#toc}

全局控制 TOC 显示级别：

```toml
[tableOfContents]
  startLevel = 1
  endLevel = 3 # 不包含 endLevel，所以只显示 1-2 级别。
```

文件级别控制是否显示：

-   showTableOfContents: true


### <span class="section-num">1.13</span> Branch Pages {#branch-pages}

List pages 和 Taxonomy Pages 是有差别的：

-   List 一般是因为 Branch Pages 引起的，显示的是该 branch 下所有文档列表；
-   Taxonomy 是一组预定义的列表，后续可以用于对文档指定 taxonomy 的 term，用于对文档进行归类。
-   某个 Taxonomy List 页面显示一组 term list；
-   terms list 页面显示归属到该 term 的文档列表。


### <span class="section-num">1.14</span> Homepage # {#homepage}

Layout:	layouts/index.html
Content:	content/_index.md


### <span class="section-num">1.15</span> List pages # {#list-pages}

Layout:	layouts/_default/list.html
Content:	content/../_index.md

```text
.
└── content
    └── projects
        ├── _index.md          # /projects
        ├── first-project.md   # /projects/first-project
        └── another-project
            ├── index.md       # /projects/another-project
            └── project.jpg
```

可以在 \_index.md  中设置适合该 branch 下的所有文档的通用参数：

```text
---
title: "Projects"
description: "Learn about some of my projects."
cascade:
  showReadingTime: false
---
This section contains all my current projects.
```

In this example, the special cascade parameter is being used to hide the reading time on any sub-pages within
this section. By doing this, any project pages will not have their reading time showing. This is a great way
to override default theme parameters for an entire section without having to include them in every individual
page.


### <span class="section-num">1.16</span> Taxonomy pages {#taxonomy-pages}

是比较特殊的 branch pages，因为他们需要预定义：

1.  编辑文件 config/_default/config.toml 中的 taxonomies
    -   key = value： key 是单数，values 是复数。

<!--listend-->

```toml
[taxonomies]
  tag = "tags"
  category = "categories"
  author = "authors"
  series = "series"
```

然后就可以在 font mattter 中指定复数的 taxonomies，例如：

```markdown
---
title: "Into the Lion's Den"
description: "This week we're learning about lions."
animals: ["lion", "cat"]
---
```

-   这在 animals taxonomy 下创建了两个 term：lion 和 cat

Although it’s not obvious at this point, Hugo will now be `generating list and term pages` for this new
taxonomy. By default the listing can be accessed at `/animals/` and the term pages can be found at `/animals/lion/`
and `/animals/cat/`.

The list page will `list all the terms` contained within the taxonomy. In this example, navigating to `/animals/`
will show a page that has links for “lion” and “cat” which take visitors to `the individual term pages`.

The term pages will `list all the pages contained within that term`. These term lists are essentially the same
as normal list pages and behave in much the same way.

In order to add custom content to taxonomy pages, simply create `_index.md` files in the content folder using
the taxonomy name as the sub-directory name.

```text
.
└── content
    └── animals
        ├── _index.md       # /animals
        └── lion
            └── _index.md   # /animals/lion

```

Anything in these content files will now be placed onto the generated taxonomy pages. As with other content,
the front matter variables can be used to override defaults. In this way you could have a tag named lion but
override the title to be “Lion”.

To see how this looks in reality, check out the tags taxonomy listing on this site.

tags 和 categories 都是预定义的 taxonomies。

自定义 list 页面如下：创建目录和文件：

1.  content/categories/_index.md
2.  content/tags/_index.md

title：可以设置页面标题。

```markdown
---
title: 分类列表
---
Blowfish 支持一些定制一些 taxonomies 页面。

Blowfish has full support for Hugo taxonomies and will adapt to any taxonomy set up. Taxonomy listings like this one also support custom content to be displayed above the list of terms.

---
```

分类页面定制：
content/categories/emacs/_index.md

```markdown
---
title: Emacs
---
Emacs 分类下的页面

可以自定义呀！

---
```


### <span class="section-num">1.17</span> Leaf Pages {#leaf-pages}


### <span class="section-num">1.18</span> single {#single}

Layout:	layouts/_default/single.html
Content (standalone):	content/../page-name.md
Content (bundled):	content/../page-name/index.md

Leaf pages in Hugo are basically `standard content pages`. They are defined as pages that `don’t contain any
sub-pages`. These could be things like `an about page`, or an `individual blog post` that lives in the blog section
of the website.

The most important thing to remember about leaf pages is that unlike branch pages, `leaf pages should be named
index.md without an underscore`. Leaf pages are also special in that they can be `grouped together at the top
level` of the section and named with a unique name.

```text
.
└── content
    └── blog  # section 名称，用于对 leaf page 进行 group
        ├── first-post.md     # /blog/first-post  # 访问时不带文件名后缀
        ├── second-post.md    # /blog/second-post
        └── third-post                   # bundle package，必须是 index.md 文件名
            ├── index.md      # /blog/third-post
            └── image.jpg

```

Leaf pages have a wide variety of front matter parameters that can be used to customise how they are
displayed.


### <span class="section-num">1.19</span> External links {#external-links}

-   点击时跳转到外部页面

<!--listend-->

```markdown
---
title: "My Medium post"
date: 2022-01-25
externalUrl: "https://medium.com/"
summary: "I wrote a post on Medium."
showReadingTime: false
_build:
  render: "false"
  list: "local"
---
```


### <span class="section-num">1.20</span> simple  pages {#simple-pages}

Layout:	layouts/_default/simple.html
Front Matter:	layout: "simple"

通过在 front matter 中添加配置 layout: "simple" 可以对当前页面启用 simple layout。

simple layout 默认会以 full-width template 来显示当前页面内容。

```text
---
title: "My landing page"
date: 2022-03-08
layout: "simple"
---
This page content is now full-width.
```


### <span class="section-num">1.21</span> 导航菜单 {#导航菜单}

<https://gohugo.io/content-management/menus/>

```yaml
---
menu:
  main:
    params:
      class: center
    parent: Products
    pre: <i class="fa-solid fa-code"></i>
    weight: 20
title: Software
---
```

位置（可以在博文的 fronter matter 部分设置）：

-   menu："main"  # header 导航菜单
-   menu："footer"  # 页末导航菜单

排序：

-   wight：值越大越靠右

文档级别：例如：在 header 导航栏显示一个 now 链接，点击时显示所在的文档：

```toml
# aritle front matter
menu:
  main:  # 头部
   name: now  # 菜单项名称
   weight: 40
```

文档级别可以是 目录 \_index.md 文件，例如：

```nil
# cat content/ebpf/_index.md

---
title: eBPF  #
menu:
  main:
    name: eBPF # 菜单项名称
    weight: 100
    identifier: "ebpf" # 必须配置该字段才显示
---

Blowfish 支持一些定制一些 taxonomies 页面。

Blowfish has full support for Hugo taxonomies and will adapt to any taxonomy set up. Taxonomy listings like this one also support custom content to be displayed above the list of terms.

---
```

后续，所有在该目录下创建的文件，都可以通过 `localhost:1313/ebpf/` 来访问

-   `/ebpf/` 是  contenxt 下的子目录名称。

站点级别： config/_default/menus.zh-cn.toml

```toml
[[main]]    # main 表示 header 导航菜单
  name = "标签"   # 菜单名称
  pageRef = "tags"  # 相对于 content 目录的子目录或文件，这里为 content/tags 目录，故显示为一个列表
  weight = 30

[[main]]
  pre = "github" # icon
  name = "Now"
  pageRef = "now.md"  # 引用 content/now.md 页面，可以不加文件名后缀
  weight = 40

[[footer]]  # footer menu
  name = "标签"
  pageRef = "tags"
  weight = 10

[[footer]]
  name = "分类"
  pageRef = "categories"
  weight = 20
```


## <span class="section-num">2</span> ox-hugo 设置 {#ox-hugo-设置}


### <span class="section-num">2.1</span> 输出文档 {#输出文档}

文档根目录：由 org-hugo-base-dir 配置，markdown 内容会保存到该目录的 contenxt/&lt;HUGO_SECTION&gt; 目录下。

ox-hugo 全局缺省配置：

```emacs-lisp
(use-package ox-hugo
  :after ox
  :config
  (setq org-hugo-base-dir (expand-file-name "~/blog/blog.opsnull.com"))
  (setq org-hugo-section "posts")
  (setq org-hugo-front-matter-format "yaml")
  (setq org-hugo-export-with-section-numbers t)
  (setq org-hugo-auto-set-lastmod t))
```

By 文档或目录配置：
HUGO_BASE_DIR:

-   By 文档：#+hugo_base_dir: ~/blog/blog.opsnull.com
-   By 目录：使用 .dir-locals.el 来设置 org-hugo-base-dir 变量，例如在 ~/docs 目录下创建 .dir-locals.el 文件，内容如下：

<!--listend-->

```emacs-lisp
;;; Directory Local Variables            -*- no-byte-compile: t -*-
;;; For more information see (info "(emacs) Directory Variables")

((yas/minor-mode . ((org-hugo-base-dir .  (expand-file-name ~/blog/local.view)))))
```

文档输出目录由 `HUGO_SECTION` 配置:

-   全局：(setq org-hugo-section "posts")
-   文件：#+hugo_section: posts
-   subtree：:EXPORT_HUGO_SECTION: shell
-   如果设置为 `/` 则表示的是 content/ 目录；
-   如果是 subtree，对于子 tree，可以通过设置 EXPORT_HUGO_SECTION_FRAG  来在父 tree 的路径下添加子目录, 例如：
    -   父 tree 设置 :EXPORT_HUGO_SECTION: a
    -   子 tree 设置 :EXPORT_HUGO_SECTION_FRAG: b, 则子 tree 的输出保存位置  content/a/b
-   参考：<https://ox-hugo.scripter.co/doc/hugo-section/>

输出文档名称：

1.  文件：#+titile 来指定；
2.  subtree：:EXPORT_FILE_NAME: 来指定；

注意：如果使用 POST Bundle，则文档输出为一个目录：

-   目录名称由 EXPORT_HUGO_BUNDLE 指定；
-   文档名称由 :EXPORT_FILE_NAME: 指定，但一般为 index 或 \_index（建议, 可以使输出文件名唯一）;
    -   如果未 EXPORT_FILE_NAME（不建议）， 则文件名称和源文件名称一致。

对于 subtree property :EXPORT_FILE_NAME:：

-   EXPORT_FILE_NAME 为 subtree 指定输出的 post file name
-   EXPORT_FILE_NAME 不能嵌套，也即一个子 subtree 不能再定义该 Property；
-   如果使用 branch/leaf 结构，EXPORT_FILE_NAME 只能用于 leaf nodes
    -   也即设置 EXPORT_FILE_NAME: index

发布命令：

-   发布 file 或 subtree：
    -   C-c C-e H H: org-hugo-export-wim-to-md
        -   If point is in `a valid Hugo post subtree`, export `that subtree` to a Hugo post in Markdown
            -   设置了 subtree property：EXPORT_FILE_NAME
        -   If the file is intended to `be exported as a whole` (i.e. has the `#+title` keyword), export the whole
            Org file to a Hugo post in Markdown.
    -   C-c C-e H A： org-hugo-export-wim-to-md :all-subtrees
        -   If the Org file has one or more `‘valid Hugo post subtrees’`, export them to Hugo posts in Markdown.
        -   If the file is intended to be exported as a whole (i.e. `no ‘valid Hugo post subtrees’ at all`, and
            has the `#+title` keyword), export the whole Org file to a Hugo post in Markdown.
-   只发布 file：
    -   C-c C-e H h：Export the Org file to a Hugo post in Markdown. This is same as calling the
        org-hugo-export-to-md function interactively.

建议：One post per Org subtree (preferred)

-   Export only the current post Org subtree, or
-   Export all valid Hugo post subtrees in a loop.

不建议：One post per Org file

-   This works but you won’t be able to leverage `Org-specific benefits like tag and property inheritance`, use of
    TODO states to translate to `post draft state, =auto weight calculation` for pages, `taxonomies and menu items`,
    etc.


### <span class="section-num">2.2</span> hugo front-matter {#hugo-front-matter}

hugo 支持两种 front matter 格式: yaml 或 toml:

-   file based

<!--listend-->

```text
#+hugo_front_matter_format: yaml
```

subtree based

```text
:PROPERTIES:
:EXPORT_HUGO_FRONT_MATTER_FORMAT: yaml
:END:
```

设置输出文件名:

-   file-base-exported: 需要设置  `#+title` ;
-   subtree-base-exported: 需要设置 `EXPORT_FILE_NAME`;
-   如果配置了 `#+hugo_bundle` 或者 :EXPORT_HUGO_BUNDLE:, 则使用它作为文章目录名, EXPORT_FILE_NAME 可以设置为 index
    或 \_index;

subtree-based:

| Hugo front-matter (TOML)           | Org                                     | Org description                                                            |
|------------------------------------|-----------------------------------------|----------------------------------------------------------------------------|
| `title = "foo"`                    | `* foo`                                 | Subtree heading                                                            |
| `date = 2017-09-11T14:32:00-04:00` | `CLOSED: [2017-09-11 Mon 14:32]`        | Auto-inserted `CLOSED` subtree property when switch to Org **DONE** state  |
| `date = 2017-07-24`                | `:EXPORT_DATE: 2017-07-24`              | Subtree property                                                           |
| `publishDate = 2018-01-26`         | `SCHEDULED: <2018-01-26 Fri>`           | Auto-inserted `SCHEDULED` subtree property using default `C-c C-s` binding |
| `publishDate = 2018-01-26`         | `:EXPORT_HUGO_PUBLISHDATE: 2018-01-26:` | Subtree property                                                           |
| `expiryDate = 2999-01-01`          | `:EXPORT_HUGO_EXPIRYDATE: 2999-01-01:`  | Subtree property                                                           |
| `lastmod = <current date>`         | `:EXPORT_HUGO_AUTO_SET_LASTMOD: t`      | Subtree property                                                           |
| `lastmod = <current date>`         | `#+hugo_auto_set_lastmod: t`            | Org keyword                                                                |
| `tags = ["toto", "zulu"]`          | `* foo :toto:zulu:`                     | Subtree heading tags                                                       |
| `categories = ["x", "y"]`          | `* foo :@x:@y:`                         | Subtree heading tags with `@` prefix                                       |
| `draft = true`                     | `* TODO foo`                            | Subtree heading Org TODO state set to `TODO`.                              |
| `draft = false`                    | `* foo` or `* DONE foo`                 | Subtree heading Org TODO state not set or set to `DONE`.                   |
| `weight = 123` (manual)            | `:EXPORT_HUGO_WEIGHT: 123`              | Manual setting of page weight                                              |
| `weight = 123` (auto-calc)         | `:EXPORT_HUGO_WEIGHT: auto`             | When set to `auto`, page weight is auto-calculated                         |
| `tags_weight = 123` (manual)       | `:EXPORT_HUGO_WEIGHT: :tags 123`        | Manual setting of _FOO_ taxonomy weight, by setting to `:FOO VALUE`        |
| `tags_weight = 123` (auto-calc)    | `:EXPORT_HUGO_WEIGHT: :tags auto`       | When set to `:FOO auto`, _FOO_ taxonomy weight is auto-calculated          |
| `weight = 123` (in `[menu.foo]`)   | `:EXPORT_HUGO_MENU: :menu foo`          | Menu weight is auto-calculated unless specified                            |

file-based-export:

| Hugo front-matter (TOML)         | Org                                  |
|----------------------------------|--------------------------------------|
| `title = "foo"`                  | `#+title: foo`                       |
| `date = 2017-07-24`              | `#+date: 2017-07-24`                 |
| `publishDate = 2018-01-26`       | `#+hugo_publishdate: 2018-01-26`     |
| `expiryDate = 2999-01-01`        | `#+hugo_expirydate: 2999-01-01`      |
| `lastmod = <current date>`       | `#+hugo_auto_set_lastmod: t`         |
| `tags = ["toto", "zulu"]`        | `#+hugo_tags: toto zulu`             |
| `categories = ["x", "y"]`        | `#+hugo_categories: x y`             |
| `draft = true`                   | `#+hugo_draft: true`                 |
| `draft = false`                  | `#+hugo_draft: false`                |
| `weight = 123`                   | `#+hugo_weight: 123`                 |
| `tags_weight = 123`              | `#+hugo_weight: :tags 123`           |
| `categories_weight = 123`        | `#+hugo_weight: :categories 123`     |
| `weight = 123` (in `[menu.foo]`) | `#+hugo_menu: :menu foo :weight 123` |

参考：

1.  <https://ox-hugo.scripter.co/doc/org-meta-data-to-hugo-front-matter/>


### <span class="section-num">2.3</span> Post Bundle 和 Thumbnail {#post-bundle-和-thumbnail}

使用 post bundle 可以将一篇文档发布为一个目录，这样后续可以在目录中添加 thumbnail&amp;hero 图片（feature\* 开头）或
background 图片（background\* 开头）。

-   EXPORT_HUGO_BUNDLE: 指定  post bundle 的目录名称；
-   EXPORT_FILE_NAME: 建议设置，这样放置源文件重命名后，输出多份，一般是固定的 index，；

<div class="verse">

,\*  DONE shell 历史 :<bash:lang:@lang:blog>:<br />
CLOSED: <span class="timestamp-wrapper"><span class="timestamp">[2023-05-07 Sun 16:10]</span></span><br />
:PROPERTIES:<br />
:EXPORT_DATE: <span class="timestamp-wrapper"><span class="timestamp">[2023-05-07 Sun 15:00]</span></span><br />
:EXPORT_HUGO_BUNDLE: 2023-05-07-shell-history<br />
:EXPORT_FILE_NAME: index<br />
:END:<br />
<br />
这篇博客分享下 UNIX/Linux shell 的历史。<br />
<br />
\#+hugo: more<br />

</div>

导出结果：

```shell
$ ls  2023-05-07-shell-history/
featured.png  images/  index.md
$ ls  2023-05-07-shell-history/images/shell_兼容和历史/
2023-05-07_18-23-36_screenshot.png
$
```


## <span class="section-num">3</span> 写 blog {#写-blog}


### <span class="section-num">3.1</span> datetime {#datetime}

缺省日志格式：org-hugo-date-format: "%Y-%m-%dT%T%z"，示例：2017-07-31T17:05:38-04:00

通过设置 #+hugo_auto_set_lastmod: t，ox-hugo 在输出的文档中使用当前 date 来创建或更新 lastmod 字段。

File-based

{{< figure src="images/datetime/2023-07-21_19-17-43_screenshot.png" width="400" >}}

Subtree-based Exports

1.  Date:
2.  CLOSED: <span class="timestamp-wrapper"><span class="timestamp">[2018-01-23 Tue 14:10]</span></span>

2.Publish Date

-   SCHEDULED: <span class="timestamp-wrapper"><span class="timestamp">&lt;2060-01-26 Mon&gt;</span></span>
-   Expire Date
-   This is set using the :EXPORT_HUGO_EXPIRYDATE: property.
-   Last modified
-   :EXPORT_HUGO_LASTMOD: property.
-   :EXPORT_HUGO_AUTO_SET_LASTMOD: to a non-nil 时 ox-hugo 自动设置该时间戳；
    -   文件级别：#+HUGO_AUTO_SET_LASTMOD: t


### <span class="section-num">3.2</span> image link {#image-link}

引用 $HUGO_BASE_DIR/static 目录下的图片，如 ~/hugo/static/images/foo.png：

-   发布后 ~/hugo/static 作为根目录, 对下面文件的引用使用 /images/foo.png；

Inline Image（直接显示图片）：

1.  Unhyperlinked（不可点击）： `[[/images/org-mode-unicorn-logo-200px.png]]`
2.  hyperlinked to an image（可以点击）：=[![](/images/org-mode-unicorn-logo-50px.png)](/images/org-mode-unicorn-logo-200px.png)=

显示图片链接：=[[/images/org-mode-unicorn-logo-200px.png][Click here to see org-mode-unicorn-logo-200px.png]=

引用 static 目录外的图片：

-   如果是位于 static 目录外，且扩展名为org-hugo-external-file-extensions-allowed-for-copying；
-   ox-hugo 会将它们拷贝到 static 目录。

<!--listend-->

```text
[[~/some-dir/static/images/foo.png]]
```

如果文件 source path 包含 `/static/` 则 ox-hugo 将该文件 copy 到 `$HUGO_BASE_DIR/static/` 目录时，会保留源文件路径中 `/static/` 后的结构:

{{< figure src="/images/写_blog/2023-07-23_14-57-33_screenshot.png" width="400" >}}

{{< figure src="images/写_blog/2023-07-23_14-56-28_screenshot.png" width="400" >}}

如果文件 source path 不包含 `/static/` 则会拷贝到 org-hugo-default-static-subdirectory-for-externals 子目录
(ox-hugo):

{{< figure src="/images/写_blog/2023-07-23_14-58-21_screenshot.png" width="400" >}}


### <span class="section-num">3.3</span> detail-summary {#detail-summary}

可以使用 #+details 嵌套的 #+summary 来对一部分内容进行合并和展开：

<div class="verse">

#+begin_example<br />
\#+begin_details<br />
\#+begin_summary<br />
Why is this in **green**?<br />
\#+end_summary<br />
You will learn that later below in section.<br />
\#+end_details<br />
\#+end_example<br />

</div>

效果如下：

<details>
<summary>Why is this in green?</summary>
<div class="details">

You will learn that later below in section.
</div>
</details>


### <span class="section-num">3.4</span> summary {#summary}

在文件级别设置 summary 和配置是否显示 summary：

<div class="verse">

:EXPORT_HUGO_CUSTOM_FRONT_MATTER: :summary "这篇博客分享下 UNIX/Linux shell 的历史。这篇博客分享下 UNIX/Linux shell 的历史。这篇博客分享下 UNIX/Linux shell 的历史。"<br />
:EXPORT_HUGO_CUSTOM_FRONT_MATTER+: :showSummary true<br />

</div>

或者在正文前使用 #+hugo: more 标记来分割 summary 和正文。


### <span class="section-num">3.5</span> Tags and Categories {#tags-and-categories}

Subtree-base-exports:

-   EXPORT_HUGO_TAGS
-   EXPORT_HUGO_CATEGORIES
-   tags:header line 上的 org tag,如 :tag1:tag2:
-   categories: header line 上的 @ 开头的 tag,如 :@cat1:@cat2:

File-base-exports:

1.  \#+hugo_tags: tag1 tag2
2.  \#+hugo_categories: cat1 cat2
3.  统一: #+filetags
    -   \#+filetags: tag1 tag2
    -   \#+filetags: @cat1 @cat2


### <span class="section-num">3.6</span> 不输出 header 内容 {#不输出-header-内容}

1.  默认由变量 `org-export-exclude-tags` 指定的 tags 来控制;
2.  可以在文件级别或者特定 header section 级别打上 `noexport` tag.


### <span class="section-num">3.7</span> Author {#author}

指定多个 #+author 来指定多个作者：

```text
#+author: FirstName LastName
#+author: FAuthor1 LAuthor1
#+author: FAuthor2 LAuthor2
```

或者使用 PROPERTY 格式，逗号分割：

```text
:PROPERTIES:
:EXPORT_AUTHOR: FAuthor1 LAuthor1, FAuthor2 LAuthor2
:END:
```

不输出 author：

```text
#+options: author:nil

:PROPERTIES:
:EXPORT_OPTIONS: author:nil
:END:
```

blowfish 不显示 Author：

-   Article 设置：showAuthor: false


### <span class="section-num">3.8</span> series {#series}

先定义一个 series taxonomies

```toml
[taxonomies]
  tag = "tags"
  category = "categories"
  author = "authors"
  series = "series"
```

Mark Articles

```markdown
series: ["Documentation"]
series_order: 11
```

org-mode 语法：

<div class="verse">

:EXPORT_HUGO_CUSTOM_FRONT_MATTER+: :series '("shell") :series_order 1<br />

</div>

然后可以通过 <http://localhost:1313/series/> 来访问。

Marking an article as part of a series will automatically display the series module as you see in this page
for example. You can choose whether that module starts opened or not using the article.seriesOpened global
variable in params.toml or the front-matter parameter seriesOpened to specify an override at the article
level.


### <span class="section-num">3.9</span> org tree properties {#org-tree-properties}

<div class="verse">

CLOSED: <span class="timestamp-wrapper"><span class="timestamp">[2023-05-07 Sun 16:10]</span></span><br />
<br />
:PROPERTIES:<br />
:EXPORT_DATE: <span class="timestamp-wrapper"><span class="timestamp">[2023-05-07 Sun 15:00]</span></span><br />
\# 生成目录结构<br />
:EXPORT_HUGO_MENU: :menu "main"<br />
\# 生成各种 key:value, key 用 : 开头;<br />
:EXPORT_HUGO_CUSTOM_FRONT_MATTER: :foo bar :baz zoo :alpha 1 :beta "two words" :gamma 10<br />
\# 生成 md 文件名<br />
:EXPORT_FILE_NAME: shell-history<br />
:END:<br />

</div>


### <span class="section-num">3.10</span> 保存时自动导出 {#保存时自动导出}

M-x org-hugo-auto-export-mode


### <span class="section-num">3.11</span> Custom Front-matter Parameters {#custom-front-matter-parameters}

Custom Front-matter Parameters 是对 hugo 没特殊意义，但是对 theme 有意义的配置：

-   subttee: `:EXPORT_HUGO_CUSTOM_FRONT_MATTER:` property
-   file： `#+hugo_custom_front_matter:`

subtree：

```text
:PROPERTIES:
:EXPORT_HUGO_CUSTOM_FRONT_MATTER: :key1 value1 :key2 value2  # 一行可以设置多个 key value
:EXPORT_HUGO_CUSTOM_FRONT_MATTER+: :key2 value2  # 多行的话，MATTER 后面加 +
:END:
```

file：

```text
#+hugo_custom_front_matter: :key1 value1
#+hugo_custom_front_matter: :key2 value2
```

list 语法：

```text
:PROPERTIES:
:EXPORT_HUGO_CUSTOM_FRONT_MATTER: :animals '(dog cat "penguin" "mountain gorilla") # list 语法
:EXPORT_HUGO_CUSTOM_FRONT_MATTER+: :integers '(123 -5 17 1_234)
:EXPORT_HUGO_CUSTOM_FRONT_MATTER+: :floats '(12.3 -5.0 -17E-6)
:EXPORT_HUGO_CUSTOM_FRONT_MATTER+: :booleans '(true false)
:END:
```

输出：

```toml
animals = ["dog", "cat", "penguin", "mountain gorilla"]
integers = [123, -5, 17, 1_234]
floats = [12.3, -5.0, -1.7e-05]
booleans = [true, false]
```

Map 语法：

```text
:PROPERTIES:
:EXPORT_HUGO_CUSTOM_FRONT_MATTER: :versions '((emacs . "27.0.50") (hugo . "0.48"))
:EXPORT_HUGO_CUSTOM_FRONT_MATTER+: :header '((image . "projects/Readingabook.jpg") (caption . "stay hungry, stay foolish"))
:EXPORT_HUGO_CUSTOM_FRONT_MATTER+: :collection '((animals . (dog cat "penguin" "mountain gorilla")) (integers . (123 -5 17 1_234)) (floats . (12.3 -5.0 -17E-6)) (booleans . (true false)))
:END:
```

输出：

```toml
[versions]
  emacs = "27.0.50"
  hugo = 0.48
[header]
  image = "projects/Readingabook.jpg"
  caption = "stay hungry, stay foolish"
[collection]
  animals = ["dog", "cat", "penguin", "mountain gorilla"]
  integers = [123, -5, 17, 1_234]
  floats = [12.3, -5.0, -1.7e-05]
  booleans = [true, false]
```

其他 map of map，可以使用 Extra front-matter。

配置 subtree 属性：

-   TOML （默认）：:EXPORT_HUGO_FRONT_MATTER_FORMAT: toml；
-   YAML： :EXPORT_HUGO_FRONT_MATTER_FORMAT: yaml
-   具体选 TOML 和 YAML 需要和 ox-hugo 导出时设置的 front matter 格式一致：
    -   emacs 变量 org-hugo-front-matter-format 配置。

<div class="verse">

,#+begin_src yaml :front_matter_extra t<br />
foo:<br />
&nbsp;&nbsp;- bar: 1<br />
&nbsp;&nbsp;&nbsp;&nbsp;zoo: abc<br />
&nbsp;&nbsp;- bar: 2<br />
&nbsp;&nbsp;&nbsp;&nbsp;zoo: def<br />
,#+end_src<br />

</div>

YAML 格式：

<div class="verse">

,\* Post with YAML front-matter<br />
:PROPERTIES:<br />
:EXPORT_FILE_NAME: extra-front-matter-yaml<br />
:EXPORT_HUGO_FRONT_MATTER_FORMAT: yaml<br />
:END:<br />
The contents of the `#+begin_src yaml :front_matter_extra t` YAML<br />
block here will get appended to the YAML front-matter.<br />
,#+begin_src yaml :front_matter_extra t<br />
foo:<br />
&nbsp;&nbsp;- bar: 1<br />
&nbsp;&nbsp;&nbsp;&nbsp;zoo: abc<br />
&nbsp;&nbsp;- bar: 2<br />
&nbsp;&nbsp;&nbsp;&nbsp;zoo: def<br />
,#+end_src<br />

</div>

blowfish theme 自定义的 front-matter 参数列表: <https://blowfish.page/docs/front-matter/>

示例:

```text
:EXPORT_HUGO_CUSTOM_FRONT_MATTER: :summary "这篇博客分享下 UNIX/Linux shell 的历史。这篇博客分享下 UNIX/Linux shell 的历史。这篇博客分享下 UNIX/Linux shell 的历史。"
:EXPORT_HUGO_CUSTOM_FRONT_MATTER+: :showSummary true
:EXPORT_HUGO_CUSTOM_FRONT_MATTER+: :series '("shell") :series_order 1
```

参考：<https://ox-hugo.scripter.co/doc/custom-front-matter/>


### <span class="section-num">3.12</span> 中文支持 {#中文支持}

导出时去掉文档中间的空格。

The locale is manually set to Chinese or Japanese by setting it to `zh or ja` using `#+hugo_locale:` keyword (or
`EXPORT_HUGO_LOCALE` property).


### <span class="section-num">3.13</span> menu front-matter {#menu-front-matter}

So on each Page, the user can specify the keys of the associated `Menu Entry` using the menu
front-matter.

{{< figure src="/images/写_blog/2023-07-23_21-14-55_screenshot.png" width="400" >}}

In Org mode, these Menu Entry keys are specified using the `:EXPORT_HUGO_MENU:` property
(subtree-based exports) or `#+hugo_menu:` keyword (file-based exports). They are set in this property
list form:

```text
:EXPORT_HUGO_MENU: :menu <menu name> <:key 1> <val 1> <:key 2> <val 2> ..
```

The `:menu key is mandatory` because that’s used to specify the current Page’s Menu Entry’s parent
Menu name.

-   menu name 两个类型：main 和 footer，分别表示页面头部和底部的导航。

{{< figure src="/images/写_blog/2023-07-23_21-17-22_screenshot.png" width="400" >}}

例如：

-   未指定 :parent 时，直接添加到 main 菜单栏中, 指定时 value 应该是父 menu 的 Name 或 Identifier 值。
-   未指定 :name 时，默认使用文档标题, 如 :EXPORT_FILE_NAME: 设置的值;
-   :identifier : 如果多个菜单项名称相同, 则需要配置该参数来区分. 如果未设置则默认为 :title 值;
-   :title : 默认为文档标题;

<div class="verse">

#+,#+hugo_menu: :menu main :parent emacs<br />

</div>

例如:

<div class="verse">

,\* Posts under the `main` Menu<br />
:PROPERTIES:<br />
:EXPORT_HUGO_MENU: :menu main<br />
:END:<br />
,\*\* Post 1<br />
:PROPERTIES:<br />
:EXPORT_FILE_NAME: post-1<br />
:END:<br />
,\*\* Post 2<br />
:PROPERTIES:<br />
:EXPORT_FILE_NAME: post-2<br />
:END:<br />

</div>

产生的 menu 如下:

<div class="verse">

[menu]<br />
&nbsp;&nbsp;[menu.main]<br />
&nbsp;&nbsp;&nbsp;&nbsp;weight = 3001<br />
&nbsp;&nbsp;&nbsp;&nbsp;identifier = "post-1"  # 来源于文档标题, 进而来源于 :EXPORT_FILE_NAME:<br />
<br />
[menu]<br />
&nbsp;&nbsp;[menu.main]<br />
&nbsp;&nbsp;&nbsp;&nbsp;weight = 3002<br />
&nbsp;&nbsp;&nbsp;&nbsp;identifier = "post-2"<br />

</div>

另一个例子:

<div class="verse">

,\* Parent subtree<br />
:PROPERTIES:<br />
:EXPORT_HUGO_MENU: :menu "something here" :parent posts<br />
:END:<br />
,\*\* Post 1<br />
:PROPERTIES:<br />
:EXPORT_FILE_NAME: foo<br />
:EXPORT_HUGO_MENU_OVERRIDE: :identifier "abc" :weight 100<br />
:END:<br />
,\*\* Post 2<br />
:PROPERTIES:<br />
:EXPORT_FILE_NAME: bar<br />
:EXPORT_HUGO_MENU_OVERRIDE: :weight 1<br />
:END:<br />
<br />

</div>

输出:

```text
[menu]
  [menu."something here"]
    parent = "posts"
    weight = 100
    identifier = "abc"

[menu]
  [menu."something here"]
    identifier = "post-2"
    parent = "posts"
    weight = 1
```

file-based 的带 parent ment 的例子：

```text
#+hugo_menu: :menu main :parent Emacs
```

参考：<https://ox-hugo.scripter.co/doc/menu-front-matter/>


## <span class="section-num">4</span> 其他主题 {#其他主题}

<https://github.com/adityatelange/hugo-PaperMod>

<https://github.com/xianmin/hugo-theme-jane/blob/master/README-zh.md>
