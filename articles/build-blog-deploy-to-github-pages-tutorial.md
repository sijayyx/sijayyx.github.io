---
title: 构建Vue SPA博客并在GitHub Pages上部署
author: Sijayyx
date: 2026-10-22
summary: 本文介绍使用Vue从零构建个人博客，处理一系列关键技术问题，并自动部署到Github Pages的流程。
tags: [ 教程, Vue, Git ]
cover: /images/articles/c-lang-femboy.jpg
---

> 本教程假设您掌握一定的Vue和Git基本知识，熟悉Markdown编写，同时具有良好的网络环境以访问Github。为了类型规范，脚本语言使用TypeScript。
>
> 在完成之后，您将可以在Github Pages上部署自己的纯前端静态博客，并实现Markdown解析，深浅模式切换，404处理，自动构建与部署等功能。
>
> 本教程使用人脑生成。

## 开始之前

您可能早就想在网上写点东西，但是不太喜欢博客园/CSDN，想要搭建一个自己的博客。

一个个人博客的核心功能就做了两件事：存放文章，将文章展示给读者。实现它最简单的方式只需要一个后端服务和一个前端服务，并将它们部署在服务器上。如果您做过全栈开发，一定会明白我的意思——后端服务负责存储、检索数据，前端负责用户交互，将数据显示出来。

前后端部署的博客对于全栈开发者手到擒来，它的优点是，服务完全由自己维护，在中国大陆的响应速度较快，实现一些复杂的数据查询非常方便。缺点是，需要长期租用服务器以及域名。并不适合资金紧张，或者不想在上面产生不必要的开支的博主们。

如上文所述，博客只是存文章和展示文章而已。正因如此，静态博客可以方便地部署在免费静态网站托管服务上。Github、Cloudflare、Vercel等公司均提供这些服务。不过默认情况下，您的博客只能使用其网站的二级域名——这对您来说应该不是什么大问题，对吧？

> 您也可以在您的域名提供商处设置DNS解析，并在Github Pages中配置自定义域名。Github将为您签发免费的SSL证书。

静态博客需要您在构建时就将网页的**大部分**数据都包含在构建产物中。这个构建产物通常是一个 `index.html` 文件，以及它引用的 `.css`、`.js` 和一堆资源文件。有一部分数据，比如字体、着色器等资源，可以由用户浏览器从 CDN 处下载。

