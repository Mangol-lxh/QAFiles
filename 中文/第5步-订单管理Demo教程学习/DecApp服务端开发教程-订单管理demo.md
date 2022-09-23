# 说明

本教程基于 cyfs-dapp-cli 脚手架创建的模板项目进行讲解，使用模拟器环境进行开发。
在模板项目中，包含了一个订单管理系统 Demo，这个 Demo 中实现了对订单对象的 CRUD 操作。
通过这个 Demo 系统的实现，帮助开发者快速熟悉并掌握基本的 Service 开发技能。

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

## 工程结构

在 hello-demo/src 目录下，有三个子文件夹：

1. common: 前端和 Service 端可以共同复用的代码
2. service: Service 工程代码
3. www: 前端工程代码

### 代码修改

1. 将 src/common/constant.ts 源码中的 DEC_ID_BASE58 常量值修改为自己的 dec app id，在 cyfs.config.json -> app_id
2. 将 src/common/constant.ts 源码中的 APP_NAME 常量值修改为自己的 dec app 名字，在 cyfs.config.json -> app_name

## cyfs_helper

src/common/cyfs_helper 文件夹下的几个文件，分别对 MetaClient、Stack 以及各类常见的较复杂的操作进行了一些封装，方便开发者使用。

## 两个环境

服务端代码可以在模拟器和 OOD 两个环境运行。在开发测试阶段，为便于调试，建议使用模拟器环境。发布上线阶段，再切换到 OOD 环境。

### 模拟器环境

在本机运行 zone-simulator 模拟器工具进行开发测试的环境，模拟器工具在根目录下的 tools/zone-simulator 文件夹下。

#### 模拟器可选端口

模拟器可以最多模拟两个 zone，每个 zone 里面包含一个 ood + 两个 device，所以一共存在六个协议栈，为了方便外部 sdk 调用，所以使用了固定端口，分配如下：

|     设备      | bdt-port | http-port | ws-port |
| :-----------: | :------: | :-------: | :-----: |
|  zone1-ood1   |  20001   |   21000   |  21001  |
| zone1-device1 |  20002   |   21002   |  21003  |
| zone1-device2 |  20003   |   21004   |  21005  |
|  zone2-ood1   |  20010   |   21010   |  21011  |
| zone2-device1 |  20011   |   21012   |  21013  |
| zone2-device2 |  20012   |   21014   |  21015  |

#### 打开并等待 Stack 上线

如果想要使用 zone1-ood1 的协议栈，可以使用下述代码:

```typescript
import * as cyfs from 'cyfs-sdk';

async function openAndWaitStackSimulatorOOD(
    decId: cyfs.ObjectId
): Promise<cyfs.SharedCyfsStack | null> {
    const param = cyfs.SharedCyfsStackParam.new_with_ws_event_ports(21000, 21001, decId);
    if (param.err) {
        console.error(`init SharedCyfsStackParam failed, ${param}`);
        return null;
    }
    const stack = cyfs.SharedCyfsStack.open(param.unwrap());
    const ret = await stack.wait_online(cyfs.Some(cyfs.JSBI.BigInt(1000000)));
    if (ret.err) {
        console.error(`wait stack online failed, ${ret}`);
        return null;
    }
    return stack;
}
```

### OOD 环境

发布上线的环境，如各类主机。

#### 打开并等待 Stack 上线

```typescript
import * as cyfs from 'cyfs-sdk';

async function openAndWaitStackOOD(decId: cyfs.ObjectId): Promise<cyfs.SharedCyfsStack | null> {
    const stack = cyfs.SharedCyfsStack.open_default(decId);
    const ret = await stack.wait_online(cyfs.Some(cyfs.JSBI.BigInt(1000000)));
    if (ret.err) {
        console.error(`wait stack online failed, ${ret}`);
        return null;
    }
    return stack;
}
```

# Service 启动程序

Service 启动程序完整代码位于 src/service/entry/app_startup.ts

Service 启动程序主要有 3 步：

1. 开启 Service 日志。
2. 打开并等待 Stack 上线。
3. 在 Stack 上注册路由。

## 开启 Service 日志

日志可以让我们快速的发现并定位问题，对解决线上问题十分有帮助。不同的操作系统，应用日志的存储路径略有不同；

