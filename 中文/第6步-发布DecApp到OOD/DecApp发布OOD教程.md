# 说明

本教程基于 cyfs-dapp-cli 脚手架创建的模板项目进行讲解。

# 重要的配置文件

在项目的根目录下，我们重点关注 cyfs.config.json 和 service_package.cfg 这两个文件。

## cyfs.config.json

app_name: 在 cyfs create -n 中输入的名字，也是该 app 的名字。同一个 owner 的 app 名字不能相同。支持中文等非 ASCII 字符，这个名字会显示在应用商店和 App 管理界面上
version: 表示 App 的当前版本，当执行 cyfs deploy 时，会打包并部署 App 的当前版本，部署后，会将该版本的第三位自动加一，作为新的当前版本
description：App 描述，每次部署时，此处填写的描述文字会更新到 App 对象中，在应用商店上显示。
icon: App 图标，只支持 cyfs 链接。每次部署时更新，在应用商店和 App 管理界面上展示
service：配置 Dec App Service 相关打包和运行配置，详情下述：

1. pack: 指示打包 Service 时，需要打包哪些目录
2. type: 指定 service 的类型，目前支持"node"和"rust"两种
   2.1 当指定为"node"时，会将 service 的打包目录，cyfs 目录，package.json 文件一起拷贝到${dist}/service/${target}目录下
   2.2 当指定为"rust"时，只将 service 的打包目录拷贝到${dist}/service 下，service 的编译等工作需用户事先手动执行
   2.3 dist_targets：指定 service 支持哪些平台，默认和 ood 支持的平台一致。如果在 OOD 支持，但 service 不支持的平台上安装该 Dec App，会返回安装失败错误
   2.4 app_config: 配置 service 的运行配置。可以通过添加[target_name]:[config_path]字段的形式，给每个平台配置不同的运行配置。默认的 default 表示当没有该平台对应配置时，使用的默认配置

web: 配置 Dec App Web 部分的相关打包和运行配置，详情下述

1. folder: 配置 Dec App Web 部分的根目录
2. entry: 可配置 Web 首页的文件名，当前该配置修改无效，首页文件名必须为 index.html

dist: 打包临时目录的名字，默认为"dist"。如果该默认值与其他本地目录冲突，可以修改为其他不冲突的目录名
ext_info：与 App 部署，安装，运行无关的其他配置，一般与 App 的展示有关。详情下述：

1. medias: 配置 App 相关的媒体信息，目前应用商店的 DecApp 详情页会使用该信息
2. list: 配置 DecApp 详情页中展示的应用截图。只支持 cyfs 链接，展示的截图顺序与配置顺序相同

app_id：由 owner id 和 app_name 计算得来的 app id，不能自行修改。在代码中任何需要使用 Dec App Id 的地方，需要用这里展示的 ID。

## service_package.cfg

install：安装脚本
start：启动脚本
stop：停止脚本
status：运行状态查询脚本

# 发布前的代码调整

在项目的前端和服务端功能开发完成并在模拟器环境调试完毕后，即可准备将其发布到自己的 OOD 上。在正式发布前，我们需要对前端和服务端的部分涉及使用模拟器的代码进行修改，以满足发布后的环境需求。

## 前端部分

调整 src/www/initialize.ts 代码,将模拟器模式切换为 OOD 模式，调整后的代码如下：

```typescript
export async function init() {
    useSimulator(SimulatorZoneNo.REAL, SimulatorDeviceNo.FIRST);
    await waitStackRuntime(DEC_ID);
}
```

## 服务端部分

调整 src/service/routers/create_order_req.ts 代码，将跨 Zone 通知中的对方 peopleId 字符串，由模拟器 2 的 peopleId 改为真实环境下的目标用户的 peopleId。

# 打包项目

由于我们模板项目的前端部分是使用 React 框架进行开发，因此，需要使用 webpack 进行打包。
在项目根目录下，运行如下指令：

```shell
yarn build
```

指令执行完，可以在项目根目录下看到新增了 dist 和 deploy 文件夹：

-   dist: 前端打包产物，由 webpack 生成
-   deploy: 项目中所有 typescript 编译后的 js 文件，由 tsconfig.json 配置

# 发布 Dec App 到 OOD(待)

接下来，我们先打开 CYFS 浏览器再运行如下指令:

-   mac

```shell
yarn mac-deploy-pre
yarn deploy
```

如果过程中出现以下错误：

```
[error],[2022-09-14 19:39:09.175],<>,owner mismatch, exit deploy., cyfs.js:389
```

这个报错代表当前的 owner 与应用的 owner 不匹配，我们需要手动修改应用的 owner，在项目根目录打开终端，输入以下指令：

```shell
cyfs modify -o ~/.cyfs_profile/people
yes
```

执行命令，打印出 _save app object success_ 的字样，代表修改成功。此时，我们重新走一遍发布流程即可。

-   windows

```shell
yarn deploy
```

最终，终端会显示上传的信息，上传完成后，终端显示如下信息：

```
Upload DecApp Finished.
CYFS App Install Link: cyfs://5r4MYfFbqqyqoA4RipKdGEKQ6ZSX3JzNRaEpMPKiKWAQ/9tGpLNnbNtojWgQ3GmU2Y7byFm7uHDr1AH2FJBoGt5YF
```

恭喜你，这代表我们的 Dec App 已经成功发布到了 OOD。

# 配置 ACL 生效，nightly 才需要(linux 实体 OOD 为例)

**注意**其他类型的 OOD 也是类似操作

1. 远程登录 OOD
2. 把当前 dec app 所需的 acl 内容追加至/cyfs/etc/acl/acl.toml 文件末尾
3. 重启 gateway 使最新的 acl 生效

# 去 CYFS 浏览器中安装 Dec App 并查看

1. 复制 CYFS App Install Link 后面的这串 CYFS 链接，去 CYFS 浏览器中打开 Dec App Store 页面(cyfs://static/DecAppStore/app_store_list.html)，点击*通过 URL 安装*按钮，把刚才 Dec App 的安装链接粘贴进去，安装即可。
2. 安装完成后，可以去应用已安装列表页面(cyfs://static/DecAppStore/app_installed_list.html)看到刚刚安装好的 Dec App。
3. Dec App 的前端入口，可以在 CYFS 浏览器首页(cyfs://static/browser.html)或者已安装列表找到，点击即可查看前端效果。