听起来略显复杂，不过有很多成熟的博客框架可以替代您做这些事情，比如 [Hexo](https://hexo.io/zh-cn/)、[WordPress](https://cn.wordpress.org/)、[VuePress](https://vuepress.vuejs.org/zh/) 等，GitHub Pages 也会使用默认的 Jekyll 为您构建静态网页。这些框架为您提供了方便的构建工作流、丰富的主题样式等。如果您只是想写写文章，或技术能力有限，那么它们十分适合您。

但是，如果您想高度自定义自己的博客（比如再塞个 HTML 小游戏进去），写自己的样式，或者想给自己一点挑战性的技术实践，您完全可以从基本的 Vue 框架开始，像开发前端项目一样一步一步搭建起自己的博客页面，只不过要处理一些额外的技术问题。让我们开始吧！

## 准备工作

### 创建Github Pages仓库

您需要一个Github账户。登录账户后，请创建一个新的仓库，仓库名必须为`您的用户名.github.io`，选择将此仓库设为公开。这个仓库是每个账户唯一的。

在仓库的Settings - Pages中，将发布分支设置为`main`，文件夹为`/root`，通常这是默认配置。

### 配置Vue项目

假设您已经创建了一个使用Vite作为构建工具，并使用TypeScript作为脚本语言。我们使用npm作为包管理器，当然您也可以使用您自己的包管理器。

我们将安装一系列构建依赖项，请使用`npm i`命令安装所需依赖。

- `markdown-it`: 解析Markdown文本
- `markdown-it-anchor`: 为Markdown文本生成锚点
- `markdown-it-container`: 插入自定义样式的容器
- `markdown-it-texmath`: 支持Markdown的Latex渲染
- `markdown-it-toc-done-right`: 为Markdown生成大纲（Table of Contents）
- `@highlightjs/vue-plugin`: 在文本中渲染代码块
- `highlight.js`: 为代码块提供高亮
- `js-yaml`: 解析YAML字符串
- `katex`: 解析Latex

> 推荐您安装`vue-router`作为路由组件，您也可以自行处理页面路由。

您还需要安装一些开发依赖项，请使用`npm i -D`命令安装。

- `gray-matter`: 解析Markdown的Front Matter
- `tsx`: 执行TypeScript脚本

## 开发页面

### 存储Markdown文本

Vite在构建时，会将`public/`下所有资源文件按照目录结构原样复制到构建产物的根目录中（与`index.html`同一层级）。**我们将所有Markdown文件放在`public/articles/`目录下**，在代码中通过`fetch`函数读取。在填写URL时，并不需要写完整的路径，而是将`/`作为资源根路径。

```ts
// 读取 public/articles/{name}.md
const fileName = encodeURIComponent(`${name}.md`)  
const res = await fetch(`/articles/${fileName}`) 

return await res.text()
```

### 处理Markdown元数据

文章列表页需要展示所有文章的页面。在构建前，我们通过执行一段脚本，读取所有Markdown文本的Front Matter中的元数据，将其输出为一个`info.json`文件，放在文章目录中。

Front Matter 是写在 Markdown 或文本文件最开头的一段结构化元数据，通常使用YAML编写。它的格式如下：

```yml
---
key1: val1
key2: val2
# 其余数据...
---

## 正文...
```

您可以填写如标题、日期、作者、标签等数据。Front Matter不会也不应该在页面中被展示出来。

在项目根目录下创建一个`/scripts/generate-index.ts`文件，作为解析Markdown元数据的脚本。核心代码如下：

```ts
import fs from 'node:fs'  
import path from 'node:path'  
import {fileURLToPath} from 'node:url'  
import matter from 'gray-matter'  

const __filename = fileURLToPath(import.meta.url)  
const __dirname = path.dirname(__filename)  
  
// 项目根目录  
const ROOT = path.resolve(__dirname, '..')  
const ARTICLES_DIR = path.join(ROOT, 'public', 'articles')  
const OUTPUT_FILE = path.join(ROOT, 'public', 'articles', 'info.json')

// 这里的数据您可以自定义，但是要和Front Matter内匹配。
export interface ArticleMeta {  
    name?: string;  // 文件名
    title?: string  // 页面标题
    date?: string   // 日期
    summary?: string  // 摘要
    cover?: string   // 主图URL
    tags?: string[]  // 标签  
}

// .....读取所有.md，输出string[]......

// 解析Front Matter
function parseFrontMatter(filePath: string): ArticleMeta | null {  
    try {  
        const raw = fs.readFileSync(filePath, 'utf-8')  
        const {data} = matter(raw)  // 调用grey-matter
  
        if (!data.title) {  
            console.warn(`[warn] 缺少 title，跳过: ${filePath}`)  
            return null  
        }  
  
        // 处理 date：可能是 Date 对象、字符串、数字  
        let date = ''  
        if (data.date instanceof Date) {  
            date = data.date.toISOString().slice(0, 10)  
        } else if (data.date != null) {  
            date = String(data.date)  
        }  
  
        // 处理 tags：可能是数组、字符串  
        let tags: string[] = []  
        if (Array.isArray(data.tags)) {  
            tags = data.tags.map((t: unknown) => String(t))  
        } else if (typeof data.tags === 'string') {  
            tags = [data.tags]  
        }  
  
        // 文件名，去掉md后缀
        const name = path.basename(filePath, '.md')  
  
        return {  
            name,  
            title: String(data.title),  
            date,  
            summary: data.summary ? String(data.summary) : undefined,  
            cover: data.cover ? String(data.cover) : undefined,  
            tags,  
  
        }  
    } catch (err) {  
        console.error(`[error] 解析失败: ${filePath}`, err)  
        return null  
    }  
}

function main() {  
    const files = readFiles(ARTICLES_DIR) // 读取所有md  
  
    const articles: ArticleMeta[] = []  
    for (const file of files) {  
        const meta = parseFrontMatter(file)  
        if (meta) articles.push(meta)  
    }  
  
	// 您可在这里插入别的操作，比如排序
  
    const output = articles  
  
	// 写入info.json
    fs.mkdirSync(path.dirname(OUTPUT_FILE), {recursive: true})  
    fs.writeFileSync(OUTPUT_FILE, JSON.stringify(output, null, 2), 'utf-8')  
}  
  
main()
```

将`package.json`中的`"scripts"`值修改如下：

```json
"scripts": {  
  "gen:articles": "tsx scripts/generate-index.ts",  
  "build": "npm run gen:articles && vite build",  
  "dev": "npm run gen:articles && vite",
}
```

在执行`npm run build`时，脚本将在构建前执行。

在读取文章列表时，只需要：

```ts
let articles: ArticleMeta[] = []
const res = await fetch('/articles/info.json')  
articles = await res.json()
```

即可获得所有文章元数据。

### Markdown解析与渲染

`markdown-it`会将Markdown文本解析为HTML字符串，在此过程中，一系列插件可以影响最终的呈现效果。最终的HTML字符串将交由`v-html`处理。我们配置基础的Markdown解析、TOC、代码高亮、数学公式渲染。

```ts
const htmlContent = ref<string>('')  // md渲染出的html
const tocHtml = ref<string>('') // toc的html

const md = new MarkdownIt({  
    breaks: true,  
    linkify: true,  
    // 代码高亮
    highlight(str: string, lang: string): string {  
        if (lang && hljs.getLanguage(lang)) {  
            try {  
                return (  
                        '<pre class="hljs codeblock"><code>' +  
                        hljs.highlight(str, {language: lang, ignoreIllegals: true}).value +  
                        '</code></pre>'  
                )  
            } catch (__) {  
                // 高亮失败  
            }  
        }  
        return '<pre class="hljs"><code>' + md.utils.escapeHtml(str) + '</code></pre>'  
    },  
})  
  
const slugify = (s: string): string =>  
        encodeURIComponent(String(s).trim().toLowerCase().replace(/\s+/g, '-')) 
         
// 配置 anchor 插件  
md.use(markdownItAnchor, {  
    permalink: false,  
    slugify,  
})

// latex渲染支持，以$符号作为分隔符  
md.use(texmath, {engine: katex, delimiters: 'dollars'});  

// 配置 toc 插件  
md.use(markdownItTocDoneRight, {  
    placeholder: '\\${toc}',  // 占位符
    slugify,  
    containerClass: 'table-of-contents',  
    listType: 'ul',  
    level: [1, 2, 3],  
    callback: (html: string): void => {  
        tocHtml.value = html  
    },  
})
```

在页面结构中，将Markdown正文和TOC显示出来：

```vue
<div v-html="htmlContent"></div>
<div v-html="tocHtml"></div>
```

此外，您需要在文章页面引入用于代码高亮的CSS：

```ts
import 'highlight.js/styles/atom-one-dark.css';
```

在`main.ts`中使用Katex的CSS：

```ts
import 'katex/dist/katex.min.css'
```

如果您想调整正文样式，则必须在页面的`<styles scoped>`内使用`:deep()`选择器，对`htmlContent`所处的元素样式进行定义。比如我的样式是这么写的：

```css
/* 段落 */  
:deep(p) {  
    line-height: 1.6;  
    font-size: 18px;  
    margin: 0 0 1.25em;  
}  
  
/* 标题 */  
:deep(h1) {  
    font-size: 40px;  
    line-height: 1.15;  
    font-weight: 700;  
    color: var(--text-primary);  
    margin-top: 0;  
    margin-bottom: 0.5em;  
}
```

### 处理TOC与滚动

`markdown-it-anchor`会为每一个标题生成对应的ID，随后，`markdown-it-toc-done-right`将标题解析为嵌套的`<li>`列表，并使用`<a>`表示每一个标题。点击TOC条目时，浏览器通过获取正文中对应ID的位置，执行滚动函数。

为`tocHtml`的`<div>`添加点击函数：

```vue
<script>
const handleTocClick = (event: MouseEvent): void => {  
    const target = event.target as HTMLElement | null  
    if (!target || target.tagName !== 'A') return  
  
    const href = target.getAttribute('href')  
    if (!href || !href.startsWith('#')) return  
  
    event.preventDefault()  
    const elementId = href.slice(1)  // 去掉#，得到ID
    smoothScrollTo(elementId)  // 平滑滚动到ID对应的标题锚点处
  
    // 立即高亮  
    activeId.value = elementId  
  
    history.pushState(null, '', href)  
}
</script>

<div v-if="tocHtml" v-html="tocHtml" @click="handleTocClick"></div>
```

如果需要自定义TOC的HTML样式，您同样需要使用`:deep()`选择器。

```css
:deep(.table-of-contents ul) {  
    list-style: none;  
    padding: 0;  
    margin: 0;  
  
    li {  
        a {
        /* .....自定义样式....*/
        }
    }
}
```

### 处理路由与404界面

我们默认使用`vue-router`作为路由组件。配置您其它页面的路由路径后，使用以下代码创建路由实例：

```ts
const routes: RouteRecordRaw[] = [  
	// ...... 别的路由
	{  
        path: ':pathMatch(.*)*',  
        name: 'NotFound',  
        component: NotFoundView // 404界面 
    },  
]

const router = createRouter(  
    {  
        history: createWebHistory(),  
        routes,  
        scrollBehavior(to, from, savedPosition) {  
            if (savedPosition) {  
                return savedPosition;  
            }  
            return {top: 0};  
        },  
    }  
)
```

在这里，我们使用`createWebHistory`创建浏览器历史，它具有更好的SEO和更美观的URL。当URL没有匹配到任何界面时，Vue将显示您编写的404界面。

由于Github Pages是静态网页托管服务，它并不支持Vue Router使用的Web History模式，当您处于某个子路径下（比如`xxx.github.io/articles`）进行刷新时，Github Pages实际上会请求您仓库路径下的`/articles`的这个文件，但这个文件并不存在，网页将会显示404。

我们可以自己写一个`404.html`覆盖默认的404界面。由于Github Pages默认使用Jekyll构建静态网页，它会干扰我们的页面构建，所以请您在`public/`文件夹下创建一个名为`.nojekyll`的文件，内容不用编辑。它会指示Github Pages不使用Jekyll构建。

接着，在`index.html`中的`<head>`中插入如下的代码：

```html
<script>  
    (function () {  
        let redirect = sessionStorage.redirect;  
        delete sessionStorage.redirect;  
        if (redirect && redirect !== location.href) {  
            history.replaceState(null, null, redirect);  
        }  
    })();
</script>
```

在`public/`下创建一个`404.html`，在`<head>`中写上这段代码：

```html
<script>  
    sessionStorage.redirect = location.href;  
    location.replace('/');
</script>
```

访问不存在的路径时，Github Pages会返回`404.html`，其中的脚本会将用户当前的路径暂存到`sessionStorage`中，随后将用户重定向回`index.html`。`index.html`中的代码会恢复用户访问的地址，并加载Vue。Vue Router启动后将读取这个地址，将用户导航到正确的页面。

### 推送与部署

Github Pages需要将`index.html`直接放在仓库的根目录。我们将在构建目录（通常是`dist/`）进行Git仓库的初始化。它和您源代码仓库是分开的。

请在项目的源代码库中执行以下命令：

```bash
cd dist

git init
git add .

git commit -m "first commit"

# 设置远程仓库
git remote add origin https://github.com/yourname/yourname.github.io.git

# 推送
git push -u origin main
```

初始化操作后，我们设置远程仓库，并将构建产物目录中的内容整体推到远程仓库中。Github仓库默认分支名为`main`，如果您的分支名为`master`，请修改本地分支名称。

Github Pages在每次推送后将自动执行页面构建工作流。等待一至两分钟后，您将可以访问您的页面。

## 后续操作

### 更新文章

为了方便日常文章的更新与推送，您可以将构建与推送命令合并为一个npm脚本。在将您的新文章放入`public/articles/`文件夹后，执行此脚本，Vite将自动完成构建与部署。

在`package.json`中的`"scripts"`下添加以下内容：

```json
"deploy": "npm run build && cd dist && git status && git add -A && git commit -m 提交消息 && git push -f origin main",
```

这里使用了`-f`选项来强制推送分支，如果您不喜欢，可以去掉这个选项。

执行以下命令来进行更新与部署：

```bash
npm run deploy
```

---

感谢您看到这里！水平有限，如有错误，请不吝指出。