-   mac: ~/Library/cyfs/log/app/<app_name>
-   windows: C:\cyfs\log\app\<app_name>
-   linux: /cyfs/log/app/<app_name>

基于 CYFS SDK 开启 Service 日志非常简单，代码如下：

```typescript
import * as cyfs from 'cyfs-sdk';

cyfs.clog.enable_file_log({
    name: APP_NAME,
    dir: cyfs.get_app_log_dir(APP_NAME)
});
```

## 打开并等待 Stack 上线

通过引入 cyfs_helper 中的 waitStackOOD 方法，我们可以很方便实现打开并等待 Stack 上线，代码如下：

```typescript
import { waitStackOOD } from 'src/common/cyfs_helper/stack_wraper';

const waitR = await waitStackOOD(DEC_ID);
if (waitR.err) {
    console.error(`service start failed when wait stack online, err: ${waitR}.`);
    return;
}
```

细心的小伙伴可能会发现，这里没有像前端初始化那样，显式的指定 Stack 连接的是模拟器还是真实 OOD，其实是因为在 waitStackOOD 方法中运行了 src/common/cyfs_helper/stack_wraper.ts 中的 checkSimulator 方法，这个方法会帮助我们通过解析命令行参数来指定模拟器，默认是选择真实 OOD 。

因此，当你去查看 package.json 文件中的 scripts 内容时，会看到运行模拟器 1 的指令为：

```
"sim1": "npx tsc & node move_deploy.js & node deploy/src/service/entry/app_startup.js --start --simulator 1"
```

这里，我们通过命令行的形式指定使用模拟器 1

## 在 Stack 上注册路由

在 Stack 上注册路由与传统的 Web2 服务端开发路由模块开发很相似，都离不开路由路径、请求参数、响应参数和路由模块。
在设计路由模块时，应遵循高内聚、低耦合以及单一职责等设计原则，一个路由模块只干好一件事情。

在我们的示例中，使用 addRouters 来批量注册路由，提高开发效率，addRouters 函数的代码如下：

```typescript
import * as cyfs from 'cyfs-sdk';

export type RouterArray = Array<{
    reqPath: string;
    router: postRouterHandle;
}>;

async function addRouters(stack: cyfs.SharedCyfsStack, routers: RouterArray): Promise<void> {
    for (const routerObj of routers) {
        const handleId = `post-${routerObj.reqPath}`;
        const r = await stack.router_handlers().add_post_object_handler(
            cyfs.RouterHandlerChain.Handler,
            handleId,
            1,
            `dec_id==${stack.dec_id!}`, // filter config
            cyfs.RouterHandlerAction.Pass,
            cyfs.Some(new PostRouterReqPathRouterHandler(routerObj))
        );

        if (r.err) {
            console.error(`add post handler (${handleId}) failed, err: ${r}`);
        } else {
            console.info(`add post handler (${handleId}) success.`);
        }
    }
    console.log(`added ${Object.entries(routers).length} routers success.`);
}
```

学习过《HelloWorld 教程》的小伙伴应该不会陌生，这里的主要区别是每一个路由模块不再是直接使用函数，而是组装成了一个包含 reqPath 和 router 的对象，reqPath 代表的是请求路径，router 代表路由函数名。
我们需要重点关注的是继承接口类 cyfs.RouterHandlerPostObjectRoutine 后的 call 方法实现，代码如下：

```typescript
class PostRouterReqPathRouterHandler implements cyfs.RouterHandlerPostObjectRoutine {
    ...
    public call(
        param: cyfs.RouterHandlerPostObjectRequest
    ): Promise<cyfs.BuckyResult<cyfs.RouterHandlerPostObjectResult>> {
        const reqPath = param.request.common.req_path;
        if (reqPath === this.m_routerObj.reqPath) {
            return this.m_routerObj.router(param);
        } else {
            return Promise.resolve(
                cyfs.Ok({
                    action: cyfs.RouterHandlerAction.Pass
                })
            );
        }
    }
}
```

而在《HelloWorld 教程》中，call 方法实现如下：

```typescript
class PostRouterReqPathRouterHandler implements cyfs.RouterHandlerPostObjectRoutine {
    ...
    public call(
        param: cyfs.RouterHandlerPostObjectRequest
    ): Promise<cyfs.BuckyResult<cyfs.RouterHandlerPostObjectResult>> {
        return this.m_router(param);
    }
}
```

