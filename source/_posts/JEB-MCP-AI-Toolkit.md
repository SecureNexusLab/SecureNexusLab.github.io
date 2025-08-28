---
layout: page
title: 用 JEB + MCP 把逆向装进你的 AI 工具箱，京东獬豸信息安全实验室开源 JEB-MCP
author: SecureNexusLab
cover: true
sidebar: []
readmore: true
date: 2025-08-28 19:50:00
tags: 
- 公众号推文
- 二进制安全
categories:
- 公众号推文
---
# 用 JEB + MCP 把逆向装进你的 AI 工具箱，京东獬豸信息安全实验室开源 JEB-MCP

> 让你的 Cline / Cursor / RooCode 直接“会用”JEB，自动拉取反编译代码、分析调用关系、重命名符号，提升 APK 分析效率。

## 工具简介

- 名称：JEB-MCP
- 开箱即用：为 JEB Pro 提供本地服务与插件，接入主流 AI 编码助手即可远程操控 JEB。
- 能力覆盖关键场景：清单解析、导出组件识别、类/方法反编译、调用/重写关系、批量重命名等。
- 面向自动化：通过 MCP（Model Context Protocol）暴露稳定工具集，让助手按“任务流”持续推进审计工作。

## 它能做什么？

在 AI 工具里直接下发任务，调用 JEB 的常用能力，包括但不限于：

- 读取工程信息：
- 读取 AndroidManifest.xml
- 获取所有导出 Activity、统计数量、按索引读取
- 代码理解与导航：
- 获取类/方法反编译代码
- 获取方法调用者、重写关系
- 查询父类、接口、类的方法/字段列表
- 批量重命名：
- 重命名类名、方法名、字段名（便于语义化工程）

## 适用人群

- 逆向工程/安全审计工程师（Android 方向）
- 希望把“半自动审计”升级为“AI 驱动自动化”的团队/个人
- 用 Cline / Cursor / RooCode 做重命名、注释补全、调用链追踪的开发者

## 一句话说明原理

- 在 `JEB` 内部运行一个 `Python 2.7` 脚本插件（MCP.py），启动本地 `JSON-RPC` 服务。
- 在外部用一个轻量 `MCP` 服务器（server.py）把这套 `JSON-RPC` 能力“桥接”给你的 AI 工具。
- 你的助手就能像“调用 API”一样调用 JEB 的逆向能力。

* * *

## 相似项目推荐

- IDA Pro 集成
- ida-pro-mcp
- 地址：https://github.com/mrexodia/ida-pro-mcp
- 特点：为 IDA 提供 MCP 能力桥接，与本项目理念相同，适合 IDA 用户将逆向工作流接入 AI 助手。
- Ghidra 集成
- GhidraMCP
- 地址：https://github.com/LaurieWired/GhidraMCP
- 特点：为 Ghidra 提供 MCP 服务器与插件，涵盖反编译、重命名、注释、原型与类型设置、导入导出等多种工具；适合二进制逆向场景。

如果你的日常栈是 IDA/Ghidra，以上两个项目可以无缝参考迁移思路；若偏 Android，jebmcp 则更贴近 APK/Dex 的高频需求。

项目链接获取可关注以下公众号回复 JEB-MCP

# 安装使用

> 以下内容主要来自该仓库的 README文件：

## 环境要求
- Python >= 3.11  
- 安装 uv https://docs.astral.sh/uv/getting-started/installation/

你需要已安装的 JEB Pro，并确保可运行脚本（含 jython 文件）。


## 3 分钟快速开始

1. 复制脚本到 JEB
- 将仓库中的 src/jeb_mcp/MCP.py 复制到 $JEB_INSTALLATION_DIR/scripts
- 确保 jython 相关文件在该目录可用

2. 在 JEB 中启动脚本

- JEB 菜单：File -> Scripts -> Scripts selector...
- 选择并运行 MCP.py，正常输出示例：
- 
```
    [MCP] Plugin loaded  
    [MCP] Plugin running  
    [MCP] Server started at http://127.0.0.1:16161
```

3. 在你的 AI 工具中配置 MCP
- 将本地 MCP 服务加入到 Cline/Cursor/RooCode 等工具的 MCP 配置中（可参考仓库示例）
- 然后就可以在对话中下发类似「连接 MCP JEB」「分析某 Activity」「批量重命名短名方法」等任务
## 详细接入指南

