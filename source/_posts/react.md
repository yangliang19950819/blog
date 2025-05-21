---
title: react数据状态管理 和 reactSSR
cover: /img/life-z.webp
---

### useReducer 配合 createContext

```ts
    // Provider.ts 父组件提供 state 以及改变 state的方法dispatch 给到 createContext(xxx).Provider value={xxx}  
    import React,{ createContext , ReactNode, Dispatch ,useReducer} from "react";
    const reducer = (state:typeof initialState,action:Action) => {
        switch(action.type){
            case 'first':
                return {...state,first: action.payload.first};
            case 'last':
                return {...state,last : action.payload.last};
            default:    
                return state    
        }
    }
    const initialState = {
        first:'jack',
        last:'Herrington',
    }
    interface InitVal {
        first:string,
        last:string,
        dispatch:Dispatch<Action>
    }
    export const UserContext = createContext<InitVal>(initialState as InitVal);
    export const UseContextComponent:React.FC<Props>  = ({chilren}) =>{
        const [state,dispatch] = useReducer(reducer,initialState)
        const initVal1 = {
            ...state,
            dispatch
        }
        return (<UserContext.Provider value={initVal1}>
        {chilren}
        </UserContext.Provider>)
    }
    // Comsuer.ts 子组件通过useContext 拿到父组件提供的方法绑定点击函数dispatch 触发数据更新 导出父组件包裹的高阶组件给外部使用 
    import { UserContext ,  UseContextComponent } from "./Provider";
    import React,{ useContext }from "react";
    function ConsumerComponent(){
    const { first,last, dispatch } = useContext(UserContext);
        return (
            <div>
                First: {first}
                Last: {last}
                <br />
                <button onClick={()=>{
                    dispatch({type:'last',payload:{
                        first:'Jane',
                    last:'Smitch',}})
                }}>change</button>
            </div>
        )
    }
    export const ConsumerWrap = ()=><UseContextComponent chilren={<><ConsumerComponent /></>}  ></UseContextComponent>
```


### redux react-redux redux-thunk的联合使用

```ts
    // store/index.ts
    import { createStore , applyMiddleware}  from "redux";
    import thunk from "redux-thunk"; // thunk中间件 支持actions可以写入异步的脏操作等
    export const initVal = {
        count:0
    }
    interface ActionObj {
        type:"ADD" | "REDUCE",
        payload:typeof initVal
    }
    interface ActionFunObj {
        ADD:()=>typeof initVal,
        REDUCE:()=>typeof initVal,
    }
    function reducer (state:typeof initVal = initVal,action:ActionObj) {
        const actionFunObj:ActionFunObj = {
            ADD:function(){
                return { 
                    count:state.count + action.payload.count
                }
            },
            REDUCE:function(){
                return { 
                    count:state.count - action.payload.count
                }
            }
        }
        const type = action.type;
        return actionFunObj[type] ? actionFunObj[type]() : {count:0};
    }
    const store = createStore(reducer,applyMiddleware(thunk));

    export default store;

    // store/actions/counter.ts
    export const ADD = "ADD";
    export const REDUCE = "REDUCE";
    export const add = () => {return {type:ADD,payload:{count:3}}} 
    export const reduce = () => {return {type:REDUCE,payload:{count:3}}} 

    // store/actions/fetchData.ts
    import { ADD } from "./counter";
    import { Dispatch } from "redux";
    import { initVal } from "../index";
    export function fetchData (){
        return (dispatch:Dispatch,getState:()=> typeof initVal) => {
            fetch('https://www.fastmock.site/mock/ee2eb713c094e4a631c1eec3ee9b3386/jiong/api/ssr')
            .then(res => res.json())
            .then(res => dispatch({type:ADD,payload:{count:res.data.count}}))
        }
    }

    // App.tsx
    import React from "react";
    import "./App.css";
    import store from "./store/index";
    import { UseStateWrap } from "./components/useState";
    import { Provider } from "react-redux";

    function App() {
    return (
        <Provider store={store}>
        <div className="App">
            <header className="App-header" >
            <UseStateWrap />
            </header>
        </div>
        </Provider>
    );
    }
    export default App;

    // useState.tsx  useSelector useDispatch 以hooks的方式显示与改变状态数据
    import { connect, useSelector, useDispatch} from "react-redux"; 
    import { add,reduce , ADD ,REDUCE} from "../store/actions/counter";
    import { fetchData } from "../store/actions/fetachData";
    import { initVal } from "../store";
    function mapStateToProps(state:{count:number}){
        return {
            count:state.count
        }
    }
    const mapDispatchToProps = {
        add,
        reduce,
        fetchData
    }
    function UseState(props:{
        count:number,
        add:typeof add,
        reduce:typeof reduce,
        fetchData:() => void
    }){
        const dispatch = useDispatch();
        return <>
            <h3>redux</h3>
            count:{props.count}
            <br />
            useSelector:{useSelector((state)=>{
                return (state as typeof initVal).count
            })}
            <br />
            <button onClick={()=>{props.add()}}>加</button>
            <button onClick={()=>{props.reduce()}}>减</button>
            <button onClick={()=>{props.fetchData()}}>fetchData</button>
            <button onClick={()=>{dispatch({type:ADD,payload:{count:1}})}}>useDispatch加</button>
            <button onClick={()=>{dispatch({type:REDUCE,payload:{count:1}})}}>useDispatch减</button>
        </>
    }
    export const  UseStateWrap = connect(mapStateToProps,mapDispatchToProps)(UseState)
```