在《HelloWorld 教程》中，Service 只注册了一个路由，所以不管你的请求路径是什么，只要是该应用发起的请求，都会被默认路由模块执行。
而在我们这里，由于需要注册很多个功能不同的路由模块，所以要先判断请求路径与实际路由路径是否相同，只有相同才能执行对应的路由模块。

# 对象建模

在 CYFS 中，一切数据都是以对象的形式存在，这里说的对象其实就是 NamedObject，对象存储在 RootState(类似文件系统)上。
网络中的许多操作都需要 NamedObject 才能进行，如发起请求时，请求参数必须是一个 NamedObject 对象，返回响应时，返回参数也必须是一个 NamedObject 对象。
在 RootState 上，每个 Dec App 都有一个专属的路径，在此路径下，我们可以存储 Dec App 所需要的一切对象数据。
因此，我们必须学会如何为对象建模。

## 设计 Order 对象数据的存储结构

参见 doc/对象模型.md

Order 对象数据的存储结构如下：

```
|--orders
|   |--order1.key => Order对象
|   |--order2.key => Order对象
```

我们的 Order 对象存放在应用根目录下的 orders 路径下，这个路径下的所有 Order 对象数据都储存在一个 Map 对象中，key 是 Order 对象中的字符串 key，value 是 Order 对象。

## Order 对象建模

-   参见 src/common/objs/obj_proto.proto

Order 对象属于命名对象(NamedObject)，在 CYFS 中，我们使用 proto3 的来支持对各种命名对象(NamedObject)的编解码。

我们这里设计的 Order 对象包含 5 个属性：

1. key: 订单标识
2. timestamp: 创建订单的时间戳
3. price: 订单金额
4. buyer: 订单的购买者
5. status: 订单状态，打开或者关闭

在 proto 文件中的定义如下：

```proto
syntax = "proto3";

enum OrderStatus {
	CLOSED = 0;
	OPEN = 1;

}

message Order {
	string key = 1;
    optional uint64 timestamp = 2;
	optional uint64 price = 3;
	optional  string buyer = 4;
    optional OrderStatus status = 5;
}
```

说明：

1. enum 用来定义枚举，message 用来定义对象， optional 表示属性是可选的，其值可能为空。
2. timestamp 等 4 个属性设置可选，是为了方便查询或者删除订单时复用 Order 对象来发起请求，Order 对象只携带 key 值。

## 编译 proto 文件为 Typescript 和 Javascript 的对象定义

在利用 proto3 生成 Typescript 和 Javascript 的对象定义 时，不同的系统在操作上有所差异。

### mac

在项目根目录下，执行指令如下：

```shell
yarn proto-mac-pre
yarn proto-mac
```

**注意** 由于是直接执行的 protoc 执行程序，可能会弹窗提示 _无法打开“protoc”，因为无法验证开发者_，需要开发者按照以下路径去设置：
_系统偏好设置_ -> _安全性与隐私_ -> _允许从以下位置下载的 App_ -> 选择 _App Store 和认可的开发者_ -> 点击 _仍然允许_
按照这个路径设置好，重新执行指令即可。
运行完毕，在 src/common/objs 文件夹下，生成了 obj_proto_pb.d.ts 和 obj_proto_pb.js 这两个文件。在 obj_proto_pb.d.ts 声明文件中，我们看到了 Order 对象的类型定义。

### windows

```shell
yarn proto-windows
```

运行完毕，在 src/common/objs 文件夹下，生成了 obj_proto_pb.d.ts 和 obj_proto_pb.js 这两个文件。在 obj_proto_pb.d.ts 声明文件中，我们看到了 Order 对象的类型定义。

## 根据 Typescript 的对象定义生成 Order 对象文件

-   完整代码见 src/common/objs/order.ts

我们需要重点关注的是其中的 class Order 和 class OrderDecoder；

1. 创建 Order 对象，使用 class Order 的 create 方法
2. 使用 class OrderDecoder 解码出 Order 对象

## 创建 ResponseObject 对象文件

-   proto 定义见 src/common/objs/obj_proto.proto

-   ResponseObject 对象文件在 src/common/objs/response_object.ts

在我们的示例中，并不是全部的接口都返回 Order 对象，个别接口需要返回其他信息，我们非 Order 对象的响应对象定义为 ResponseObject 对象。
由于方法与 Order 对象类似，在此就不赘述。

