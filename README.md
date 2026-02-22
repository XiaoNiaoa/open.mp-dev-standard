# open.mp/sa-mp 服务器开发规范
为了避免开发者在服务器开发过程中，系统功能逐渐坍塌。本规范的引入，并非为了增加开发负担，而是为了建立一套对抗系统熵增的秩序，共同维护一个默认的社区标准和规范。

本规范欢迎所有人提交 PR 或 Issue 进行改进。所有采纳的修改都将记入贡献者名单，共同维护 SA‑MP/open.mp 中文开发社区的良好生态。规范的生命力在于持续迭代，而非一成不变。

## 文件区分

其实文件夹怎么分没有铁打的标准，每个项目情况都不一样，但大致的思路都是差不多的

先按“谁先用谁、谁依赖谁”排顺序，再按“这是游戏里的哪个系统”来分组，没必要一开始就搞得很复杂，先写着写着自然就知道哪里该拆了

这里仅提供示范思路：
```text
/Server
  ├── gamemodes/
  │    └── main.pwn           // 主文件，负责 #include 模块
  │    └── locale/            // 如果你打算做多语言的话
  │    └── utils/             // 自定义封装好的通用函数、工具（和 libraries 差不多）
  │    └── core/              // 低层、必须先加载的核心系统（不依赖业务）
  │    │   ├── config/        // 配置相关 如 服务器名称 版本 规则 服务器设置
  │    │   ├── shared/        // 所有模块的“通用语言” 共享静态定义数据 用于校验 杜绝跨模块的变量污染
  │    │   └── database/      // 数据库(如：MySQL) 
  │    ├── world/             // 游戏世界环境相关（地图、动态对象、时间等等）
  │    └── modules/           // 所有功能模块存放处
  │         ├── players/      // 玩家模块
  │         ├── vehicles/     // 车辆模块
  │         ├── houses/       // 房屋模块
  │         └── admin/        // 管理员模块
  │    └── libraries/         // 第三方库 (如 mysql, sscanf等)
  │    └── filterscripts/     // 可动态加载的独立功能（脚本玩法、工具之类的附加项内容）
```

## 模块间通信准则

模块之间严禁直接操作对方的 static 变量，必须通过 API

如果 vehicles 模块需要获取玩家的某个状态，必须调用 Player_GetSomeStatus(playerid)，严禁直接读取 gPlayerData

## 命名规范

