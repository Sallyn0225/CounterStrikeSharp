# CS2 1.41.9.7（build 14197，2026-11-11）网络消息句柄修复

这个分支（`fix-1.41.9.7`）基于官方 **v1.0.378**，只用来编译一个能在 CS2 1.41.9.7 上运行的 Windows 原生 `counterstrikesharp.dll`，等官方 CSS 发布新版之后就不用了。

## 问题

1.41.9.7 把 `INetworkMessageInternal` 从带虚表的抽象类改成了普通数据记录（40 字节的 `NetMessageInfo_t` 本身），同时 `INetworkMessages` 的虚表重排了（`FindOrCreateNetMessage` 挪到第 20 项，删掉了 `SerializeAbstract`、`AllocateAndCopyConstructNetMessageAbstract`、`DeallocateNetMessageAbstract`）。
v1.0.378 在 `PostEventAbstract` 钩子里调用 `pEvent->GetNetMessageInfo()`，会把消息名字符串当成虚表去跳转，地图一加载、发出第一条用户消息就崩溃（`counterstrikesharp.dll+0xbc7cb`）。`ClientPrint`（`PrintToChat` 等）、`ReplicateConVar` 和 `UserMessage` 相关的 native 都走同一条路。

## 改动

- `hl2sdk-cs2/public/networksystem/inetworkserializer.h`、`inetworkmessages.h`：移植自 [mrc4tt/hl2sdk@6c3f0eb](https://github.com/mrc4tt/hl2sdk/commit/6c3f0eb6aaf2d583be8d6b51a6abcd919d0253d0)。`inetworkserializer.h` 原样套用；`INetworkMessages` 那段因为基线不同（这里是官方 s2sdk `d4a8b46`），手工只做虚表的增删，其余保持原样。完整差异见 `hl2sdk-cs2-14197-netmessages.diff`。编译时工作流把 `patches/hl2sdk-cs2/` 整份覆盖进子模块。
- `src/core/UserMessage.*`、`src/scripting/natives/natives_usermessages.cpp`：移植自 [mrc4tt/CounterStrikeSharp@c9b8767](https://github.com/mrc4tt/CounterStrikeSharp/commit/c9b8767)。引擎现在把消息命名为 `"<类型> [<id>]"`，`GetMessageName` 去掉后缀，插件看到的名字和以前一样。

## 对着游戏核对过的内容（networksystem.dll / engine2.dll，1.41.9.7 Windows）

- `CNetworkMessages` 虚表（RTTI `.?AVCNetworkMessages@@`）36 项。第 8 项 `GetNetMessageInfo` 直接返回参数（为空时返回一个默认记录）；第 9 项 `FindNetworkMessage` 调用 `V_stricmp_fast`；第 10 项 `FindNetworkMessagePartial` 调用 `V_stristr_fast`；第 20 项 `FindOrCreateNetMessage` 用 `"%s [%d]"` 拼消息名。
- `CGameEventSystem` 虚表 21 项，数量没变。第 16 项（`PostEventAbstract` 的客户端掩码版本）的参数顺序和 SDK 一致，并且通过 `INetworkMessages` 第 8 项取消息信息。
- 崩溃转储里的 `pEvent`：`+0x00` 指向 `"CUserMessageAudioParameter [144]"`，`+0x18` 是 144。

## 实测（2026-11-12，Windows，CS2 1.41.9.7）

只把官方 1.0.378 安装里的 `bin/win64/counterstrikesharp.dll` 换成这个分支 `0722a59` 编出来的版本，C# API 和各插件都没动。`meta list` 显示 `CounterStrikeSharp (v1.0.378-netmsgfix @ 0722a59)`。

- de_dust2 和 bot 的本地对局跑了大约 10 分钟，没有崩溃（换之前一进地图就崩）。
- `PrintToChat` 正常（回合伤害统计插件、库存模拟器 `!ws` 的回复），`UserMessage.FromPartialName(...).Send()` 正常（NadeSystem 的 `CCSUsrMsg_RadioText`），聊天命令正常（`!scsp 13` 能回复并换上模型）。
- 同一天用户自己玩过，确认没问题。
- 后面的 `70d8814` 只改了代码格式，代码逻辑相同。

官方 CSS 发布支持 1.41.9.7 的版本后，就换回官方版，这个分支不再使用。
