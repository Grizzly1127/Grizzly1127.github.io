## Install

需要安装以下的环境：

* [Node.js](https://nodejs.org/en/download/)
* [hexo](https://hexo.io/docs/index.html)

安装hexo：

```bash
npm install -g hexo-cli
```

## Local run

```bash
hexo g & hexo s
```

## Deploy

**Linux:**

```bash
deploy.sh
```

**Windows:**

```bash
deploy.bat
```

## Method

### 创建新分类

打开终端，进入博客所在文件夹，执行命令：

```text
hexo new page <categories>
```

创建成功后，进入提示的路径：

```text
cd source/<categories>
```

目录下会生成`index.md`文件，添加`type: <categories>`到内容中

```text
---
title: <categories>
date: 2021-07-08 11:29:38
type: <categories>
---
```

### 创建新文章

打开终端，进入博客所在文件夹，执行命令：

```text
hexo new <title>
```