### recoil 原子级数据状态管理库

```ts
    // recoilState.tsx  selector相当于vue的computed是对atom的default的扩展
    import { atom,useRecoilState ,useRecoilValue ,selector} from "recoil";
    const counter = atom({
        key:'counter',
        default:''
    })
    const counterLength = selector({   
        key: 'counterLength', // unique ID (with respect to other atoms/selectors)
        get: ({get}) => {
        const count = get(counter);
        return count.length;
        },
    });
    export function RecoilState (){
        const [count,setCount] = useRecoilState(counter);
        const onChange = (e:React.ChangeEvent<HTMLInputElement>) => {
            setCount(e.target.value)
        }

        return <>
            <input type="text" value={count} onChange={onChange} />
            <br />
            RecoilState: {count}
            <br />
            useRecoilValue:{useRecoilValue(counterLength)}
        </>
    }
    // App.tsx
    import React from "react";
    import "./App.css";
    import { RecoilRoot } from 'recoil';
    import { RecoilState } from "./components/recoilState";

    function App() {
    return (
        <div className="App">
            <header className="App-header" >
            <RecoilRoot>
                <RecoilState />
            </RecoilRoot>
            </header>
        </div>
    );
    }
    export default App;
```

### reactSSR

#### webpack打包两份不同代码一份我们平常开发客户端代码一份不用处理css的供服务端执行的代码
- 注意点
  - 1.服务配置target:node library:{type:commonjs2} 打成可供node端使用的commomjs规范的包 
  - 2.避免状态store的状态单例子每次导出需要导出一个函数返回所需的对象。


