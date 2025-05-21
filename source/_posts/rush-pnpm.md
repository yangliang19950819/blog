---
title: rush+pnpm
cover: /img/rush.webp
---

### rush简介

- Rush makes life easier for JavaScript developers who build and publish many packages from a common Git repo. If you're looking to break up your giant application into smaller pieces, and you already realized why it doesn't work to put each package in a separate repo... then Rush is for you!

### rush特点

- 1.Ready for large repos
- 2.Designed for large teams
- 3.No phantom dependencies!
- 4.No NPM doppelgangers!
- 5.Easy to administer
- 6.Turnkey solution
- 7.Open model

### rush使用

- 1.在终端使用npm安装rush和pnpm
```shell
    npm i -g @microsoft/rush
    npm i -g pnpm
```

- 2.rush init创建项目
   - 此时会生成一个rush.json文件，在vscode打开可能会爆红，如果爆红需要点击右下脚的JSON把文件类型设置为JSON with Comments

![avatar](/img/rush1.webp)
![avatar](/img/rush2.webp)
![avatar](/img/rush3.webp)

- 3.在rush项目根目录创建apps文件夹,并使用create-react-app创建react项目,并在rush.json的projects中进行配置,配置完成后运行rush update下载更新依赖

```shell
npx create-react-app manage-package  --template  typescript 
```
```json
    {
      "packageName": "manage-package",
      "projectFolder": "apps/manage-package"
    },
```

- 4.cd 进入apps/manage-package rushx start运行react项目

- 5.cd .. 回到根目录新建libs文件夹,cd进入libs文件夹,使用tsdx创建单独的react组件项目,并且配置rush.json的projects,packageName要和package.json中的name要相同将组件的package.json的name也改为@shared/onecomponent,并修改下onecomponent/src/index.t组件代码,此时div里的内容为onecomponent


```ts
    import * as React from 'react';
    // Delete me
    export const Thing = () => {
    return <div>onecomponent</div>;
    };
``` 

```shell
    npx tsdx create onecomponent
```

```json
    {
      "packageName": "@shared/onecomponent",
      "projectFolder": "libs/onecomponent"
    }
```

  


- 6.创建完单个组件后cd 回到根目录执行 rush update --purge

- 7.再次进入之前的react项目在项目文件夹执行rush add,在rush add之后rush build 或者 rush rebuild。

```shell
    rush add --package @shared/onecomponent
```

- 8.最后在manage-package项目中的App.ts引入和使用@shared/onecomponent

```ts
    import React from 'react';
    import logo from './logo.svg';
    import './App.css';
    import { Thing } from '@shared/onecomponent';
    function App() {
    return (
        <div className="App">
        <header className="App-header">
            <img src={logo} className="App-logo" alt="logo" />
            <p>
            Edit <code>src/App.tsx</code> and save to reload.
            </p>
            <a
            className="App-link"
            href="https://reactjs.org"
            target="_blank"
            rel="noopener noreferrer"
            >
            Learn React
            </a>
            <Thing />
        </header>
        </div>
    );
    }
```
![avatar](/img/rush5.webp)
- 9.可以看到组件在前端项目中已经可以正常使用