# 实现对订单对象的 CRUD 操作

传统的 Web2 服务端开发中，基础功能通常是基于数据库的 CRUD：

1. Create new records
2. Retrieve records
3. Update records
4. Delete records

在我们这里，CRUD 操作的不是 records，而是对象。
除此之外，在请求方法、请求路径、请求参数、响应参数等路由模块的功能开发方面，我们与传统的 Web2 服务端开发是相似的。

## CRUD 请求路径及参数

-   完整代码见 src/common/routers.ts

为方便前端和 Service 端复用请求路径代码，我们把请求路径及参数都定义在了 src/common/routers.ts 文件中。
使用枚举来定义全部的请求路径，便于维护和修改，代码如下：

```typescript
export const enum ROUTER_PATHS {
    TEST_HELLO = '/test/hello',
    CREATE_ORDER = '/order/create',
    RETRIEVE_ORDER = '/order/retrieve',
    UPDATE_ORDER = '/order/update',
    DELETE_ORDER = '/order/delete',
    CREATE_ORDER_REQ = '/order/create_req'
}
```

## 创建订单路由

-   完整代码见 src/service/routers/create_order.ts

前端发起请求，请求参数为 Order 对象，接口接收后进行处理，响应 ResponseObject 对象。我们将对完整代码逐块的进行讲解。

1. 解析出前端发来的请求对象，判断请求对象是否是 Order 对象。如果不是 Order 对象，接口直接返回无效参数的错误响应。

```typescript
const { object, object_raw } = req.request.object;
if (!object || object.obj_type() !== AppObjectType.ORDER) {
    const msg = 'obj_type err.';
    console.error(msg);
    return Promise.resolve(makeBuckyErr(cyfs.BuckyErrorCode.InvalidParam, msg));
}
```

2. 使用 OrderDecoder 对请求发来的 Order 对象(Uint8Array)解码，如果解码出错，则直接返回错误响应，反之，我们将得到 Order 对象。

```typescript
const decoder = new OrderDecoder();
const dr = decoder.from_raw(object_raw);
if (dr.err) {
    const msg = `decode failed, ${dr}.`;
    console.error(msg);
    return dr;
}
const orderObject = dr.unwrap();
```

3. 创建 pathOpEnv，用于后续对 RootState 上的对象进行事务操作，如果创建 pathOpEnv 失败，则直接返回错误响应，反之，我们将得到创建好的 pathOpEnv

```typescript
let pathOpEnv: cyfs.PathOpEnvStub;
const stack = checkStack().check();
const r = await stack.root_state_stub().create_path_op_env();
if (r.err) {
    const msg = `create_path_op_env failed, ${r}.`;
    console.error(msg);
    return r;
}
pathOpEnv = r.unwrap();
```

4. 确定新 Order 对象将要存储的路径并对该路径上锁，如果上锁失败，直接返回错误信息。

```typescript
const path = `/orders/${orderObject.key}`;
const paths = [path];
console.log(`will lock paths ${JSON.stringify(paths)}`);
const lockR = await pathOpEnv.lock(paths, cyfs.JSBI.BigInt(30000));
if (lockR.err) {
    const errMsg = `lock failed, ${lockR}`;
    console.error(errMsg);
    pathOpEnv.abort();
    return Promise.resolve(makeBuckyErr(cyfs.BuckyErrorCode.Failed, errMsg));
}
console.log(`lock ${JSON.stringify(paths)} success.`);
```

5. 利用 Order 对象信息创建对应的 NONObjectInfo 对象，通过 put_object 操作，把 NONObjectInfo 对象新增到 RootState 上。如果 put_object 失败，则直接返回失败响应。

```typescript
const decId = stack.dec_id!;
const nonObj = new cyfs.NONObjectInfo(
    orderObject.desc().object_id(),
    orderObject.encode_to_buf().unwrap()
);
const putR = await stack.non_service().put_object({
    common: {
        dec_id: decId,
        level: cyfs.NONAPILevel.NOC, // 仅限本地操作，不会发起网络操作
        flags: 0
    },
    object: nonObj
});
if (putR.err) {
    pathOpEnv.abort();
    const errMsg = `commit put-object failed, ${putR}.`;
    console.error(errMsg);
    return Promise.resolve(makeBuckyErr(cyfs.BuckyErrorCode.Failed, errMsg));
}
```

