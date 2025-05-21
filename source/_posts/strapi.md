---
title: 如何在gatsby项目中本地运行时不请求strapi来提示项目启动速度
cover: /img/strapi.png
---

- gatsby-config-ts 配置文件中注释 gatsby-source-strapi 插件配置

```ts
// {
//   resolve: `gatsby-source-strapi`,
//   options: strapiConfig,
// },
```

- 在使用 strapi 的文件中注释对应从 strapi 服务获取数据的代码

```ts
// const result = useStaticQuery<{
//   allStrapiAvicaBlogArt: AvicaBlogArt
// }>(graphql`
//   {
//     allStrapiAvicaBlogArt(
//       filter: { recommend: { eq: true } }
//       sort: { fields: publish_time, order: DESC }
//       limit: 9
//     ) {
//       nodes {
//         slug
//         title
//         recommend
//         categories {
//           seo {
//             title
//             description
//           }
//           arts {
//             brief
//             slug
//             title
//             reading_time
//           }
//           slug
//           name
//         }
//         cover {
//           ext
//           url
//           width
//           height
//         }
//         publish_time
//       }
//     }
//   }
// `)
// const blogs: Blog[] = result?.allStrapiAvicaBlogArt?.nodes || []
```

- m1 mac 请求 strapi 服务情况下项目启动时间大致在 4 分钟多

![avatar](/img/before.png)

- m1 mac 不请求 strapi 服务情况下项目启动时间大致在 20s 左右

![avatar](/img/after.png)
