# 世界书 · astrbot_plugin_lorebook

给聊天机器人补上它"不知道"的事：人物、地点、习惯、正在发生的情况。
这些内容写在世界书里，命中条件时自动拼进上下文——不占人设、不污染历史、随时可改。

> 版本 `2.1.0` ｜ 适配 AstrBot `>= 4.16` ｜ 依赖 `PyYAML >= 6.0`
> 支持平台：aiocqhttp / telegram / discord（取决于 AstrBot 本体）

---

## 为什么需要它

模型的记忆只到训练截止日，其余全靠上下文。想让角色记住"老陈只在夜里点灯"，
靠人设硬塞会越写越长，靠聊天记录又留不住。世界书就是最小代价的补丁：
只在相关话题出现时注入那几条，平时一句话都不占。

---

## 特性

**触发**
- 关键词：主词 + 次要词，四种逻辑（任一 / 全部 / 非任一 / 非全部）
- 正则：`/角色名|称呼/i` 写进关键词或正则字段直接生效
- cron：5 段表达式定时触发，同一窗口内去重
- 定向：按用户 ID、群号限定
- 常驻：每轮都注入
- 概率：命中后按百分比掷骰

**注入**
- 7 个落点：system 前置 / 后置、AN 顶部 / 底部、按深度、EM 顶部 / 底部
- `order` 排序、`weight` 权重、`depth` 深度
- 状态机：`turns` 持续轮数、`cooldown` 冷却、`decay_minutes` 时间衰减
- `exclusive_group` 互斥组：同组只留一条，适合"多种说法里随机取一个"
- 全局字符预算，超出自动截断

**存储**
- YAML 为主，多行中文不用转义；同时兼容酒馆 World Info JSON，两种格式可混放
- 6 种模板只是字段语法糖：`common` / `resident` / `chance` / `schedule` / `user` / `group`

**自治**
- 内置 LLM 工具 `worldbook_add_entry`，模型能自己沉淀记忆
- 但**只写进待审核区**，管理员批准后才落盘生效——自治必须有闸门，否则几百轮后就是垃圾场

**WebUI 管理台**
- 书库列表、条目搜索、模板筛选、实时统计
- 全字段可视化编辑，改完即写盘并热重载
- 拖拽导入 `.json` / `.yaml`，可选合并或替换
- 一键导出、复制、重命名、删除世界书
- 群组绑定与注入参数的图形化设置
- 待审核条目逐条批准或驳回

---

## 安装

1. 把 `astrbot_plugin_lorebook` 文件夹放进 `data/plugins/`（或直接用插件市场的 zip 安装）。
2. 在面板里启用插件，首次启用会自动安装 `PyYAML`。
3. 打开插件详情页 →「世界书管理」，网页里就能建书写条目。

---

## 快速上手

### 方式一：网页里写（推荐）

1. 进入管理台，左侧「＋」新建一本世界书，例如 `mylife`。
2. 点亮顶部「启用」开关，让它参与注入。
3. 点条目栏「＋」新建条目，填标题、关键词、正文，保存。

### 方式二：直接放文件

把 YAML 丢进 `data/plugins/astrbot_plugin_lorebook/worlds/`，然后 `/世界书 reload`：

```yaml
- name: 角色：守塔人老陈
  template: common
  keys: [灯塔, 守塔人]
  secondary: [晚上, 夜]
  logic: any
  content: |
    【角色】老陈，六十岁，右手有旧伤。
    只在夜里点亮灯塔，白天闭门不见客。
  position: at_depth
  depth: 2
  order: 120
```

### 方式三：导入现成的

管理台点「导入」，把酒馆导出的 World Info JSON 拖进去即可，字段会自动映射。

---

## 目录结构

```
astrbot_plugin_lorebook/
├─ main.py                      # 入口：触发、注入、指令、LLM 工具
├─ web_api.py                   # 管理台后端接口
├─ metadata.yaml                # 插件元信息
├─ _conf_schema.json            # 配置项定义
├─ requirements.txt
├─ pages/
│  └─ lorebookManage/
│     ├─ index.html             # 管理台（单文件，无外链资源）
│     └─ _page.json
└─ worlds/                      # 世界书目录（可在配置里改）
   ├─ demo.yaml                 # YAML 示例
   ├─ example.json              # 酒馆 JSON 示例
   ├─ _pending.json             # 待审核队列（运行时生成）
   └─ auto.yaml                 # 批准后的条目（运行时生成）
```

---

## 配置