6. 使用 NONObjectInfo 的 order_id 进行创建新 Order 对象的事务操作，如果该操作失败，直接返回错误响应。

```typescript
const objectId = nonObj.object_id;
const rp = await pathOpEnv.insert_with_path(path, objectId);
if (rp.err) {
    pathOpEnv.abort();
    const errMsg = `commit insert_with_path(${path}, ${objectId}), ${rp}.`;
    console.error(errMsg);
    return Promise.resolve(makeBuckyErr(cyfs.BuckyErrorCode.Failed, errMsg));
}
```

7. 事务提交，如果事务失败，则直接响应失败信息。

```typescript
const ret = await pathOpEnv.commit();
if (ret.err) {
    const errMsg = `commit failed, ${ret}.`;
    console.error(errMsg);
    return Promise.resolve(makeBuckyErr(cyfs.BuckyErrorCode.Failed, errMsg));
}
```

8. 创建 ResponseObject 对象作为响应参数并将结果发给前端.

```typescript
const respObj: CreateOrderResponseParam = ResponseObject.create({
    err: 0,
    msg: 'ok',
    decId: stack.dec_id!,
    owner: checkStack().checkOwner()
});

return Promise.resolve(
    cyfs.Ok({
        action: cyfs.RouterHandlerAction.Response,
        response: cyfs.Ok({
            object: toNONObjectInfo(respObj)
        })
    })
);
```

## 查询订单路由

-   完整代码见 src/service/routers/retrieve_order.ts

前端发起请求，请求参数为 Order 对象，接口接收后进行处理，响应 Order 对象，我们将不对完整代码逐块的进行讲解。
查询 Order 对象的逻辑在很多地方与创建 Order 对象是一样的，因此，我们只讲解差异部分的代码块。

1. 使用 pathOpEnv 的 get_by_path 方法 从 Order 对象的存储路径中获取 Order 对象的 object_id，如果 get_by_path 返回的结果有错误，则直接响应错误信息，如果 get_by_path 返回的结果为空，也直接响应错误信息。

```typescript
const idR = await pathOpEnv.get_by_path(queryOrderPath);
if (idR.err) {
    const errMsg = `get_by_path (${queryOrderPath}) failed, ${idR}`;
    pathOpEnv.abort();
    return Promise.resolve(makeBuckyErr(cyfs.BuckyErrorCode.Failed, errMsg));
}
const id = idR.unwrap();
if (!id) {
    const errMsg = `objectId not found after get_by_path (${queryOrderPath}), ${idR}`;
    pathOpEnv.abort();
    return Promise.resolve(makeBuckyErr(cyfs.BuckyErrorCode.Failed, errMsg));
}
```

2. 使用 get_object 方法，以 Order 对象的 object_id 为参数从 RootState 上获取到 Order 对象对应的 cyfs.NONGetObjectOutputResponse 对象，如果获取 cyfs.NONGetObjectOutputResponse 对象失败，则直接返回失败信息。

```typescript
const gr = await stack.non_service().get_object({
    common: { level: cyfs.NONAPILevel.NOC, flags: 0 },
    object_id: id
});
if (gr.err) {
    const errMsg = `get_object from non_service failed, path(${queryOrderPath}), ${idR}`;
    pathOpEnv.abort();
    return Promise.resolve(makeBuckyErr(cyfs.BuckyErrorCode.Failed, errMsg));
}
```

3. 释放锁后，对 Uint8Array 格式的 Order 对象进行解码，得到最终的 Order 对象，如果解码失败，则直接返回失败信息。

```typescript
pathOpEnv.abort();
const orderResult = gr.unwrap().object.object_raw;
const decoder = new OrderDecoder();
const r = decoder.from_raw(orderResult);
if (r.err) {
    const msg = `decode failed, ${r}.`;
    console.error(msg);
    return Promise.resolve(
        makeBuckyErr(cyfs.BuckyErrorCode.Failed, 'decode order obj from raw excepted.')
    );
}
const orderObj = r.unwrap();
```

## 更新订单路由

-   完整代码见 src/service/routers/update_order.ts

前端发起请求，请求参数为 Order 对象，接口接收后进行处理，响应 ResponseObject 对象。与前文一样，我们只讲解差异部分的代码块。

