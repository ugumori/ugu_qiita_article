<!--
title:   Nuxt.js + TypeScript で ESLintとprettierの設定をする
tags:    ESLint,Nuxt,TypeScript,prettier
id:      51be7592e39a6a0439b7
private: false
-->
2020年くらいに作ったNuxtプロジェクトでESLintとprettierを設定したが、時間が経ったので改めて環境を設定し直してみた。
ESLintは環境変化が激しく、まとまった情報があまりなかったので備忘として残しておく。

# 環境
`
# node --version
v16.16.0
# yarn --version
1.22.15
`

# 手順
ESLintとprettierを追加。
`
yarn add -D prettier
yarn add -D eslint
`
.eslintrcを作成。
対話で聞いてくれるので答えていく。
`
yarn create @eslint/config # eslintの設定ファイルを作成
 How would you like to use ESLint? …
  To check syntax only
❯ To check syntax and find problems
  To check syntax, find problems, and enforce code style
? What type of modules does your project use? …
❯ JavaScript modules (import/export)
  CommonJS (require/exports)
  None of these
? Which framework does your project use? …
  React
❯ Vue.js
  None of these
? Does your project use TypeScript? › No / Yes # Yesを選択
? Where does your code run? …  (Press <space> to select, <a> to toggle all, <i> to invert selection)
✔ Browser
✔ Node
? What format do you want your config file to be in? …
  JavaScript
  YAML
❯ JSON
eslint-plugin-vue@latest @typescript-eslint/eslint-plugin@latest @typescript-eslint/parser@latest
? Would you like to install them now? › No / Yes # Yesを選択
? Which package manager do you want to use? …
  npm
❯ yarn
  pnpm
`
プラグインの追加。
`
yarn add -D @nuxtjs/eslint-config-typescript  # vueファイルをリント対象にする
yarn add -D eslint-config-prettier # Prettierとの競合ルールをオフにする
yarn add -D eslint-plugin-nuxt     # Nuxt用のesLint設定を追加
`

出来上がったpackage.jsonのdependenciesがこちら。
(ESLintとprettier以外は省略)
`package.json
"devDependencies": {
    "@nuxtjs/eslint-config-typescript": "^10.0.0",
    "@typescript-eslint/eslint-plugin": "^5.33.1",
    "@typescript-eslint/parser": "^5.33.1",
    "eslint": "^8.22.0",
    "eslint-config-prettier": "^8.5.0",
    "eslint-plugin-nuxt": "^3.2.0",
    "eslint-plugin-vue": "^9.3.0",
    "prettier": "^2.7.1",
}
`

.eslintrcがこちら。
extendsの記述は後ろにあるものが優先、上書きされるので、
eslint一般設定 < vue < nuxt < prettier
となるように記述。
prettierは最後。
https://github.com/prettier/eslint-config-prettier
`.eslintrc.json
{
  "root": true,
  "env": {
    "browser": true,
    "node": true,
    "es2021": true
  },
  "extends": [
    "eslint:recommended",
    "plugin:vue/vue3-essential",
    "plugin:@typescript-eslint/recommended",
    "@nuxtjs/eslint-config-typescript",
    "plugin:nuxt/recommended",
    "prettier"
  ],
  "overrides": [],
  "parserOptions": {
    "parser": "@typescript-eslint/parser",
    "ecmaVersion": "latest",
    "sourceType": "module"
  },
  "plugins": ["vue", "@typescript-eslint"],
  "rules": {}
}
`

prettierの設定。
`.prettierrc.json
{
  "semi": false,
  "arrowParens": "always",
  "singleQuote": false,
  "trailingComma": "es5",
  "printWidth": 100,
  "tabWidth": 2
}
`

# 採用しなかったもの
## eslint-webpack-plugin(旧eslint-loader)
eslint-loaderは現在非推奨となっており、代わりにeslint-webpack-pluginを使う。
https://github.com/webpack-contrib/eslint-loader

eslint-loaderは以前採用していたが、Github ActionsのCI TestでPR作成時にeslintを走らせてるし、VSCode等のIDEでも常に警告を表示してくれるので、eslint-webpack-pluginも今回を機に採用をやめてみた。

## eslint-plugin-prettier
prettierをリンタールールのように実行できるプラグイン。
自分自身ESLintとprettierの境目をあまり分かっていなかったので、その辺分かりやすくするため導入見送り。

（公式でも現在非推奨）
https://prettier.io/docs/en/comparison.html
https://prettier.io/docs/en/integrating-with-linters.html#notes