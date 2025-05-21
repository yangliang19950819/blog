---
title: 基于yapi-to-typescript根据yapi接口文档自动.d.ts文件
cover: /img/yapi.png
---

#### 使用代码生成工具，根据 YApi 或 Swagger 的接口定义生成 TypeScript 或 JavaScript 的接口类型及其请求函数代码。

- 下载安装依赖

```ts
yarn add -D yapi-to-typescript
```

- 配置 ytt 文件

```ts
import { defineConfig } from "yapi-to-typescript";

export default defineConfig([
  {
    serverUrl: "http://183.146.28.170:3000",
    typesOnly: true,
    target: "typescript",
    reactHooks: {
      enabled: false,
    },
    prodEnvName: "test",
    outputFilePath: "./typings/api/yapi.d.ts",
    dataKey: "data",
    projects: [
      {
        token: "xxx", // yapi文档设置-token配置
        categories: [
          {
            id: 0,
            getRequestFunctionName(interfaceInfo, changeCase) {
              return changeCase.camelCase(interfaceInfo.path);
            },
          },
        ],
      },
    ],
  },
]);
```

- 配置 package.json 文件

```json
"ytt": "npx ytt"
```

- 执行 yarn ytt 生成.d.ts 文件

  ![avatar](/img/type.png)
