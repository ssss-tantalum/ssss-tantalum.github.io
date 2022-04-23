---
title: "Hugo & GithubActions を使ってGitHubで完結するブログを作ってみた"
description: "このブログの技術とかそのへんについてまとめ"
tags: ["Tech", "GitHub Actions", "Hugo"]
date: 2022-04-23T14:00:00+09:00
draft: false
---

## はじめに

はじめまして。ヘボヘボWebエンジニアの ssss.tantalum (たんたる) と申します。  
以前から「ブログとかやってみたいなー」と思っていたのですが、ついに重い腰を上げました。

が、ブログサービスとかに登録して記事を書くのがそれはまぁ面倒くさい。  
その時ふと「全部 GitHub で済まされへんのか？」と思い調べてみたら意外とできたので、その方法を書きます。   
GitHub の草がいっぱい生えるとモチベーションあがるしね。

この記事の内容は色々適当なので、あまり当てにしないでください。  
致命的に間違っている所があって、どうしても指摘したい場合、プルリクを投げていただけると幸いです。

## 使用している技術

- Hugo
    - Golang 製の静的サイトジェネレーター
        - パッと調べたら楽ちんそうだったので採用
- GitHub
    - Issues
        - 記事の作成
    - Pull Request
        - 記事の最終確認
        - GitHub Pages にデプロイするときのフック
    - Actions
        - Issues → Pull Request の自動生成
        - Pull Request が main にマージされたら GitHub Pages にデプロイする

## Hugo プロジェクトを作成する

参考: https://gohugo.io/getting-started/quick-start/

```bash
hugo new site blog
```

なんか適当に記事を作成して、リポジトリに上げる。

```
hugo new posts/hello-world.md
```

テーマの設定とかもお好みでよしなにやる。

## Issue から Pull Request を自動生成する

GitHub Actions を利用して Issue から Pull Request を自動生成するように、リポジトリに `.github/workflows/publish.yaml` を作成する。

```yaml
name: Pushlish post from issue

on:
  issues:
    types: [labeled]

jobs:
  build:
    # issue のラベルが 'publish' だったら Actions を実行する
    if: github.event.label.name == 'publish'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Generate Post
        env:
          POST_TITLE: ${{ github.event.issue.title }}
          POST_BODY: ${{ github.event.issue.body }}
        # なんかうまいこと issue のタイトルでマークダウンファイルを作成し、issue の内容を突っ込む
        # Hugoのデフォルト設定で content/posts 配下にマークダウンファイルを設置すると記事として読み込んでくれる
        run: |
          cat > "content/posts/${POST_TITLE}.md" << EOF
          ${POST_BODY}
          EOF

      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v3
        with:
          delete-branch: true
          title: "publish: ${{ github.event.issue.title }}"
          # この辺は参考記事をほぼ丸パクリなので自分なりにアレンジしてみる
          body: |
            Automagically sprouted for publishing.
            Merging will publish to: https://ssss-tantalum.github.io/posts/${{ github.event.issue.title }}
            Closes #${{ github.event.issue.number }}
          reviewers: "${{ github.repository_owner }}"
          commit-message: "post: ${{ github.event.issue.title }}"
```

![スクリーンショット 2022-04-23 15 54 13](https://user-images.githubusercontent.com/103912715/164883774-eb7694c3-5be6-4393-938b-3b94b48c15a0.png)

こんな感じで作成してみる。

すると...設定した GitHub Actions が走り...

![スクリーンショット 2022-04-23 15 54 54](https://user-images.githubusercontent.com/103912715/164883838-ddb70f78-0081-443e-b353-1249ce6f6dc5.png)

Pull Request が作成されている。すごい。

![スクリーンショット 2022-04-23 15 55 36](https://user-images.githubusercontent.com/103912715/164883857-1ebeaaab-bfb5-46c5-8d26-504abab2950a.png)

## Pull Request が `main` にマージされたら GitHub Pages にデプロイする

`main` にマージされたことをフックに、Pull Request の内容を GitHub Pages にデプロイする。

例の如く `.github/workflows/gs-pages.yaml` を作成する。

```yaml
name: github pages

on:
  push:
    branches:
      - main  # Set a branch to deploy

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}
          external_repository: ssss-tantalum/ssss-tantalum.github.io
          publish_dir: ./public
```

Pull Request をマージしてみると...

![スクリーンショット 2022-04-23 16 01 36](https://user-images.githubusercontent.com/103912715/164884052-c1f2db35-3e27-48fd-884c-a40ac63379b3.png)

デプロイしてくれてるみたい。嬉しい。

ちゃんと GitHub Pages にも反映されていますね。よかった。

![スクリーンショット 2022-04-23 16 02 36](https://user-images.githubusercontent.com/103912715/164884099-4cca9be8-c941-43a1-8d88-0d3a8ad3ad32.png)

試しにやってみたけど、画像とかもちゃんと反映されてる。すごいやん。

## 終わりに

めちゃくちゃ適当にやってみましたが、意外となんとかなりました。  
全部 GitHub で完結するのはまじでありがたい。  
草もいっぱい生えて良き良き。  

なにかの参考・きっかけになれば幸いです。

## 参考文献

- https://gohugo.io/
- https://hacknote.jp/archives/58054/
- https://shazow.net/posts/github-issues-as-a-hugo-frontend/