``` js 
    // webpack配置 
    // webpack.base.js
    const { resolve } = require('path');
    const TerserPlugin = require('terser-webpack-plugin');
    module.exports = {
        output:{
            path:resolve(__dirname,"../../server/web")
        },
        cache:{
            type:'filesystem'
        },
        module:{
            rules:[
                {
                    test:/\.tsx?$/,
                    exclude:/node_modules/,
                    use:{
                        loader:'swc-loader',
                    }
                },
            ]
        },
        resolve:{
            extensions:['.js','.jsx','.ts','.tsx']
        },
        optimization:{
            minimize:true,
            minimizer:[new TerserPlugin()]
        }
    }

    // webpack.client.js
    const { resolve,join } = require("path");
    const { merge } = require('webpack-merge');
    const HtmlWebpackPlugin = require('html-webpack-plugin');
    const MiniCssExtractPlugin = require('mini-css-extract-plugin');
    const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
    // const { CleanWebpackPlugin } = require('clean-webpack-plugin'); 不开启我有用的文件都没了
    // const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin; 
    const baseConfig = require('./webpack.base');
    const clientConfig = {
        mode:'production',
        entry:resolve(__dirname,'../entry.client.tsx'),
        output:{
            filename:'js/[chunkhash].js',
            assetModuleFilename: 'images/[hash:5][ext]'
        },
        devServer:{
            contentBase:resolve(__dirname,"../../server/web"),
            port:8088,
            historyApiFallback:true,
            quiet:true
        },
        module:{
            rules:[
                {
                    test:/\.css$/,
                    use:[MiniCssExtractPlugin.loader,"css-loader"]
                },
                {
                    test:/\.(png|jpg|jpeg|gif|webp)$/i,
                    type:"asset",
                    // generator: {
                    //     filename: 'img/[chunkhash][ext]'
                    // }
                }
            ]
        },
        plugins:[
            new HtmlWebpackPlugin({
                template:resolve(__dirname,"../public/template.html"),
                favicon:resolve(__dirname,"../public/favicon.ico")
            }),
            new MiniCssExtractPlugin({
                filename:'css/[chunkhash].css'
            }),
            // new PrerenderSPAPlugin({
            //     staticDir: join(__dirname, "../../server/web"),
            //     routes: [ '/' ],
            //     renderer: new Renderer({
            //         headless: false,
            //         renderAfterDocumentEvent: 'render-event',
            //     })
            //   }),
            // new BundleAnalyzerPlugin(),
        ],
        // externals:{
        //     'react-syntax-highlighter':'SyntaxHighlighter',
        // },
        optimization:{
            minimizer:[
                new CssMinimizerPlugin()
            ],
            // runtimeChunk:{
            //     name:'runtime'
            // },
            splitChunks:{
                chunks:'all',
                maxInitialRequests:3,
                maxAsyncRequests:5,
                minChunks:3,
                minSize:{
                    javascript:30 * 1024,
                    style:30 * 1024
                },
                maxSize:{
                    javascript:110 * 1024,
                    style:110 * 1024
                },
                cacheGroups:{
                    common:{
                        chunks:'all',
                        name:'common',
                        priority:-20,
                        enforce:true,
                        reuseExistingChunk:true
                    }
                }
            }
        }
    }
    module.exports = merge(baseConfig,clientConfig)

    // webpack.server.js
    const { resolve } = require("path");
    const nodeExternals = require("webpack-node-externals");
    const { merge } = require("webpack-merge");
    const baseConfig = require("./webpack.base");
    const serverConfig = {
        mode:"production",
        entry:resolve(__dirname,"../entry.server.tsx"),
        output:{
            filename:"server.bundle.js",
            assetModuleFilename: 'images/[hash:5][ext]',
            library: {
                type: 'commonjs2',
            },
        },
        module:{
            rules:[{
                test:/\.css$/,
                use:'ignore-loader'
            },
            {
                test:/\.(png|jpg|jpeg|gif|webp)$/i,
                type:"asset",
            }
        ]
        },
        target:"node",
        externals:nodeExternals()
    }
    module.exports = merge(baseConfig,serverConfig)
```
- 客户端和服务端打包的入口文件
  - 可以看到服务端打包导出的是一个函数参数是koa框架提供的上下文ctx 在node端拿到真实的路由之后匹配前端的路由生成对应页面的模版替换 <div id="app"></div> 
  - 拿到routes里面的每个对象，如果有loadData方法的则执行它拿到结果，再把结果挂载到ctx对象上服务端再将数据挂载到window对象上，然后在初始化store时赋值上
  - 数据预获取是为了避免客户端拿到js再去请求服务端的数据，这样子链路过长，消耗时间。