1. `VS Code`打开工程后，目录结构大致如下：
```
    ├── README.md  
    ├── jeb-mcp  
    │   ├── pyproject.toml  
    │   ├── src  
    │   │   └── jeb_mcp  
    │   │       ├── MCP.py  
    │   │       ├── server.py  
    │   │       └── server_generated.py  
    │   └── uv.lock  
    └── sample_cline_mcp_settings.json`
```
2. 在 VS Code 左侧点击 RooCode 图标，打开右上角 `...` 菜单，选择“编辑项目 MCP”

- 工具会在当前目录生成 `.roo/mcp.json`

3. 将 `.roo/mcp.json` 修改为如下示例（核心是通过 uv 启动 server.py）：

```
    {  
      "mcpServers":{  
        "jeb":{  
          "command":"uv",  
          "args":["--directory","jeb-mcp/src/jeb_mcp","run","server.py"],  
          "timeout":1800,  
          "disabled":false,  
          "autoApprove":[  
            "ping",  
            "check_connection",  
            "get_manifest",  
            "get_all_exported_activities",  
            "get_exported_activities_count",  
            "get_an_exported_activity_by_index",  
            "get_class_decompiled_code",  
            "get_method_decompiled_code",  
            "get_method_overrides",  
            "get_method_callers",  
            "get_superclass",  
            "get_interfaces",  
            "get_class_methods",  
            "get_class_fields",  
            "rename_class_name",  
            "rename_method_name",  
            "rename_field_name"  
          ],  
          "alwaysAllow":[  
            "check_connection",  
            "get_class_decompiled_code",  
            "get_class_fields",  
            "ping",  
            "get_manifest",  
            "get_all_exported_activities",  
            "get_exported_activities_count",  
            "get_an_exported_activity_by_index",  
            "get_method_decompiled_code",  
            "get_method_callers",  
            "get_method_overrides",  
            "get_superclass",  
            "get_interfaces",  
            "get_class_methods",  
            "rename_class_name",  
            "rename_method_name",  
            "rename_class_field"  
          ]  
        }  
    }  
    }
```
4. 回到 JEB，菜单 File -> Scripts -> Scripts selector... 选中并运行 `jeb-mcp/src/jeb_mcp/MCP.py`

- 在 JEB 的 Logger 中看到如下输出即表示成功：
```
    [MCP] Plugin loaded  
    [MCP] Plugin running  
    [MCP] Server started at http://127.0.0.1:16161
```

5. 在 RooCode 对话中直接下发任务，例如：


- 连接MCP JEB  
- 分析 D:\xxx.apk 中 Lnet/xxx/MainActivity; 的功能  
- 根据功能重命名所有方法名长度小于 3 的方法  
- 若调用了其他类的方法，递归分析相关类并同样重命名  
- 输出完整分析过程
## 在 Cline / Cursor 中使用

- 新增一个 MCP 服务，命令指向 uv，并传入以下参数启动：`--directory jeb-mcp/src/jeb_mcp run server.py`
- 运行前确保已在 JEB 中启动了 MCP.py 脚本，并且控制台输出包含：
```
    [MCP] Server started at http://127.0.0.1:16161
```

- 之后即可在对话里让助手调用 JEB 执行分析、获取反编译代码、重命名等操作
- 具体配置项可参考仓库附带的示例文件：`sample_cline_mcp_settings.json`

## 演示

使用 JEB-MCP对APK漏洞分析的演示图：

![图中展示了使用JEB-MCP对APK漏洞分析](/images/JEB-MCP-AI-Toolkit/1.png)



## 常见问题（FAQ）

- Q: 需要什么版本的 Python？
- A: Python 3.11 及以上版本。
- Q: 运行不起来怎么办？
- A: 先用 uv 安装依赖；在 JEB 中确认脚本已加载并看到“Server started”日志；确认 16161 端口未被占用。
- Q: JEB 里需要什么额外组件？
- A: 确保脚本目录中 jython 文件可用，以便脚本正常运行。
- Q: 安全性如何？
- A: 服务默认本地启动；建议在本机使用，避免暴露到不受控网络。

## 常态化招新

一、填报志愿与上传简历

投递渠道为团队邮箱：securenexuslab@gmail.com，投递简历主要包含两个方面内容：

1、个人方向以及意向发展方向；

2、简历PDF或其他材料。

合并为一个 PDF 发送团队邮箱，简历投递命名可以标注一下感兴趣的方向（可多列）

二、面试安排

每个月固定十五号对上个月投递的简历进行筛选，一周内进行面试通知。