- 函数：PascalCase(所有单词首字母大写), 前缀是模块名 (如: Player_Load(playerid))。
- 全局变量：
```text
前缀 g_ 代表全局变量 (gobal) 比如: new g_Var;
前缀 gc_ 代表全局变量 (gobal const) 比如: new const gc_Var;
前缀 gs_ 代表全局静态变量（作用域仅在当前模块内） (gobal static) 比如: static gs_Var;
前缀 gsc_ 表示全局静态常数变量 (gobal static const) 比如: static const gsc_Var;
```
- 局部变量: 首单词小写, 后面单词首字母大写，或小写的缩写 (如: new number, vehicleIndex, id, pos)
- 常量/宏：大写 SNAKE_CASE (如: #define MAX_VEHICLES 2000`)。
- 文件：全小写，分割使用 '-' (如: player-main.inc、player-impl.inc)；主文件 main.pwn。
- 枚举与数据结构规范: 采用 E_MODULENAME_DATA 格式（如 E_PLAYER_DATA），全大写，采用 'E_' 开头
- 枚举成员采用: 模块名_字段名 全大写 SNAKE_CASE, 便于快速区分、选择、修改等等
```c
// 房屋模块
enum E_HOUSE_DATA
{
    HOUSE_DBID,
    HOUSE_OWNER[MAX_PLAYER_NAME],
    Float:HOUSE_ENTRANCE_X,
    Float:HOUSE_ENTRANCE_Y,
    Float:HOUSE_ENTRANCE_Z,
    bool:HOUSE_IS_LOCKED,
    HOUSE_PRICE
}
static gs_HouseData[MAX_HOUSES][E_HOUSE_DATA];

// 载具模块
enum E_VEHICLE_DATA
{
    VEHICLE_DBID,
    VEHICLE_MODEL,
    Float:VEHICLE_HEALTH,
    VEHICLE_OWNER_ID
}
static gs_VehicleData[MAX_VEHICLES][E_VEHICLE_DATA];
```

## 缩写准则

只允许通用缩写

如：ID (Identity), Pos (Position), Rot (Rotation), Max/Min, Msg (Message), Cmd (Command) 等等

错误示例：pLvl（到底是 Level 还是 Leave？）

禁止：将 Owner 缩写为 O, 将 Price 缩写为 Pr，禁止使用无意义缩写

## 包含守卫：

打开任何一个 .inc 文件，第一眼能看到“这个文件要放在哪里、依赖谁”，不用翻 main.pwn 的 include 列表猜顺序或者人工记忆顺序，长期维护更加直观清晰

尤其是相同层级的脚本，由于相互之间可能存在依赖关系，比如房屋模块需要使用到玩家模块的信息(金钱等等)，包含守卫可以很好地理清关系，避免脚本运行时出现问题，在编译阶段完美规避问题的出现

注意注释保持简短，别写成大段文档

```c++
// 防止重复包含 替换 SCRIPT_NAME 即可
#if defined _INC_SCRIPT_NAME
	#endinput
#endif
#define _INC_SCRIPT_NAME

// 模块之间的依赖说明
#if !defined _INC_OTHER_SCRIPT_NAME
	#error 需要包含 other-script.inc.
#endif

// 环境约束 确保编译环境正确
#if !defined _INC_open_mp
	#error 需要包含 open.mp.inc.
#endif

// 依赖项的显式校验 快速定位缺失依赖
#tryinclude <Pawn.RakNet>
#if !defined PAWNRAKNET_INC_
    #error 需要使用 Pawn.RakNet 插件
#endif

// 插件版本与功能对齐
#tryinclude <streamer>
#if !defined Streamer_IncludeFileVersion
    #error cannot read from file: "streamer.inc"
#elseif Streamer_IncludeFileVersion != 0x296
    #error 不兼容的 streamer 插件版本, 请使用 2.9.6 版本.
#endif

// 零开销的默认配置 可在后续代码中随时关闭或开启调试
#if !defined SCR_DEBUG
	#define SCR_DEBUG false
#endif

// 或者这样
#if SCR_DEBUG
    #define DebugMessage(%1) printf("[Script Debug] " %1)
    #define ErrorMessage(%1) printf("[Script Debug-Error] " %1)
#else
    #define DebugMessage(%1);
    #define ErrorMessage(%1);
#endif
```

## ALS 钩子标准

用传统 ALS 钩子扩展回调/函数

示例：回调钩子 (OnGameModeInit)
```c
// 替换 SCR 即可
public OnGameModeInit()
{
    #if defined SCR_OnGameModeInit
        return SCR_OnGameModeInit();
    #else
        return 1;
    #endif
}
#if defined _ALS_OnGameModeInit
    #undef OnGameModeInit
#else
    #define _ALS_OnGameModeInit
#endif
#define OnGameModeInit SCR_OnGameModeInit
#if defined SCR_OnGameModeInit
    forward SCR_OnGameModeInit();
#endif
```

示例：stock 函数钩子 (SetPlayerScore)
```c++
// 替换 SCR 即可
stock bool:SCR_SetPlayerScore(playerid, score)
{
    if(SetPlayerScore(playerid, score))
    {
		// ......
        return true;
    }
    return false;
}
#if defined _ALS_SetPlayerScore
    #undef SetPlayerScore
#else
    #define _ALS_SetPlayerScore
#endif
#define SetPlayerScore SCR_SetPlayerScore
```

## 函数作用域规范

1. static stock (模块私有函数)
   
- 作用域仅在当前 .inc 文件中, 它只在内部使用
- 命名以 下划线 开头，采用 _ModuleName_FunctionName 格式（如 _House_CheckDistance）

2. stock (模块公开接口)

- 定义：作用域全服务器, 命名采用 ModuleName_FunctionName 格式

```c
#define HOUSE_TAX_RATE 0.1

// 只有本模块能用，外部无法使用，也不会冲突
static stock _House_CalculateTax(houseid) 
{
    return floatround(float(gs_HouseData[houseid][HOUSE_PRICE]) * HOUSE_TAX_RATE);
}

// 对外接口
stock bool:House_GetTotalCost(houseid, &cost)
{
	if(houseid < 0 || houseid >= MAX_HOUSES)
		return false;

    // 内部逻辑调用私有函数
    cost = gs_HouseData[houseid][HOUSE_PRICE] + _House_CalculateTax(houseid);
	return true;
}
```

## 注释标准

对于函数的说明注释使用 [Doxygen](https://github.com/doxygen/doxygen) 风格，直接关系到长期维护，至少包含 功能简述、参数、返回值、注意事项

```c
/**
 * 获取玩家当前拥有的房产数量。
 * @param playerid 玩家ID
 * @return 房产数量，失败返回 -1
 */
stock Player_GetPropertyCount(playerid);
```

## 标签规范

用 enum + 宏定义标签，确保类型安全。

示例：WEATHER
```c
enum WEATHER:
{
    WEATHER_UNKNOWN = -1,
    WEATHER_EXTRASUNNY_LA = 0,
    WEATHER_SUNNY_LA,
    WEATHER_EXTRASUNNY_SMOG_LA,
    WEATHER_SUNNY_SMOG_LA,
    WEATHER_CLOUDY_LA,
    WEATHER_SUNNY_SF,
	...,
	...
}
```

## 错误处理标准

规范：返回值语义化

逻辑判断返回类型 bool:

成功返回 ID ( >= 0)，失败返回 INVALID_..._ID (-1)


## 开源协议 (License)

本项目遵循 [MIT License](LICENSE) 协议。
你可以自由地使用、修改和分发本规范。
