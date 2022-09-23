# 准备条件

1. 请确保已使用 Cyber Chat 创建好 DID。
2. 请确保已经安装好 cyfs-tool。如果是 nightly 版本，请安装 cyfs-tool-nightly。
   指令如下：

```shell
npm i -g cyfs-tool
```

# 注意事项

请注意以下 2 点：

1. 导入身份时，Cyber Chat 所在手机和电脑必须在同一个局域网，才能导入成功。
2. 控制台请使用等宽字体，否则展示的二维码可能无法扫描或扫描出错

# 导出身份到电脑

在电脑的终端，输入以下指令：

```shell
cyfs import-people
```

**注意** 导出的身份文件，默认保存在%USERHOME%/.cyfs_profile，如需更改保存位置，可以使用： -s, --save 新文件夹路径
执行该命令后，会在控制台展示一个二维码。
使用 Cyber Chat 的"扫一扫"功能，扫描该二维码，即可将 people 身份文件导入到电脑上，后续可以用该身份文件创建，部署工程。导入的身份文件名为 people.desc, people.sec
