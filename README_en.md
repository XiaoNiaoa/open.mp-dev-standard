# open.mp/sa-mp Server Development Specification

To prevent the gradual decay of system functionality during server development, this specification is introduced—not to increase the development burden—but to establish a framework of order that counteracts systemic entropy. Its goal is to jointly maintain a default community standard and set of norms.

Everyone is welcome to submit PRs or Issues to improve this specification. All accepted modifications will be recorded in the contributors list, helping to sustain a healthy ecosystem within the SA‑MP/open.mp Chinese development community. The vitality of a specification lies in continuous iteration, not permanence.

## File Organization

There is no rigid standard for how folders should be structured; every project has its own needs. However, the general ideas are similar.

Order items by “who uses whom, who depends on whom”, then group them by “which game system this belongs to”. There is no need to overcomplicate it from the start—as you write, the right places to split will naturally become clear.

Here is only a sample approach:
```text
/Server
  ├── gamemodes/
  │    └── main.pwn           // Main file, responsible for #include of modules
  │    └── locale/            // If you plan to support multiple languages
  │    └── utils/             // Custom generic functions and utilities (similar to libraries)
  │    └── core/              // Low-level, must-be-loaded-first core systems (no business logic dependencies)
  │    │   ├── config/        // Configuration related, e.g., server name, version, rules, server settings
  │    │   ├── shared/        // “Common language” for all modules; shared static definitions for validation; prevents cross‑module variable pollution
  │    │   └── database/      // Database (e.g., MySQL)
  │    ├── world/             // Game world environment (maps, dynamic objects, time, etc.)
  │    └── modules/           // All feature modules reside here
  │         ├── players/      // Player module
  │         ├── vehicles/     // Vehicle module
  │         ├── houses/       // House module
  │         └── admin/        // Admin module
  │    └── libraries/         // Third-party libraries (e.g., mysql, sscanf, etc.)
  │    └── filterscripts/     // Independently loadable features (additional content like scripts, tools, etc.)
```

## Inter‑Module Communication Guidelines

Direct manipulation of another module’s `static` variables is strictly prohibited; communication must occur via an API.

If the vehicles module needs to obtain a player’s state, it **must** call `Player_GetSomeStatus(playerid)`—direct access to `gPlayerData` is forbidden.

## Naming Conventions

- **Functions**: `PascalCase` (first letter of each word capitalized), prefixed with the module name (e.g., `Player_Load(playerid)`).
- **Global variables**: Lowercase `g` prefix (e.g., `gPlayerData[MAX_PLAYERS]`).
- **Local variables**: First word lowercase, subsequent words capitalized, or lowercase abbreviations (e.g., `new number`, `vehicleIndex`, `id`, `pos`).
- **Constants / Macros**: Uppercase `SNAKE_CASE` (e.g., `#define MAX_VEHICLES 2000`).
- **Files**: All lowercase, hyphen-separated (e.g., `player-main.inc`, `player-impl.inc`); main file: `main.pwn`.
- **Enumerations & Data Structures**: Format `E_MODULENAME_DATA` (e.g., `E_PLAYER_DATA`), all uppercase, start with `E_`.
- **Enum members**: Format `MODULE_FIELD_NAME` (uppercase `SNAKE_CASE`) to easily distinguish, select, modify, etc.
```c
// House module
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
static gHouseData[MAX_HOUSES][E_HOUSE_DATA];

// Vehicle module
enum E_VEHICLE_DATA
{
    VEHICLE_DBID,
    VEHICLE_MODEL,
    Float:VEHICLE_HEALTH,
    VEHICLE_OWNER_ID
}
static gVehicleData[MAX_VEHICLES][E_VEHICLE_DATA];
```

## Abbreviation Guidelines

Only universally accepted abbreviations are permitted.

Examples: ID (Identity), Pos (Position), Rot (Rotation), Max/Min, Msg (Message), Cmd (Command), etc.

**Incorrect example**: `pLvl` (is it Level or Leave?)

**Forbidden**: Abbreviating `Owner` to `O`, `Price` to `Pr`. Meaningless abbreviations are prohibited.

## Include Guards

When opening any `.inc` file, one should be able to see at a glance “where this file belongs and what it depends on”—without having to guess the order by scanning `main.pwn`’s include list or relying on memory. This makes long‑term maintenance much more intuitive.

This is especially important for scripts at the same level that may have mutual dependencies (e.g., the house module needs player information like money). Include guards help clarify these relationships, preventing runtime issues and neatly avoiding problems at compile time.

