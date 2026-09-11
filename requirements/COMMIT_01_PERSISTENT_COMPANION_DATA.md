# Commit 1：持久化队友角色加载

版本：0.1

状态：待实现

## 1. 背景

当前 SoloMapling 机器人是临时内存角色：

1. 每次创建机器人时读取数据库中的固定模板角色 CID 2。
2. 在内存中分配 `20000+` 的合成 ID。
3. 随机生成名字、等级、职业、性别、外观和装备。
4. 机器人运行期间的所有变化只保存在内存。
5. 机器人不进入普通角色自动保存链路。
6. 服务器重启后，机器人全部丢失，并重新随机生成。

这个模式适合环境人口和演出机器人，但不适合作为长期队友。

Commit 1 要解决的首要问题是：

> 让指定队友使用数据库中的真实 Character，运行期间可以正常保存，重启后能够恢复到原来的角色状态。

## 2. 目标

### 2.1 核心目标

- 在需求目录或服务端指定位置维护 `account_use.txt`。
- `account_use.txt` 初始包含 5 个账号。
- 服务端启动时读取这些账号。
- 根据账号从数据库中解析对应 Character。
- 使用真实 Character ID、名字、等级、职业、属性和资产初始化机器人。
- 不再为这些角色随机生成身份和基础数据。
- 运行期间允许角色数据正常落盘。
- 服务端正常关闭或机器人移除前保存角色。
- 下一次启动时继续加载同一个 Character，而不是重新生成。

### 2.2 MVP 支持范围

- 只支持持久化队友模式。
- MVP 固定一个 World 和一个 Channel。
- MVP 假设一个账号只对应一个队友 Character。
- MVP 建议只加载 5 个队友。
- MVP 可以暂时关闭现有随机环境机器人。
- MVP 保留现有 `TrainingBot` 或 `FollowerBot` 作为行为控制器。
- 后续由 CompanionBot 统一接管行为。

### 2.3 非目标

- 本轮不实现外部管理 UI。
- 本轮不实现 AP、SP、技能和装备管理 API。
- 本轮不实现真实 HP、死亡和复活。
- 本轮不实现月妙、Boss Profile 或新的副本 AI。
- 本轮不解决不同账号角色的多频道独立 Client 问题。
- 本轮不将 19 种机器人类型全部改造成持久化队友。

## 3. 当前实现分析

### 3.1 当前创建流程

当前入口位于：

- [BotGeneration.java](/home/code/maplestory/SoloMapling/src/main/java/soloMapling/ArtificialPlayer/BotGeneration.java:83)
- [BotTypeManager.java](/home/code/maplestory/SoloMapling/src/main/java/soloMapling/ArtificialPlayer/BotTypeManager.java:46)
- [Server.java](/home/code/maplestory/SoloMapling/src/main/java/net/server/Server.java:962)

当前流程为：

```text
EnvironmentManager
    -> BotGeneration.createBot()
    -> Character.loadCharFromDB(2, botClient, false)
    -> set synthetic bot ID
    -> randomize name, level, job and appearance
    -> add to Channel and World
    -> create BotSM
    -> start scheduled BotSM tick
```

### 3.2 当前模板数据

模板数据由以下迁移创建：

[162-fmbot-data.sql](/home/code/maplestory/SoloMapling/src/main/resources/db/data/162-fmbot-data.sql:1)

模板具有以下特征：

```text
Account: fmbot
Character ID: 2
Name: fmbot
Level: 1
Job: Beginner
Equipped items: none
```

这不是为每个队友单独维护的角色，而是所有临时机器人的公共模板。

### 3.3 当前机器人身份判断

[BotHelpers.java](/home/code/maplestory/SoloMapling/src/main/java/soloMapling/ArtificialPlayer/BotHelpers.java:37) 当前通过 ID 判断机器人：

```java
private static boolean isArtificial(int id) {
    return id > 20000;
}
```

这个规则会阻止低 ID 的真实 Character 被识别为机器人，因此 Commit 1 必须改为显式注册。

### 3.4 当前保存行为

[Character.loadCharFromDB()](/home/code/maplestory/SoloMapling/src/main/java/client/Character.java:6900) 只有最后一个参数为 `true` 时才设置：

```java
ret.loggedIn = true;
```

当前机器人使用：

```java
Character.loadCharFromDB(cid, getBotClient(), false)
```

因此机器人的 `Character.loggedIn` 始终为 `false`。