1. 使用 pathOpEnv 的 get_by_path 方法从旧的 Order 对象的存储路径中获取旧的 Order 对象的 object_id，如果 get_by_path 返回的结果有错误，则直接响应错误信息，如果 get_by_path 返回的结果为空，也直接响应错误信息。

```typescript
const idR = await pathOpEnv.get_by_path(queryOrderPath);
if (idR.err) {
    const errMsg = `get_by_path (${queryOrderPath}) failed, ${idR}`;
    pathOpEnv.abort();
    return Promise.resolve(makeBuckyErr(cyfs.BuckyErrorCode.Failed, errMsg));
}
const id = idR.unwrap();
if (!id) {
    const errMsg = `unwrap failed after get_by_path (${queryOrderPath}) failed, ${idR}`;
    pathOpEnv.abort();
    return Promise.resolve(makeBuckyErr(cyfs.BuckyErrorCode.Failed, errMsg));
}
```

2. 利用新的 Order 对象信息创建对应的 NONObjectInfo 对象，通过 put_object 操作，把 NONObjectInfo 对象更新到 RootState 上。如果 put_object 操作失败，则直接返回失败信息。

```typescript
const nonObj = new cyfs.NONObjectInfo(
    orderObject.desc().object_id(),
    orderObject.encode_to_buf().unwrap()
);
const decId = stack.dec_id!;
const putR = await stack.non_service().put_object({
    common: {
        dec_id: decId,
        level: cyfs.NONAPILevel.NOC,
        flags: 0
    },
    object: nonObj
});
if (putR.err) {
    console.error(`commit put-object failed, ${putR}.`);
    pathOpEnv.abort();
    return putR;
}
```

3. 利用 pathOpEnv，用新的 Order 对象的 NONObjectInfo 对象的 object_id 进行替换旧的 Order 对象的事务操作。如果操作结果有错误，则直接返回错误信息。

```typescript
const objectId = nonObj.object_id;
const rs = await pathOpEnv.set_with_path(queryOrderPath, objectId!, id, true);
console.log(
    `set_with_path(${queryOrderPath}, ${objectId!.to_base_58()}, ${id.to_base_58()}, true), ${rs}`
);
if (rs.err) {
    console.error(`commit set_with_path(${queryOrderPath},${objectId},${id}), ${rs}.`);
    pathOpEnv.abort();
    return rs;
}
```

## 删除订单路由

-   完整代码见 src/service/routers/delete_order.ts

前端发起请求，请求参数为 Order 对象，接口接收后进行处理，响应 ResponseObject 对象。与前文一样，我们只讲解差异部分的代码块。

1. 使用 pathOpEnv 的 remove_with_path 方法，传入将要删除的 Order 对象的 object_id 进行删除 Order 对象的事务操作。如果 remove_with_path 返回的结果有错误，则直接响应错误信息。

```typescript
const rm = await pathOpEnv.remove_with_path(queryOrderPath, objectId);
console.log(`remove_with_path(${queryOrderPath}, ${objectId.to_base_58()}), ${rm}`);
if (rm.err) {
    console.error(`commit remove_with_path(${queryOrderPath}, ${objectId}), ${rm}.`);
    pathOpEnv.abort();
    return rm;
}
```

# 实现创建订单的跨 Zone 通知

前面，我们实现了对 Order 对象的 CRUD 操作，但这些操作都是在同一个 Zone 进行的，也就是说，这些对象数据的变动只有自己知道。
如果我们需要把对象数据发给其他人，应该如何操作呢？
答案是：发起跨 Zone 请求。
从实现原理上讲，就是我们自己的 OOD 向其他人的 OOD 发起请求，传递对象数据，等待对方处理并得到响应结果的过程。

一个完整的跨 Zone 请求，应该分为以下 3 个步骤：

1. 自己的 OOD 发起指向特定 OOD 的请求
2. 对方 OOD 接收请求后进行处理并响应
3. 自己的 OOD 接收到响应结果

## 发起一个跨 OOD 请求

-   完整代码参见 src/service/routers/create_order.ts

细心的小伙伴可能已经注意到，在 create_order.ts 文件源码中，有几行被注释掉的代码：

