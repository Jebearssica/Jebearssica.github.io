---
layout: post
title: Error in jekyll chirpy deployment
date: 2022-03-30 01:47 +0800
tags: [config, bug fix]
categories: [Others]
author: Jebearssica
---

以前用gitbook的那个模板的, 感觉不好看换了一个[chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)的, 部署的过程中应该是 Github Action 的坑更多一点

## vscode git push GitHub/workflows

```log
refusing to allow an OAuth App to create or update workflow `.github/workflows/pages-deploy.yml`
```

隐隐约约记得一年前Github发了个邮件, 让你注意用token进行管理

[stackoverflow](https://stackoverflow.com/questions/64059610/how-to-resolve-refusing-to-allow-an-oauth-app-to-create-or-update-workflow-on)这里有进行讨论, 但人家只给出了 IntelliJ 的解决方案, vscode 里我都没搜到 token 的相关设置, 得下一个非官方的 GitHub 插件

<https://stackoverflow.com/questions/66231282/how-to-add-a-github-personal-access-token-to-visual-studio-code> 这里的解决方案是每一个 repo 都进行一次token设置

```sh
git remote set-url origin https://username:token@github.com/username/repository.git
```

有点过于离谱了, 考虑一下有没有折中的方案, 现在先放着, 毕竟除了这个 repo 需要上传pages-deploy.yml 其他 repo 还用不到这个功能

## 主页为空白页

- [x] 仍待解决

有可能因为 Github Action 被 ban 了, 正在给官方写 ticket 要求解封, 甚至 UJS112 组织的GitHub action 也被封了, 所以也没法测试

至少在organization中, 我用另一个未被ban的账号commit能够正常触发GitHub action从而解决该问题, 但是我并没有测试main分支是否有问题

原 repo 的 [issue](https://github.com/cotes2020/jekyll-theme-chirpy/issues/502) 中也有提到 workflow 中对main这个默认主分支和master默认主分支的问题, 目前暂定这两种情况

最终确定, master & main都行, 并不存在上述bug, 已经反馈给上述issue. 最终造成该原因就是GitHub action disable

## 新版本 htmlproofer 测试无法通过

### `'a' tag is missing a reference`

html5 允许该行为发生, 在 jekyll 渲染后生成的 html 文件可能存在这样的问题, 但这个并非人手动改的, 因此在 CI 中增加 `allow-missing-href=true` 从而允许这个行为

```yml
- name: Test site
run: |
    bundle exec htmlproofer _site \
    \-\-disable-external=true \
    \-\-ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/" \
    \-\-allow-missing-href=true \
```

> FYI, `\-` 的含义为 -, 而 `-` 的含义为 _. 此外需在命令后方输入 `\` 确保参数仍然指示同一个命令
{: .prompt-info }

### `xxx is not an HTTPS link`

不允许非 https 的链接, 我很赞成, 但 oiwiki 不更新证书我有啥办法, 增加 `enforce-https=false` 阻止这个检查

```yml
- name: Test site
run: |
    bundle exec htmlproofer _site \
    \-\-disable-external=true \
    \-\-ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/" \
    \-\-allow-missing-href=true \
    \-\-enforce-https=false \
```
