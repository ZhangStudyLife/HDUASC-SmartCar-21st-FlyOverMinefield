---
name: expose-air-float-param
description: Connect an existing float global in the HDU SmartCar CYT4BB7_Air firmware to the AirComm parameter table and the CYT4bb7_Car screen menu so it can be displayed and adjusted from the car. Use when a user asks to expose, register, display, synchronize, or tune an existing Air-side float through the air-ground serial link and car menu. Do not use for new variables, integer parameters, run-data telemetry, or image-core remote parameters.
---

# Expose Air Float Parameter

Expose one existing Air-side `float` global through the current named-parameter
path. Keep AirComm and the Car menu as one ordered contract.

## Required inputs

Obtain these values before editing:

- Air global variable name
- Car menu default value
- Edit step
- Minimum and maximum values
- Existing Car menu group name

If any value is ambiguous, ask before editing. Do not invent tuning limits or a
menu group.

## Preconditions

1. Work from the repository root containing `CYT4BB7_Air` and `CYT4bb7_Car`.
2. Read both subtree `AGENTS.md` files.
3. Inspect both worktrees and preserve unrelated changes.
4. Run the bundled check before editing:

```powershell
powershell -ExecutionPolicy Bypass -File ".agents\skills\expose-air-float-param\scripts\check_air_float_param.ps1" -Name <variable>
```

The pre-edit check may report that the variable is not registered yet, but the
Air and Car parameter catalogs must already have matching names, order, actual
counts, and count constants.

## Validate variable ownership

Find exactly one non-`extern` Air `.c` definition whose declared type is a
writable `float` or `volatile float`. Reject `const`, integer types, macros,
structure members, local variables, and multiple definitions.

Find the owning module's `.h` declaration. If the definition exists but the
public declaration is missing, add `extern float <name>;` to that owning header
only when AirComm needs it. Do not place a duplicate storage definition or an
unowned `extern` in the protocol module. Confirm the existing include path makes
the declaration visible to `air_comm_air.c`.

If the variable does not exist, stop. Identify the business module that owns the
behavior from the requested use and tell the user to define the variable in that
module's `.c` file and declare it in the matching `.h` file. Do not create it.

## Patch the Air side

Edit only
`CYT4BB7_Air/project/code/Protocols/AirComm/air_comm_air.c` plus the owning
header when a missing declaration requires it.

1. Add the parameter at the appropriate logical position in
   `air_comm_air_init()`:

```c
AIR_COMM_REGISTER_FLOAT(parameter_name, parameter_name, min_value, max_value);
```

2. Increase `AIR_COMM_PARAM_TABLE_MAX` by one.
3. Increase `AIR_COMM_DEFAULT_PARAM_COUNT` by one.
4. Keep the parameter name identical to the C global identifier.
5. Do not change the variable's value or business logic.

Reject the change if the name is already registered or the new count would
violate an existing capacity check.

## Patch the Car side

Edit only:

- `CYT4bb7_Car/project/code/Menu/menu_air_support.c`
- `CYT4bb7_Car/project/code/Menu/menu_air_support.h`

Add the entry to `s_air_param_definitions[]` at the same relative position as
the Air registration:

```c
MENU_AIR_PARAM(parameter_name, default_value, step_value,
               min_value, max_value, "Existing Group")
```

Preserve the array's comma style. Increase `MENU_AIR_MAX_PARAMS` and
`MENU_AIR_EXPECTED_PARAM_COUNT` by one.

Verify the group already exists in `air_param_menu[]` in `menu_config.c` and has
room under `MENU_MAX_ITEMS`. Do not add a new group unless the user explicitly
requests it.

Do not add parameter-specific rendering. The current menu builder reads
`s_air_param_definitions[]`, creates visible screen entries, edits the cached
value, and commits it to AirComm through the generic menu path.

## Verify

Run the bundled check again. It must report:

- one Air definition and at least one owning-header declaration
- one AirComm registration
- one Car menu entry
- identical ordered Air and Car parameter names
- identical actual counts
- matching Air count constants
- matching Car count constants
- final `PASS`

Also run scoped `git diff --check` in both subtrees and re-read the edited
regions. Build the Car CM7_0 target with:

```powershell
& "C:\Program Files\IAR Systems\Embedded Workbench 9.2\common\bin\iarbuild.exe" `
  "project\iar\project_config\cyt4bb7_cm_7_0.ewp" -build Debug
```

Do not run an Air command-line IAR build; the Air subtree rules prohibit it.
Report that Air validation was structural only. Do not commit, create branches,
or alter unrelated files unless the user explicitly requests it.

## Stop conditions

Stop without source edits when:

- the variable is missing, not `float`, local, or multiply defined
- ownership or header visibility cannot be established
- the name already exists on either side
- the requested menu group is missing or full
- Air and Car catalogs are inconsistent before editing
- count or storage capacity cannot accommodate the parameter