```typescript
const stackWraper = checkStack();
// If here is the windows simulator environment, C:\cyfs\etc\zone-simulator\desc_list -> zone2 -> people,
// If here is the mac simulator environment, ~/Library/cyfs/etc/zone-simulator/desc_list -> zone2 -> people,
// otherwise, you should use real poepleId.
const peopleId = '5r4MYfFVtnu7yAP5XSZGg8JsqZuzyqozH6oXCLMPb8h8';
await stackWraper.postObject(orderObject, ResponseObjectDecoder, {
    reqPath: ROUTER_PATHS.CREATE_ORDER_REQ,
    decId: stack.dec_id!,
    target: cyfs.PeopleId.from_base_58(peopleId).unwrap().object_id // Here is the difference between the same zone and cross zone.
});
```

这段代码实现了向目标用户 Id 为 5r4MYfFVtnu7yAP5XSZGg8JsqZuzyqozH6oXCLMPb8h8 的 Zone 发起了指定 decId 的 CREATE_ORDER_REQ 的请求，请求参数是当前新建的 Order 对象，期望的响应对象是 ResponseObject，在 5r4MYfFVtnu7yAP5XSZGg8JsqZuzyqozH6oXCLMPb8h8 的 OOD 上需要有一个对应的路由模块来接收、处理并响应结果。

-   需要注意的是，跟一般的同 Zone 内的请求不同的是，我们这里的请求参数中，多指定了一个 target 属性，这个就是目标用户的 PeopleId。

现在，我们去除这几行代码注释,修改代码中的 peopleId 值为你的模拟器第二个 Zone 的 peopleId。这样，我们的创建订单路由就新增了跨 Zone 通知指定 OOD 的功能。
但是，光通知对方是不够的，我们还需要得到对方的响应才行。因此，我们需要开发一个接收创建订单的跨 Zone 通知的路由模块。

## 接收跨 Zone 请求的路由

首先，我们需要明确这个路由模块的基本功能：对方接收到新 Order 对象后，将其存储在/orders/<Order.key>路径下，存储操作成功后返回响应对象。
在 src/service/routers/create_order_req.ts 中，我们准备好了示例代码。
通过对比 create_order_req.ts 与 create_order.ts 的源码，我们很容易发现，create_order_req.ts 比 create_order.ts 只是多了开头的 10 行代码，其他的处理逻辑几乎一模一样。
下面，我们来看看多出来的这 10 行代码：

```typescript
const stack = checkStack().check();
const owner = stack.local_device().desc().owner()!.unwrap();
console.log(`current target -----> ${req.request.common.target?.to_base_58()}`);
if (!owner.equals(req.request.common.target!)) {
    console.log(`should transfer to -> ${req.request.common.target}`);
    return Promise.resolve(
        cyfs.Ok({
            action: cyfs.RouterHandlerAction.Pass
        })
    );
}
```

这里我们详细讲解其中的关键 2 行代码：

1. 第 2 行，通过 Stack 获取到自己的 OwnerId；
2. 第 4 行，判断自己的 OwnerId 是否与请求的 target 值相等，如果相等，说明这个请求还是在自己的 Zone 内，需要将其 Pass 转发到 Zone 外，也就是发给目标 Zone，如果不相等，说明是来自 Zone 外的请求，可以继续处理并响应。

# 在模拟器环境运行 Service

通过前面的学习，我们订单管理系统 Demo 的服务端代码已经全部准备完毕。接下来，我们需要在模拟器环境上跑起来。

## 配置模拟器的 ACL 文件

打开 doc/acl.toml 配置文件，可以看到如下配置信息：

```toml
hello-request-post = { action = "*-post-object", res = "/dec_app/9tGpLNnfhjHxqiTKDCpgAipBVh4qjCDjxYGsUEy5p5EZ/32768", group = { location = "outer" }, access = "accept" }
hello-response-post = { action = "*-post-object", res = "/dec_app/9tGpLNnfhjHxqiTKDCpgAipBVh4qjCDjxYGsUEy5p5EZ/32769", group = { location = "outer" }, access = "accept" }
resp-object-post = { action = "*-post-object", res = "/dec_app/9tGpLNnfhjHxqiTKDCpgAipBVh4qjCDjxYGsUEy5p5EZ/32770", group = { location = "outer" }, access = "accept" }
order-post = { action = "*-post-object", res = "/dec_app/9tGpLNnfhjHxqiTKDCpgAipBVh4qjCDjxYGsUEy5p5EZ/32771", group = { location = "outer" }, access = "accept" }
```

