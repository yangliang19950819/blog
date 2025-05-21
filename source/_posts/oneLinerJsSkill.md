---
title: 20个一行js的小技巧
cover: /img/oneLiner.webp
---

### 获取浏览器cookie

```js
const cookie = name => `; ${document.cookie}`.split(`; ${name}=`).pop().split(';').shift();

cookie('_ga');
// Result: "GA1.2.1929736587.1601974046"
```


### 转换 RGB to Hex


```js
const rgbToHex = (r, g, b) =>
  "#" + ((1 << 24) + (r << 16) + (g << 8) + b).toString(16).slice(1);

rgbToHex(0, 51, 255); 
// Result: #0033ff
```


### 复制到剪切板

```js
const copyToClipboard = (text) => navigator.clipboard.writeText(text);

copyToClipboard("Hello World");
```

### 检查日期是否合法

```js
const isDateValid = (...val) => !Number.isNaN(new Date(...val).valueOf());

isDateValid("December 17, 1995 03:24:00");
// Result: true
```


### 查找一年中的某一天

```js
const dayOfYear = (date) =>
  Math.floor((date - new Date(date.getFullYear(), 0, 0)) / 1000 / 60 / 60 / 24);

dayOfYear(new Date());
// Result: 264
```


### 转换字符的首字母为大写

```js
const capitalize = str => str.charAt(0).toUpperCase() + str.slice(1)

capitalize("follow for more")
// Result: Follow for more
```


### 两个日期之间相隔多少天

```js
const dayDif = (date1, date2) => Math.ceil(Math.abs(date1.getTime() - date2.getTime()) / 86400000)

dayDif(new Date("2020-10-21"), new Date("2021-10-22"))
// Result: 366
```


### 清除所有的cookies

```js
const clearCookies = document.cookie.split(';').forEach(cookie => document.cookie = cookie.replace(/^ +/, '').replace(/=.*/, `=;expires=${new Date(0).toUTCString()};path=/`));
```


### 生成随机的hex

```js
const randomHex = () => `#${Math.floor(Math.random() * 0xffffff).toString(16).padEnd(6, "0")}`;

console.log(randomHex());
// Result: #92b008
```


### 数组去除重复

```js
const removeDuplicates = (arr) => [...new Set(arr)];

console.log(removeDuplicates([1, 2, 3, 3, 4, 4, 5, 5, 6]));
// Result: [ 1, 2, 3, 4, 5, 6 ]
```


### 获取url参数

```js
const getParameters = (URL) => {
  URL = JSON.parse('{"' + decodeURI(URL.split("?")[1]).replace(/"/g, '\\"').replace(/&/g, '","').replace(/=/g, '":"') +'"}');
  return JSON.stringify(URL);
};

getParameters(window.location)
// Result: { search : "easy", page : 3 }
```

### 从日期中取出时间

```js
const timeFromDate = date => date.toTimeString().slice(0, 8);

console.log(timeFromDate(new Date(2021, 0, 10, 17, 30, 0))); 
// Result: "17:30:00"
```

### 判断数字是奇数还是偶数

```js
const isEven = num => num % 2 === 0;

console.log(isEven(2)); 
// Result: True
```

### 在数字中找平均数

```js
const average = (...args) => args.reduce((a, b) => a + b) / args.length;

average(1, 2, 3, 4);
// Result: 2.5
```


### 滚动到浏览器顶部

```js
const goToTop = () => window.scrollTo(0, 0);

goToTop();
```


### 反转字符串

```js
const reverse = str => str.split('').reverse().join('');

reverse('hello world');     
// Result: 'dlrow olleh'
```

### 检查数组是否为空

```js
const isNotEmpty = arr => Array.isArray(arr) && arr.length > 0;

isNotEmpty([1, 2, 3]);
// Result: true
```

### 用户选择的文本范围或光标的当前位置

```js
const getSelectedText = () => window.getSelection().toString();

getSelectedText();
```


### 数组乱序

```js
const shuffleArray = (arr) => arr.sort(() => 0.5 - Math.random());

console.log(shuffleArray([1, 2, 3, 4]));
// Result: [ 1, 4, 3, 2 ]
```

### 检查是否是暗黑模式

```js
const isDarkMode = window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches

console.log(isDarkMode) // Result: True or False
```