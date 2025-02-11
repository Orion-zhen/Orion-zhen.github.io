# 贡献指南

感谢你访问我的个人网站, 感谢你愿意贡献你的知识 ❤

## 文件结构

### 文章

如果你想要上传你的文章, 请将其在 `content/post` 文件夹中, 按照你的文章的 **类型(categories)** 来创建/放置对应的文件夹. 例如 教程类型 的文章就放在 `tutorial` 文件夹下. 你可以在类型文件夹中根据不同的主题新建细分文件夹

所有的文件夹名和文件名都应该使用小写英文字符, 连字符使用 `-`

一个正常的文章路径应该如下格式:

```shell
content/post/类型/细分主题(可选)/文件名.md
```

以下是一个示例:

```shell
content/post/test/contribute/test-article.md
```

### 图像

如果你想要上传图像, 请确保你的图像大小在 GitHub repo 允许范围内. 本仓库不允许 LFS, 所以请压缩或替换过大的图片文件

所有图片均应存放在 `static` 文件夹中, 并且遵循和引用该图片的文章一样的路径. 换言之, 将对应文章的路径中的 `content` 替换为 `static`, 即是这张图片的路径

对一篇文章中的单个图片, 文件名应该使用 `引用文章名-图片名` 的格式:

```shell
static/post/类型/细分主题(可选)/文章名-图片名
```

以下是一个示例:

```shell
static/post/test/contribute/test-article-test-image.png
```

如果一篇文章有多个图片, 文件名应该使用 `引用文章名/图片名` 的格式:

```shell
static/post/类型/细分主题(可选)/文章名/图片名
```

以下是一个示例:

```shell
static/post/test/contribute/test-article/test-image-1.png
```

要在文章中引用图像, 只需要将图像路径中的 `static` 删除即可: `/post/类型/细分主题(可选)/文件名-图片名`. 注意, 最开头的 `/` 是必要的

## 文章内容

在撰写文章时, 需要注意:

1. 所有的标点符号都应该是英文标点
2. 行内英文单词, 行内代码的两侧需要用空格包含, 但标点符号可以紧随. 示例:
   - 正确: 这是一段 example code, 用来演示格式要求
   - 正确: 这是一句带有括号(parentheses)的话
   - 错误: 这是一段`wrong`示例
3. 文章格式要求请遵守 `markdownlint` 的规则
4. 注意区分并正确填写文章的 **类型(categories)** 和 **标签(tag)**. 类型是根据文章的功能来决定的, 诸如教程, 笔记, 日志等; 标签则是根据文章的主题来确定的, 诸如大模型, 编程, 数学等

## 检查工作

当你完成了内容撰写后, 请将文章的 `draft` 状态设置为 `false`, 并且运行 `hugo server` 命令来验证你的改动是否合法. 当一切正常后, 你可以提交 PR