| 配置项 | 默认 | 说明 |
|---|---|---|
| `worldsDir` | 空 | 世界书目录，留空用插件自带 `worlds/` |
| `activeWorlds` | `[]` | 全局启用的书（不含后缀），也可在管理台勾选 |
| `groupBindings` | `{}` | `{"群号": ["书A"]}`，键写 `*` 表示全局兜底 |
| `scanDepth` | 4 | 扫描最近几条消息作为触发文本 |
| `budgetChars` | 2000 | 单轮注入的字符预算 |
| `caseSensitive` | `false` | 关键词区分大小写 |
| `matchWholeWords` | `false` | 整词匹配，避免"猫"命中"熊猫" |
| `recursive` | `false` | 是否用已注入内容继续触发其它条目 |
| `maxRecursionSteps` | 0 | 递归层数上限，0 表示不限 |
| `cronWindowSeconds` | 90 | cron 命中后的去重窗口，防止一轮内重复触发 |
| `selfWriteEnabled` | `true` | 允许 LLM 往待审核区写条目 |
| `adminOnly` | `true` | 重载 / 测试 / 审批仅管理员可用 |
| `debugLog` | `false` | 输出匹配过程日志，排查没触发时打开 |

---

## 条目字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | string | 条目标题，留空时用第一个关键词 |
| `template` | string | `common` / `resident` / `chance` / `schedule` / `user` / `group` |
| `content` | string | 实际注入的文本 |
| `keys` | list | 主关键词，逗号分隔；可写 `/正则/i` |
| `secondary` | list | 次要关键词，配合 `logic` 复核 |
| `logic` | string | `any` / `all` / `not_any` / `not_all` |
| `regex` | list | 额外正则触发条件 |
| `constant` | bool | 常驻，每轮注入 |
| `probability` | int | 命中后的触发概率，1–100 |
| `cron` | string | 定时表达式，如 `30 7 * * *` |
| `users` / `groups` | list | 只对这些人 / 群生效 |
| `position` | string | `before` / `after` / `an_top` / `an_bottom` / `at_depth` / `em_top` / `em_bottom` |
| `depth` | int | `at_depth` 时插入到倒数第几条消息 |
| `order` | int | 同位置下的排序，越大越靠前 |
| `weight` | float | 预算不足时按权重取舍 |
| `turns` | int | 触发后保持生效的轮数 |
| `cooldown` | int | 触发后冷却的轮数 |
| `decay_minutes` | int | 闲置多少分钟后状态衰减 |
| `exclusive_group` | string | 互斥组名，同组只取一条 |
| `disable` | bool | 停用，不参与匹配 |

酒馆 v2 字段会自动映射：`key` → `keys`，`keysecondary` → `secondary`，
`comment` → `name`，`selectiveLogic` → `logic`，`disable` 原样保留。

### 模板与字段的对应

| 模板 | 含义 | 实际落到 |
|---|---|---|
| `common` | 普通关键词触发 | `keys` + `secondary` + `logic` |
| `resident` | 常驻 | `constant = true` |
| `chance` | 概率触发 | 未设概率时默认 50 |
| `schedule` | 定时 | 需要填 `cron` |
| `user` | 定向用户 | `users` |
| `group` | 定向群 | `groups` |

---

## 指令

| 指令 | 权限 | 说明 |
|---|---|---|
| `/世界书` 或 `/世界书 list` | 所有人 | 列出书库，`✓` 表示当前生效 |
| `/世界书 reload` | 管理员 | 重新扫描目录 |
| `/世界书 test <文本>` | 管理员 | 用一段话模拟触发，看命中了哪些条目 |
| `/世界书 pending` | 所有人 | 查看待审核队列 |
| `/世界书 approve <序号>` / `approve all` | 管理员 | 批准并写入 `auto.yaml` |
| `/世界书 reject <序号>` / `reject all` | 管理员 | 驳回 |

---

## 常见问题

**改了文件但没生效**
执行 `/世界书 reload`，或在管理台任意保存一次（会自动重载）。

**条目写了却从不触发**
1. 这本书有没有启用（管理台开关或 `activeWorlds`）；
2. `/世界书 test <你的话>` 看命中情况；
3. 打开 `debugLog` 看匹配日志；
4. 检查 `groupBindings` 是否把当前群指向了别的书。

**注入内容太长**
调小 `budgetChars`，或把条目的 `weight`、`order` 调低，让重要的先占预算。

**待审核一直涨**
那是模型在自说自话。可以关掉 `selfWriteEnabled`，或者定期 `reject all`。

**想给不同群不同设定**
用 `groupBindings`，或给条目填 `groups` 字段做更细的定向。

---

## 许可

MIT License，见 [LICENSE](LICENSE)。有 bug 或想要的功能，欢迎提 issue。