```ts
    // entry-client.jsx
    import React from "react";
    import ReactDom from "react-dom";
    import App from "./pages/app";
    import { RecoilRoot } from "recoil";
    ReactDom.render(
        <RecoilRoot>
            <App />
        </RecoilRoot>,
        document.getElementById("app")
    )

    // entry-server.jsx 导出函数接受ctx 数据需要预先加载的函数执行
    import React from "react";
    import { Routes,routes } from "./router/index";
    import { StaticRouter } from "react-router-dom";
    import { RecoilRoot } from "recoil";
    import { IRouterContext } from "koa-router"
    export default (ctx:IRouterContext) => {
        return new Promise((resolve) => {
            const promises = [];
            routes.some((route) => {
                if(route.path === ctx.request.path && route.loadData){
                    promises.push(route.loadData())
                }
            })
            Promise.all(promises).then((res:any) => {
                (res[0]) && ((ctx as unknown as any).window = res[0].data.data);
                resolve(
                    <RecoilRoot>
                        <StaticRouter location={ctx.request.url}>{Routes()}</StaticRouter>
                    </RecoilRoot>
                )
            })
        })
    }
    // router/index.tsx 常规路由 如果需要预获取的数据函数可以写在这里
    import React from "react";
    import { Switch, Route } from "react-router-dom";
    import Blog from "../pages/Blog"; 
    import Tool from "../pages/Tool";
    import { getData } from "../pages/Tool";
    export const routes = [
        {
            path:'/',
            exact:true,
            component:Blog
        },
        {
            path:'/tool',
            exact:true,
            component:Tool,
            loadData:getData
        }
    ]
    export const Routes = () => (
        <Switch>
        {
            routes.map((r,index) => {
                const { path, component, exact } = r;
                return (
                    <Route path={path} exact={exact} component={component} key={index} />
                )
            })
        }
        </Switch>
    )

    // ssrService.ts  这里主要是做模板替换 数据注入
    import { provide } from "inversify-binding-decorators";
    import TAGS from "../constant/tag";
    import fs from "fs";
    import { resolve } from "path";
    import { renderToString } from "react-dom/server";
    import { IRouterContext } from "koa-router";
    import { SSR } from "../interface/ssr";
    const template = fs.readFileSync(resolve(__dirname,"../web/index.html"),'utf-8');
    const handleTemplate = (template:string) => {
        return (props:{html:string,store:string}) => (
            template.replace('<div id="app"></div>',`<div id="app">${props.html}</div>${props.store}`)
        )
    }
    const serverBundle = require('../web/server.bundle.js').default;
    @provide(TAGS.SsrService)
    export class SsrService implements SSR{
        async handleSsr(ctx:IRouterContext){
            const render = handleTemplate(template);
            const jsx = await serverBundle(ctx);
            const html = await renderToString(jsx);
            return render({
                html,
                store:`<script> window.REDUX_STATE = ${JSON.stringify((ctx as unknown as any).window)}</script>`
            })
        }
    }
```


### 开启bigpipe 让模板以流的形式输出以及开启gzip压缩减少白屏时间

```ts
    import { provideThrowable } from "../ioc";
    import { controller, httpGet, interfaces, TYPE } from "inversify-koa-utils";
    import { IRouterContext } from "koa-router";
    import { SSR } from "../interface/ssr";
    import TAGS from "../constant/tag";
    import { inject } from "inversify";
    import { Readable } from "stream";     
    import { createGzip } from "zlib";
    @provideThrowable(TYPE.Controller,"SsrController")
    @controller("/")
    class SsrController implements interfaces.Controller{
        private ssrService:SSR = null!
        constructor(@inject(TAGS.SsrService) ssrService:SSR){
            this.ssrService = ssrService
        } 
        @httpGet("/")
        private async index(ctx:IRouterContext,next:()=> Promise<unknown>):Promise<void>{
        const res =  await this.ssrService.handleSsr(ctx);
        function createSsrStreamPromise(){
                return new Promise((resolve,reject)=>{
                    const htmlStream = new Readable(); // 让字符串变为stream
                    ctx.status = 200;
                    ctx.res.setHeader('content-encoding','gzip');
                    const gz = createGzip();           // 开启gzip压缩 
                    htmlStream.push(res);
                    htmlStream.push(null);
                    htmlStream.on('error',(err)=>{
                        console.log(err)
                    })
                    .pipe(gz)
                    .pipe(ctx.res)
                })
        } 
        await createSsrStreamPromise()
        }
    }
    export default SsrController
```
