---
title: webcomponent 
cover: /img/web-c-z.webp
---

## Web Components的概念和使用

- Web Components 是一套不同的技术，允许您创建可重用的定制元素（它们的功能封装在您的代码之外）并且在您的web应用中使用它们。

- Web Components旨在解决这些问题 — 它由三项主要技术组成，它们可以一起使用来创建封装功能的定制元素，可以在你喜欢的任何地方重用，不必担心代码冲突。
 
   - Custom elements（自定义元素）：一组JavaScript API，允许您定义custom elements及其行为，然后可以在您的用户界面中按照需要使用它们。

   - Shadow DOM（影子DOM）：一组JavaScript API，用于将封装的“影子”DOM树附加到元素（与主文档DOM分开呈现）并控制其关联的功能。通过这种方式，您可以保持元素的功能私有，这样它们就可以被脚本化和样式化，而不用担心与文档的其他部分发生冲突。

   - HTML templates（HTML模板）： template 和 slot 元素使您可以编写不在呈现页面中显示的标记模板。然后它们可以作为自定义元素结构的基础被多次重用。

### 代码demo

1.customElements.define方法用来注册一个 custom element
   - 表示所创建的元素名称的符合 DOMString 标准的字符串。注意，custom element 的名称不能是单个单词，且其中必须要有短横线
   - 用于定义元素行为的 类
   - 可选参数，一个包含 extends 属性的配置对象，是可选参数。它指定了所创建的元素继承自哪个内置元素，可以继承任何内置元素
   - 例如：customElements.define('word-count', WordCount, { extends: 'p' })

2.attributeChangedCallback 会在 custom element增加、删除或者修改某个属性时被调用。

3.connectedCallback 会在 custom element首次被插入到文档DOM节点上时被调用。

4.this.shadowRoot.innerHTML  ShadowRoot 接口的 innerHTML 属性设置或返回 ShadowRoot内的DOM树。

5.observedAttributes 函数体内包含一个 return语句，返回一个数组，包含了需要监听的属性名称。


```js  webC.js在静态页面引入即可使用该组件 
//写入html中的组件
<my-counter count="10"></my-counter> 

class Counter extends HTMLElement {
    constructor(){
        super();
        this.attachShadow({mode:'open'})
    }

    static get observedAttributes(){
        return ['count']
    }   

    get count(){
        return this.getAttribute('count')
    }

    set count(val){
        this.setAttribute('count',val)
    }

    inc(){
        this.count++
    }

    btnEvent(){
        let btn = this.shadowRoot.querySelector('#btn');
        btn.addEventListener('click',this.inc.bind(this))
    }

    attributeChangedCallback(prop,oldVal,newVal){   
        if(prop === 'count'){
            this.render()
            this.btnEvent()
        } 
    }

    connectedCallback(){
        this.render();
        this.btnEvent()
    }

    render(){
        this.shadowRoot.innerHTML = `
            <h1>Counter</h1>
            ${this.count}
            <button id="btn">Increment</button>

            <style>
                h1{
                    color:rgba(0,0,0,.6)
                }
            </style>
        `
    }
}

customElements.define('my-counter',Counter)
```