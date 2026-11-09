# WeKnora v0.9.0 测试项目

个人测试练习项目。对腾讯开源知识库系统 [WeKnora](https://github.com/Tencent/WeKnora) v0.9.0 的两个新功能模块做了手工功能测试，补充了 5 条 Postman 接口自动化用例，并把发现的 4 个缺陷提交到了 GitHub。

## 测试环境

| 项目 | 内容 |
|---|---|
| 被测版本 | WeKnora v0.9.0 Standard（commit f17ee8a） |
| 部署方式 | Docker Compose，本地部署，http://localhost |
| 对话模型 | 阿里云百炼 qwen-plus |
| 向量模型 | 阿里云百炼 text-embedding-v3 |
| 浏览器 | Microsoft Edge 156 |
| 操作系统 | Windows 11 25H2 |
| 接口工具 | Postman |

## 测试范围

| 模块 | 依据 | 功能点 | 用例 | 结果 |
|---|---|---|---|---|
| 执行权限与审批 | v0.9.0 版本说明 | F1–F10 | T01–T18 | 18 条全部通过 |
| 文档上传解析 | 上传页提示 + v0.9.0 版本说明 | D1–D7 | U01–U20 | 12 通过，8 不通过 |
| 知识库接口（自动化） | 浏览器 F12 抓到的接口 | — | A01–A05 | 15 条断言全部通过 |

先从版本说明和页面提示里整理出"系统承诺了什么"（功能清单），再针对每条承诺设计用例。

## 用到的测试方法

| 方法 | 用在哪里 |
|---|---|
| 判定表 | 3 种执行权限 × 查询 / 改名 / 删除，共 9 种组合（T 系列） |
| 边界值 | 49MB、正好 50M、50M+1B、0 字节文件（U03、U04、U18、U19、U20） |
| 等价类 | 同名不同内容、同内容不同名，反推出去重规则是按内容判断（U10–U12） |
| 控制变量 | 审批显示 0s 的问题，逐个改变等待时长和权限模式，找到复现条件是"等待约 3 分钟后刷新"（Q4） |
| 对比实验 | 立即取消 vs 解析中途取消，确认只有中途取消才会残留数据（U14–U16） |

## 发现的问题

疑点清单共记录 16 条：确认是缺陷 7 条，待确认 6 条，改进建议 3 条。

其中 4 条写了正式缺陷报告，查重后提交到 GitHub：

| Issue | 问题 | 严重程度 |
|---|---|---|
| [#5190](https://github.com/Tencent/WeKnora/issues/5190) | 解析中途取消文档，状态显示"已取消"，但已入库的分块没有清理，仍可被检索 | 中 |
| [#5191](https://github.com/Tencent/WeKnora/issues/5191) | 正好 50M 的文件前端放行，但请求体超过 nginx 的 50M 上限，返回 413 | 低 |
| [#5192](https://github.com/Tencent/WeKnora/issues/5192) | 审批等待约 3 分钟后刷新，状态显示红色"0s"，但审批实际未过期 | 低 |
| [#5193](https://github.com/Tencent/WeKnora/issues/5193) | 只读模式下助手引导用户去开启一个不存在的设置，并称查询操作不可用 | 低 |

## 接口自动化（Postman）

| 编号 | 接口 | 测什么 | 断言 |
|---|---|---|---|
| A01 | GET /knowledge-bases | 正常查询列表 | 200，success 为 true，data 是数组 |
| A02 | POST /knowledge-bases | 正常创建，并保存新知识库 id | 201，success 为 true，名称正确 |
| A03 | POST /knowledge-bases | 名称为空（反向用例） | 400，success 为 false，提示名称不能为空 |
| A04 | GET /knowledge-bases | 不带 Token（反向用例） | 401，提示未认证，不返回数据 |
| A05 | DELETE /knowledge-bases/{id} | 删除 A02 创建的知识库 | 200，success 为 true，提示删除成功 |

- 用环境变量保存 `base_url` 和 `token`，Token 设为 secret 类型，导出的 Collection 里不包含真实 Token
- A02 用 `pm.collectionVariables.set` 保存新知识库的 id，A05 用 `{{kb_id}}` 删除它，所以每次运行都会自己清理测试数据
- 用 Collection Runner 按 A01→A05 顺序一次运行，15 条断言全部通过

## 文件说明

| 文件 / 文件夹 | 内容 |
|---|---|
| 功能清单.md | 从版本说明和页面提示整理出的功能点（F1–F10、D1–D7） |
| 测试记录.md | 全部用例的前置条件、操作、预期、实际结果和结论 |
| 疑点清单.md | 测试中发现的可疑现象，以及确认状态 |
| 缺陷报告.md | 4 份正式缺陷报告（BUG-01～BUG-04），含复现步骤、对比实验和 GitHub 查重记录 |
| screenshots/ | 测试截图，文件名开头的编号和测试记录里的截图编号对应 |
| testfiles/ | 测试用的小文件（正常文件、损坏的 PDF、空文件、取消测试文件等）。49MB/50MB 等大文件体积太大没有上传，可以用缺陷报告 BUG-02 里的 PowerShell 命令生成 |
| postman/ | 导出的 Postman Collection，导入后配置 base_url 和 token 两个环境变量即可运行 |
