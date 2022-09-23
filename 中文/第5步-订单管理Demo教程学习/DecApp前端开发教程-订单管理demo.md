# 说明

本教程基于 cyfs-dapp-cli 脚手架创建的模板项目进行讲解，使用模拟器环境进行开发。主要讲解模板项目的前端部分，对应工程结构中的 www 目录。
前端采用 React 框架进行开发。

## 准备条件

1. 请确保你已经通过 Cyber Chat 导出自己的身份到电脑上。

-   windows 系统，在 C:\Users\<your-conputer-name>\.cyfs_profile 目录下分别有 poeple.desc 和 people.sec 文件

-   mac 系统，在 ~/.cyfs_profile 目录下分别有 poeple.desc 和 people.sec 文件

2. 请确保已经在系统中安装好 nodejs 最新稳定版以及 yarn。
3. 全局安装 cyfs-sdk、cyfs-tool 和 cyfs-dapp-cli 脚手架。
   **注意** 如果使用的是 nightly 版本，请安装 cyfs-sdk-nightly、cyfs-tool-nightly
4. 在你觉得合适的文件夹下，使用脚手架创建 dec app 工程模板，命令如下：

```shell
npm i -g cyfs-sdk cyfs-tool cyfs-dapp-cli yarn
cyfs-dapp-cli create hello-demo -t order-demo
cd hello-demo
yarn
```

# 工程结构说明

```
-apis                       // 接口文件夹，存放请求相关的封装
-assets                     // 资源文件夹，存放静态资源文件，图片等
-components                 // 公共组件文件夹
-constants                  // 存放常量的文件夹
-containers                 // 容器组件文件夹
-hooks                      // 公用 hooks 文件夹
-i18n                       // 国际化语言包
-layout                     // 布局模板组件
-pages                      // 页面文件夹
-routers                    // 路由文件夹
-stores                     // 全局状态库文件夹
-styles                     // 全局或通用样式文件
-types                      // 全局通用类型文件
-utils                      // 存放封装的工具函数文件夹
initialize.ts               // 初始化 MetaClient 及 Stack 上线
webpack.config.js           // webpack 配置文件
```

## cyfs_helper

src/common/cyfs_helper 文件夹下的几个文件，分别对 MetaClient、Stack 以及各类常见的较复杂的操作进行了一些封装，方便开发者使用。

# CYFS 有关的初始化操作

在 src/www/initialize.ts 源码中，包含了前端所需的全部 CYFS 初始化操作。初始化操作有 3 个步骤：

1. 选择模拟器或者真实环境
2. MetaClient 上线
3. 等待 Stack 上线

代码如下：

```typescript
export async function init() {
    useSimulator(SimulatorZoneNo.FIRST, SimulatorDeviceNo.FIRST);
    MetaClient.init(MetaClient.EnvTarget.NIGHTLY);
    await waitStackRuntime(DEC_ID);
}
```

# 订单管理系统的前端接口开发(CRUD)

-   完整源码参见 src/www/apis/order.ts

完成前面的初始化准备工作之后，我们现在可以与服务端接口进行交互了。
这里的 CRUD 与《DApp 服务端开发教程》中的服务端接口保持一致。

## Create Order

首先，我们需要知道服务端的 Create Order 接口所需的请求参数以及会响应的参数对象，这样，前端才能有针对性的进行逻辑处理。
查看 src/common/routers.ts 文件，创建订单接口的请求参数与响应参数如下：

```typescript
export type CreateOrderRequestParam = Order;
export type CreateOrderResponseParam = ResponseObject;
```

不难看出，请求参数是 Order 对象，响应参数 ResponseObject 对象。
发起创建订单请求需要先准备创建好的 Order 对象，然后携带这个 Order 对象向服务端发起请求，服务端处理后返回 ResponseObjec 对象，前端接收到响应后，使用 ResponseObjectDecoder 解析出 ResponseObjec 对象。
示例代码如下：

```typescript
export async function createOrder(order: OrderObject) {
    const stackWraper = checkStack();
    // 创建Order对象
    const orderObj = Order.create(order);
    // 发起请求
    const ret = await stackWraper.postObject(orderObj, ResponseObjectDecoder, {
        reqPath: ROUTER_PATHS.CREATE_ORDER,
        decId: stackWraper.decId
    });
    if (ret.err) {
        console.error(`reponse err, ${ret}`);
        return null;
    }
    // 解析出 ResponseObjec 对象
    const r = ret.unwrap();
    if (r) {
        const retObj = {
            err: r.err,
            msg: r.msg
        };
        console.log(`reponse, ${retObj}`);
        return JSON.stringify(retObj);
    }
    return null;
}
```

## Retrieve Order

查看 src/common/routers.ts 文件，创建订单接口的请求参数与响应参数如下：

```typescript
export type RetrieveOrderRequestParam = Order;
export type RetrieveOrderResponseParam = Order;
```

这里的请求参数与响应参数都是 Order 对象。
发起查询订单请求需要先准备创建好的 Order 对象，这里的 Order 对象只包含 key 值，然后携带这个 Order 对象向服务端发起请求，服务端处理后返回查询到的 Order 对象，前端接收到响应后，使用 OrderDecoder 解析出 Order 对象。
示例代码如下：

