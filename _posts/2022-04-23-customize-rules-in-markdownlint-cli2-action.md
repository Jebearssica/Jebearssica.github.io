---
layout: post
title: Customize Rules in Markdownlint-cli2-action
date: 2022-04-23 11:09 +0800
tags: [github action, config]
categories: [Others]
author: Jebearssica
---

vscode里面有个 markdown lint 的插件, 用来 format markdown 挺方便好看的, 还支持自定义规则. 那么就想一想能不能移植到 GitHub action 上自动化. 如果只是自己写的话, 直接在本地 vscode 里自定义规则就行了, 如下

```json
{
    "markdownlint.config": {
        "MD004": false,
        "MD026": false,
        "MD013": {
            "line_length": 999,
            "code_blocks": false,
            "tables": false
        }
    }
}
```

那不是考虑到还有一个实验室, 里面的小朋友们大多没有什么代码规范性的想法, 其实直接上传一个 vscode 配置文件也能解决(很多工作室都这样解决的, 方便快捷). 所以我干嘛要弄到 action 上, 此处引用生活大爆炸 S01E09 里的一段对白.

> Penny: Here is the question. Why?\
> Other: Because we can.

后续可能加一些类似于 OI Wiki 上的自动 format 的功能

- [ ]: markdown format in action

## 一些"坑"

总的来说配置其实没什么好讲的, 主要是有一些坑需要避过, 按照 [markdownlint-cli2-action](https://github.com/DavidAnson/markdownlint-cli2-action) 里面讲的, 写一个 yml 放到 `.github/workflows/`{: .filepath} 就行.

如果你需要定制化规则的话可以参照 [Example](https://github.com/chef/chef-web-docs/blob/main/.github/workflows/lint.yml) 的文件结构, 将你需要的配置文件**放入 repo 的根目录**

> 一定是根目录, 你放到另一个文件夹下定位不到, 至于为什么会这样没找到解释的地方, [download-file-action](https://github.com/carlosperate/download-file-action) 中对 location 参数的解释, 似乎意思是 location 代表你下载存放的地址
{: .prompt-warning }

## 一些可能有帮助的东西

* [markdownlint](https://github.com/DavidAnson/markdownlint)
* [markdownlint-cli2](https://github.com/DavidAnson/markdownlint-cli2)
* [markdownlint-cli2-action](https://github.com/DavidAnson/markdownlint-cli2-action)
* [Example](https://github.com/chef/chef-web-docs/blob/main/.github/workflows/lint.yml)