Keep comments brief; do not turn them into lengthy documentation.

```c++
// Prevent duplicate inclusion – replace SCRIPT_NAME accordingly
#if defined _INC_SCRIPT_NAME
	#endinput
#endif
#define _INC_SCRIPT_NAME

// Module dependency description
#if !defined _INC_OTHER_SCRIPT_NAME
	#error Need to include other-script.inc.
#endif

// Environment constraints – ensure correct compilation environment
#if !defined _INC_open_mp
	#error Need to include open.mp.inc.
#endif

// Explicit dependency verification – quickly locate missing dependencies
#tryinclude <Pawn.RakNet>
#if !defined PAWNRAKNET_INC_
    #error The Pawn.RakNet plugin is required
#endif

// Plugin version and feature alignment
#tryinclude <streamer>
#if !defined Streamer_IncludeFileVersion
    #error cannot read from file: "streamer.inc"
#elseif Streamer_IncludeFileVersion != 0x296
    #error Incompatible streamer plugin version; please use version 2.9.6.
#endif

// Zero‑overhead default configuration – debugging can be toggled off or on in subsequent code
#if !defined SCR_DEBUG
	#define SCR_DEBUG false
#endif
```

## ALS Hook Standard

Use the traditional ALS hook method to extend callbacks/functions.

**Example – Callback hook (OnGameModeInit)**:
```c
// Replace SCR accordingly
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

**Example – Stock function hook (SetPlayerScore)**:
```c++
// Replace SCR accordingly
stock bool:SCR_SetPlayerScore(playerid, score)
{
    if(SetPlayerScore(playerid, score))
    {
		// ...
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

## Function Scope Specification

1. **static stock** (module‑private functions)
   
   - Scope is limited to the current `.inc` file; for internal use only.
   - Name begins with an underscore and follows the format `_ModuleName_FunctionName` (e.g., `_House_CheckDistance`).

2. **stock** (public module interface)

   - Definition: Server‑wide scope, naming follows `ModuleName_FunctionName` format.

```c
#define HOUSE_TAX_RATE 0.1

// Only usable within this module; external code cannot access it, no conflicts
static stock _House_CalculateTax(houseid) 
{
    return floatround(float(gHouseData[houseid][HOUSE_PRICE]) * HOUSE_TAX_RATE);
}

// External interface
stock bool:House_GetTotalCost(houseid, &cost)
{
	if(houseid < 0 || houseid >= MAX_HOUSES)
		return false;

    // Internal logic calls private function
    cost = gHouseData[houseid][HOUSE_PRICE] + _House_CalculateTax(houseid);
	return true;
}
```

## Comment Standard

For function documentation comments, use the [Doxygen](https://github.com/doxygen/doxygen) style. This is essential for long‑term maintenance and should include at least: a brief description, parameters, return value, and any important notes.

```c
/**
 * Retrieves the number of properties currently owned by a player.
 * @param playerid The player's ID
 * @return The number of properties, or -1 on failure
 */
stock Player_GetPropertyCount(playerid);
```

## Tag Specification

Use `enum` + macro‑defined tags to ensure type safety.

**Example – SELECT_OBJECT**:
```c
#define SELECT_OBJECT: __TAG(SELECT_OBJECT):
enum SELECT_OBJECT:__SELECT_OBJECT
{
    UNKNOWN_SELECT_OBJECT = -1,
    SELECT_OBJECT_GLOBAL_OBJECT = 1,
    SELECT_OBJECT_PLAYER_OBJECT
}
static stock SELECT_OBJECT:_@SELECT_OBJECT() { return __SELECT_OBJECT; }

#define UNKNOWN_SELECT_OBJECT (SELECT_OBJECT:-1)
#define SELECT_OBJECT_GLOBAL_OBJECT (SELECT_OBJECT:1)
#define SELECT_OBJECT_PLAYER_OBJECT (SELECT_OBJECT:2)
```
Usage: 
```c
// Compiler will issue a warning because it is not tagged SELECT_OBJECT
SetSomething(1); 

// Compiler accepts
SetSomething(SELECT_OBJECT_GLOBAL_OBJECT);
```

## Error Handling Standard

**Rule**: Semantic return values

- Logical checks: Return `bool`.
- Success: Return an ID (`>= 0`), failure: return `INVALID_..._ID` (`-1`).

## License

This project follows the [MIT License](LICENSE).  
You are free to use, modify, and distribute this specification.