```typescript
export async function retrieveOrder(key: string) {
    const stackWraper = checkStack();
    // 创建Order对象，只包含 key 值
    const obj = Order.create({ key, decId: stackWraper.decId!, owner: stackWraper.checkOwner() });
    // 发起请求
    const ret = await stackWraper.postObject(obj, OrderDecoder, {
        reqPath: ROUTER_PATHS.RETRIEVE_ORDER,
        decId: stackWraper.decId
    });
    if (ret.err) {
        console.error(`reponse error, ${ret}`);
        return null;
    }
    // 解析 Order 对象
    const r = ret.unwrap();
    if (r) {
        const orderObj = {
            key: r.key,
            timestamp: r.timestamp,
            price: r.price,
            buyer: r.buyer,
            status: r.status
        };
        console.log(`reponse, ${JSON.stringify(orderObj)}`);
        return orderObj;
    }
    return null;
}
```

## Update Order

查看 src/common/routers.ts 文件，创建订单接口的请求参数与响应参数如下：

```typescript
export type UpdateOrderRequestParam = Order;
export type UpdateOrderResponseParam = ResponseObject;
```

这里的请求参数是 Order 对象，响应参数是 ResponseObject 对象。
发起更新订单请求需要先准备创建好的 Order 对象，然后携带这个 Order 对象向服务端发起请求，服务端处理后返回 ResponseObject 对象，前端接收到响应后，使用 ResponseObjectDecoder 解析出 ResponseObject 对象。
示例代码如下：

```typescript
export async function updateOrder(order: OrderObject) {
    const stack = checkStack();
    // 创建 Order 对象
    const orderObj = Order.create(order);
    // 发起请求
    const ret = await stack.postObject(orderObj, ResponseObjectDecoder, {
        reqPath: ROUTER_PATHS.UPDATE_ORDER,
        decId: stack.decId
    });
    if (ret.err) {
        console.error(`reponse err, ${ret}`);
        return null;
    }
    // 解析出 ResponseObject 对象
    const r = ret.unwrap();
    if (r) {
        const retObj = {
            err: r.err,
            msg: r.msg
        };
        console.log(`reponse, ${retObj}`);
        return JSON.stringify(retObj);
    }

    return null;
}
```

## Delete Order

查看 src/common/routers.ts 文件，创建订单接口的请求参数与响应参数如下：

```typescript
export type DeleteOrderRequestParam = Order;
export type DeleteOrderResponseParam = ResponseObject;
```

这里的请求参数是 Order 对象，响应参数是 ResponseObject 对象。
发起删除订单请求需要先准备创建好的 Order 对象，这里的 Order 对象只包含 key 值，然后携带这个 Order 对象向服务端发起请求，服务端处理后返回 ResponseObject 对象，前端接收到响应后，使用 ResponseObjectDecoder 解析出 ResponseObject 对象。
示例代码如下：

```typescript
export async function deleteOrder(key: string) {
    const stackWraper = checkStack();
    // 创建Order对象，只包含 key 值
    const obj = Order.create({ key, decId: stackWraper.decId!, owner: stackWraper.checkOwner() });
    // 发起请求
    const ret = await stackWraper.postObject(obj, ResponseObjectDecoder, {
        reqPath: ROUTER_PATHS.DELETE_ORDER,
        decId: stackWraper.decId
    });
    if (ret.err) {
        console.error(`reponse err, ${ret}`);
        return null;
    }
    // 解析出 ResponseObject 对象
    const r = ret.unwrap();
    if (r) {
        const retObj = {
            err: r.err,
            msg: r.msg
        };
        console.log(`reponse, ${retObj}`);
        return JSON.stringify(retObj);
    }
    return null;
}
```

# 彩蛋

恭喜你，到这一步，我们的订单管理系统 Demo 的前端部分代码基本完成了。
但是，细心的小伙伴可能会发现，在 src/www/apis/order.ts 源码中，好像漏了一个叫 listOrdersByPage 的函数还没介绍。
是的，这个函数作为文末的彩蛋给大家单独介绍。

listOrdersByPage 能够分页式的从 /orders 路径下读取一定数量的 Order 对象回来。
先看代码：

```typescript
export async function listOrdersByPage(pageIndex: number) {
    const stack = checkStack();
    // 获取自己的OwnerId
    const selfObjectId = stack.checkOwner();
    // 获取到 cyfs.GlobalStateAccessStub 实例
    const access = stack.check().root_state_access_stub(selfObjectId);
    // 使用list方法，列出 /orders 下的全部对象
    const lr = await access.list('/orders', pageIndex, 10);

    if (lr.err) {
        if (lr.val.code !== cyfs.BuckyErrorCode.NotFound) {
            console.error(`list-subdirs in(/orders) io failed, ${lr}`);
        } else {
            console.warn(`list-subdirs in(/orders) not found, ${lr}`);
        }
        return [];
    }

    const list = lr.unwrap();
    const keyList = list.map((item) => item.map!.key);
    return keyList;
}
```

跟前面的 CRUD 请求代码对比一下，看看有什么不同？
嗯，就像你看到的那样，大部分都不同。
这里没有使用 postObject 跟服务端交互，而是使用 Stack 上的 root_state_access_stub 方法获得 cyfs.GlobalStateAccessStub 实例，再使用 cyfs.GlobalStateAccessStub 实例的 list 方法直接获取 RootState 上的数据记录。
也就是说，你可以在不增加服务端接口的情况下，直接查询到某个路径下的全部对象列表，是不是比传统的 Web2 开发更方便？

# 运行前端项目

在项目根目录输入以下指令：

```shell
yarn dev
```

在 CYFS 浏览器中访问 http://localhost:8088，即可看到前端界面。