[saveCharToDB()](/home/code/maplestory/SoloMapling/src/main/java/client/Character.java:8298) 的第一段逻辑是：

```java
if (!loggedIn) {
    return;
}
```

所以临时机器人当前不会保存。

### 3.5 当前自动保存

[CharacterAutosaverTask.java](/home/code/maplestory/SoloMapling/src/main/java/net/server/task/CharacterAutosaverTask.java:34) 会遍历在线角色，并仅保存：

```java
chr != null && chr.isLoggedin()
```

因此持久化队友必须进入与普通在线角色一致的保存状态，或者增加专门的持久化标记。

### 3.6 当前机器人移除

[BotGeneration.removeBotFromServer()](/home/code/maplestory/SoloMapling/src/main/java/soloMapling/ArtificialPlayer/BotGeneration.java:183) 当前只执行：

- 从地图移除
- 从 Channel 移除
- 从 World PlayerStorage 移除
- 从 CharacterStorage 移除
- 清理 Buff 和请求状态

它不会调用角色保存。持久化队友移除前必须先保存。

## 4. account_use.txt 需求

### 4.1 MVP 文件格式

MVP 每行一个账号名：

```text
team_account_1
team_account_2
team_account_3
team_account_4
team_account_5
```

支持：

- 空行
- `#` 开头的注释
- 前后空格清理

### 4.2 账号对应的角色

MVP 约束：

```text
一个账号只能有一个队友 Character
```

解析账号后，通过以下逻辑查询：

```sql
SELECT c.id, c.name, c.world
FROM accounts a
JOIN characters c ON c.accountid = a.id
WHERE a.name = ?
ORDER BY c.id
LIMIT 2
```

处理规则：

- 查询到 0 个角色：记录错误并跳过。
- 查询到 1 个角色：加载该角色。
- 查询到多个角色：MVP 拒绝加载并输出明确错误。
- 后续版本扩展为 `account_name,character_id` 或 `account_name=character_name`。

### 4.3 文件位置

默认位置：

```text
requirements/account_use.txt
```

允许通过配置覆盖：

```yaml
USE_PERSISTENT_COMPANIONS: true
COMPANION_ACCOUNT_FILE: requirements/account_use.txt
COMPANION_WORLD: 0
COMPANION_CHANNEL: 1
COMPANION_BOT_TYPE: FOLLOWER_BOT
SPAWN_BOTS_ON_STARTUP: false
```

## 5. 目标启动流程

```text
Server 初始化数据库和 Channel
    -> 初始化 Headless Client
    -> 判断 USE_PERSISTENT_COMPANIONS
    -> 读取 COMPANION_ACCOUNT_FILE
    -> 解析账号列表
    -> 查询账号对应的 Character ID
    -> 跳过无效账号并记录错误
    -> 检查 Character 是否已经在线
    -> 从数据库加载真实 Character
    -> 保留 DB 中的名字、等级、职业、属性、技能、背包和装备
    -> 注册到 Map、Channel、World 和 PersistentBotRegistry
    -> 创建并启动指定 BotSM
    -> 持久化队友进入保存管理
```

如果配置指定关闭随机环境机器人：

```text
SPAWN_BOTS_ON_STARTUP=false
    -> 不调用 EnvironmentManager.environmentLoadStartup()
```

如果同时启用两种模式，必须保证：

- 临时机器人继续使用合成 ID。
- 持久化队友使用真实 Character ID。
- 两类机器人通过显式注册表区分。

## 6. 角色加载需求

### 6.1 使用真实 Character ID

持久化队友必须保留数据库 ID：

```text
数据库 Character ID == 内存 Character.getId()
```

禁止：

```java
bot.setID(BOT_BASE_ID + currentBotCount.getAndIncrement())
```

### 6.2 保留真实角色数据

加载后不得覆盖：

- 名字
- 等级
- 经验
- 职业
- 性别
- 力量和敏捷等属性
- HP 和 MP
- AP 和 SP
- 已学习技能
- 背包
- 装备
- 外观
- 地图

禁止对该角色调用：

```java
setBotVariables(bot)
```

除非这是明确的管理操作，而不是启动随机化。

### 6.3 避免重复上线

加载前必须检查：

- Channel PlayerStorage
- World PlayerStorage
- PersistentBotRegistry

如果 Character 已经在线或者已经被其他机器人加载，必须跳过并输出错误。

### 6.4 World 和 Channel

MVP 要求：