以 hello-request-post 为例，其 res 配置为"/dec_app/9tGpLNnfhjHxqiTKDCpgAipBVh4qjCDjxYGsUEy5p5EZ/32768"，我们需要将其中的 app_id 替换为自己当前的 app_id，当前应用的 app_id 在 cyfs.config.json -> app_id 可以找到。
修改后的配置文件内容如下：

```toml
hello-request-post = { action = "*-post-object", res = "/dec_app/<your app id>/32768", group = { location = "outer" }, access = "accept" }
hello-response-post = { action = "*-post-object", res = "/dec_app/<your app id>/32769", group = { location = "outer" }, access = "accept" }
resp-object-post = { action = "*-post-object", res = "/dec_app/<your app id>/32770", group = { location = "outer" }, access = "accept" }
order-post = { action = "*-post-object", res = "/dec_app/<your app id>/32771", group = { location = "outer" }, access = "accept" }
```

完成 acl.toml 配置文件的修改后，在 mac 下，通过文件选择器找到 ~/Library/cyfs/etc/zone-simulator/desc_list 文件，可以看到模拟器支持 2 个虚拟的 Zone，相关设备信息如下：
**注意**如果没有运行过模拟器，是不会有这些文件的，需要启动模拟器生成了这些文件后再进行操作，启动模拟器参考下文“启动模拟器”

```
zone1 as follows:
people:5r4MYfFSHP9fBMtzQBw3SvRYGztJYaWyTcTitLGWEEjG
ood:5aSixgM14GTqmvNissUUH6f7VQa7FaUX4TKPsfbHrJXn
standby_ood:5aSixgM4L2HvoY1hUKpkCzxnsCo2YMqb2KWHNxya4svq
device1:5aSixgPY7keTbYUqPg7YZfsD7mefWm7tdvwvu9is3GzL
device2:5aSixgRawu212qTshNn9SVFEXYwG6EwzNJuikpFbKYLt

zone2 as follows:
people:5r4MYfFVtnu7yAP5XSZGg8JsqZuzyqozH6oXCLMPb8h8
ood:5aSixgM8tvAoA2pjZCUeEDC6ZBSgcpgHhAYjbJYzqAXB
device1:5aSixgPBfS44McquHBLmeTLnqAcJWd64rNssqXa1rh1k
device2:5aSixgRkiHmmJb6GS4HxZWR3qHsxomgUfqpdDiCMDSui
```

### 在 mac 中配置

在 ~/Library/cyfs/etc/zone-simulator 目录下，找到 zone1 对应 _ood_ 文件夹 5aSixgM14GTqmvNissUUH6f7VQa7FaUX4TKPsfbHrJXn 以及 zone2 对应的 _ood_ 文件夹 5aSixgM8tvAoA2pjZCUeEDC6ZBSgcpgHhAYjbJYzqAXB，分别在这两个文件夹下 **新建** acl 文件夹，将 doc/acl.toml 文件复制粘贴进去即可。

### 在 windows 中配置

在 C:\cyfs\etc\zone-simulator 目录下，同 mac 操作方法。

-   需要注意的是，我们的 desc_list 文件内容可能并不相同，以上操作只是在具体的文件名有所差异，比如，你当前的 zone1 -> ood 项可能并不是 5aSixgM14GTqmvNissUUH6f7VQa7FaUX4TKPsfbHrJXn

## 启动模拟器

### mac

在项目根目录下的 tool/zone-simulator/mac 文件夹下，可以选择 aarch64 或者 x86_64 启动 zone-simulator 模拟器程序。

-   文件可能存在无执行权限问题，执行 `chmod 775 zone-simulator` 后运行即可。

### windows

在项目根目录下的 tool/zone-simulator/windows 文件夹下，启动 zone-simulator.exe 模拟器程序。

## 运行 Service

要运行 Service，我们需要使用 tsc 把项目中的 ts 文件编译成 js，在项目根目录下，打开新的终端，输入以下指令：

```shell
npx tsc
```

接着，把 src/common/objs/obj_proto_pb.js 文件拷贝到 deploy/src/common/objs/obj_proto_pb.js

最后，在根目录输入以下指令，开启模拟器模拟的两个 Zone：

```shell
yarn sim1
yarn sim2
```

-   需要注意的是，这两个指令分别对应两个终端执行。