```text
account_use.txt 中的角色必须属于同一个 World
统一加载到 COMPANION_WORLD / COMPANION_CHANNEL
```

如果 Character 的数据库 World 与配置不一致：

- 拒绝加载
- 输出包含 Character ID、数据库 World 和目标 World 的错误

## 7. 机器人身份注册

### 7.1 新增持久化队友注册表

必须增加显式注册，不能继续依赖 ID 大小：

```text
PersistentCompanionRegistry
    Set<Integer> characterIds
    Set<Integer> accountIds
```

至少提供：

```java
register(Character character)
unregister(int characterId)
isPersistentCompanion(int characterId)
getAll()
```

### 7.2 修改 isBot

`isBot` 必须改成：

```text
临时机器人注册表
    OR
持久化队友注册表
```

不能继续使用：

```text
id > 20000
```

否则低 ID 真实 Character 会在以下模块中被误判：

- 组队
- 交易
- 移动响应
- 地图观察
- Buff
- 自动保存
- 超时任务
- 环境管理
- Buff 请求

### 7.3 getCharFromChannelStorage

当前实现固定查询 World 0 / Channel 1，并在 ID 小于 1000 时加 20000。

持久化队友需要新的查询入口：

```text
getCompanionById(realCharacterId)
```

或让通用查询支持：

```text
World
Channel
真实 Character ID
```

不能对真实 ID 执行 `+20000`。

## 8. 保存需求

### 8.1 允许保存

加载持久化队友时，必须让角色满足保存条件。

备选实现：

```text
方案 A:
Character.loadCharFromDB(characterId, botClient, true)
```

缺点：

- 会执行频道角色加载逻辑
- 可能从数据库恢复队伍和信使
- 与临时机器人共享 Client 时语义复杂

```text
方案 B:
增加明确的机器人持久化标记
Character.setPersistentBot(true)
```

推荐方案 B：

```text
loggedIn 或 persistentSaveEnabled 为 true
自动保存允许
普通客户端登录逻辑仍然不依赖网络连接
```

具体字段名在实现时决定。

### 8.2 保存时机

至少覆盖：

- 定期自动保存
- 修改 AP
- 修改 SP
- 学习或提升技能
- 装备、卸下和替换装备
- 交易成功
- 拾取或丢失物品
- 等级、经验和属性变化
- 机器人主动下线
- GM 移除机器人
- 服务器正常关闭

### 8.3 自动保存

普通角色自动保存周期为 1 小时。

持久化队友建议增加更短的独立周期：

```text
60 秒到 5 分钟可配置
```

MVP 采用 5 分钟即可，同时保留关键操作后的立即保存。

### 8.4 保存冲突

保存角色时：

- 必须锁定角色状态
- 不得同时执行装备替换
- 不得同时完成交易
- 不得同时执行背包移动
- 保存失败必须记录 Character ID 和异常

### 8.5 关闭时保存

服务器关闭流程必须：

```text
停止新的 BotSM tick
    -> 等待正在执行的物品操作结束
    -> 保存所有持久化队友
    -> 从 World 和 Channel 移除
    -> 再关闭 World
```

需要接入：

- [Server.shutdownInternal()](/home/code/maplestory/SoloMapling/src/main/java/net/server/Server.java:1936)
- [World.shutdown()](/home/code/maplestory/SoloMapling/src/main/java/net/server/world/World.java:2148)

## 9. 启动和配置

### 9.1 新增配置

建议增加：

```yaml
USE_PERSISTENT_COMPANIONS: false
COMPANION_ACCOUNT_FILE: requirements/account_use.txt
COMPANION_WORLD: 0
COMPANION_CHANNEL: 1
COMPANION_BOT_TYPE: FOLLOWER_BOT
COMPANION_SAVE_INTERVAL_SECONDS: 300
```

### 9.2 启动模式

```text
USE_PERSISTENT_COMPANIONS=false
    -> 保持当前随机机器人行为

USE_PERSISTENT_COMPANIONS=true
    -> 读取账号文件
    -> 加载持久化队友
```

`SPAWN_BOTS_ON_STARTUP` 独立控制是否继续生成环境机器人。

MVP 推荐：

```yaml
USE_PERSISTENT_COMPANIONS: true
SPAWN_BOTS_ON_STARTUP: false
```

### 9.3 启动失败策略

- 单个账号无效，不阻止其他账号加载。
- 所有账号都无效，服务端继续启动并输出错误。
- 账号文件不存在，输出错误并跳过持久化队友。
- 关闭持久化队友模式后，不读取账号文件。

## 10. 文件与模块改动

| 文件或模块 | 改动 |
|---|---|
| `requirements/account_use.txt` | 新增 5 个账号示例 |
| `config.yaml` | 增加持久化队友配置 |
| `Server.java` | 启动加载、关闭保存、模式切换 |
| 新 `PersistentCompanionManager` | 文件解析、加载、启动、保存、关闭 |
| 新 `PersistentCompanionRegistry` | 显式机器人身份和账号注册表 |
| `BotGeneration.java` | 新增真实角色加载入口，保留临时机器人入口 |
| `BotTypeManager.java` | 支持使用真实 Character 启动 BotSM |
| `BotHelpers.java` | 移除 ID 大小判断，使用显式注册表 |
| `CharacterStorage.java` | 区分临时机器人和持久化队友 |
| `Character.java` | 增加可持久化机器人状态或复用登录状态 |
| `CharacterAutosaverTask.java` | 确保持久化队友进入自动保存 |
| `World.java` | 关闭时先保存持久化队友 |
| `BotDecoratorSystem` | 持久化角色启动时跳过随机装饰 |
| GM 命令 | 支持真实 Character ID 查询 |

## 11. 验收标准

### AC-1 文件读取

- `account_use.txt` 中的 5 个账号可以被读取。
- 空行和注释不会导致失败。
- 无效账号会被记录并跳过。

### AC-2 角色加载

- 每个有效账号都能解析到唯一 Character。
- 加载后保留数据库中的 Character ID 和名字。
- 加载后保留等级、职业、属性和装备。
- 不会重新随机化外观。

### AC-3 在线运行

- 5 个 Character 能同时出现在服务端。
- 可以加入玩家 Party。
- 可以启动现有 BotSM。
- 玩家可以在游戏客户端看到这些角色。

### AC-4 保存

- 修改等级后重启，等级保持不变。
- 修改 AP 或 SP 后重启，数据保持不变。
- 移动或更换装备后重启，装备状态保持不变。
- 背包变化能够保存。
- 正常关闭服务端时全部角色完成保存。

### AC-5 重启恢复

- 重启后仍然加载同一组 Character ID。
- 不会重新生成随机名字和外观。
- 不会在数据库中创建第二条重复 Character。

### AC-6 冲突保护

- 已在线角色不会被重复加载。
- 非唯一账号角色不会被任意选择。
- 不属于指定 World 的角色会被拒绝。

## 12. 实现顺序

1. 定义 `account_use.txt` 格式并添加示例。
2. 增加持久化队友配置。
3. 实现账号到 Character 的数据库查询。
4. 实现持久化队友注册表。
5. 实现真实 Character 加载并跳过随机装饰。
6. 修改 `isBot` 和角色查询逻辑。
7. 接通过程中的保存条件。
8. 增加定期保存、修改后保存和关闭保存。
9. 接入 Server 启动和关闭流程。
10. 增加 GM 查询和调试日志。
11. 完成重启、装备、背包和交易验证。

## 13. 风险

### R1 共享 Client 上下文

当前所有临时机器人共享一个 BotClient。

持久化队友来自不同账号，未来可能需要：

- 独立 Client
- 独立账号上下文
- 独立 Channel
- 独立登录状态

MVP 固定同一 World 和 Channel，降低本轮复杂度。

### R2 机器人身份判断

`id > 20000` 的判断散落影响范围广。虽然调用入口集中在 `BotHelpers`，但修改后必须回归测试：

- 地图观察
- 交易
- 组队
- Buff
- 超时
- 自动保存
- GM 命令

### R3 保存并发

机器人行为和外部工具操作可能同时修改角色。

必须通过角色锁或串行操作队列保证：

```text
一次只执行一个背包或装备事务
保存前等待在途事务结束
```

### R4 意外修改真实账号

MVP 使用专用机器人账号，避免误加载玩家主账号。

启动时建议检查：

- 账号是否位于白名单
- Character 是否已经绑定为队友
- Character 是否允许由机器人控制

## 14. 完成定义

Commit 1 完成后，系统应达到：

```text
account_use.txt
    -> 5 个账号
    -> 5 个真实 Character
    -> 真实 ID 和真实数据
    -> 内存机器人控制器
    -> 正常运行
    -> 正常保存
    -> 重启恢复
```

这一阶段不要求完成队友战斗、Boss、副本或外部管理功能，只需要先建立可靠的角色身份和持久化基础。
