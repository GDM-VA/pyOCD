---
title: User scripts
---

## Introduction

pyOCD has support for customization through what are called user scripts. These are Python scripts
that are loaded by pyOCD before it connects to the target. A user script can define functions that
are called as hooks at certain points in the connection lifetime, and can extend, modify, or
completely override the default behaviour.

If a file named `pyocd_user.py` or `.pyocd_user.py` is placed in the project directory, pyOCD will
automatically detect it and load it as a user script. If you prefer another name, you can set the
`user_script` option. Another alternative is to provide the filename using the `--script` command
line argument. If a relative path is set either with the option or command line, it will be searched
for in the project directory.


## Examples

This example user script shows how to add a new memory region.

```py
# This example applies to the Nordic nRF52 devices.

def will_connect(board):
    # Create the new ROM region for the FICR.
    ficr = RomRegion(
                name="ficr",
                start=0x10000000,
                length=0x460
                )

    # Add the FICR region to the memory map.
    target.memory_map.add_region(ficr)
```

This example shows how to override the flash algorithm for an external flash memory.

```py
# This example applies to the NXP i.MX RT10x0 devices.

# Unlike the previous example, the board argument is excluded here.
def will_connect():
    # Look up the external flash memory region.
    extFlash = target.memory_map.get_first_matching_region(name="flexspi")

    # Set the path to an .FLM flash algorithm.
    extFlash.flm = "MIMXRT105x_QuadSPI_4KB_SEC.FLM"
```

This example demonstrates setting the DBGMCU_CR register after connecting for STM32 devices.

```py
# This example applies to the ST STM32L0x1 devicess.

DBG_CR = 0x40015804

def did_connect():
    # Set STANDBY, STOP, and SLEEP bits all to 1.
    target.write32(DBG_CR, 0x7)
```

Another common use for a script is to initialize external memory such as SDRAM.


## User-defined commands

New commands accessible from the commander subcommand or gdbserver monitor commands can be easily created in a user
script.

User defined commands are created by using the `@command()` decorator on a function. The name of the new command can
either be the same as the name of the decorated function, or can be set explicitly with a `name` (or first positional)
argument to the decorator. For instance, either `@command('mycmd')` or `@command(name='anothercmd')`. Note that the
decorator requires parentheses (it must be called as a function) even if there are no parameters.

Parameters for the new command are automatically determined using introspection and type annotations. Arguments for
parameters of these types are converted to the appropriate type before the function is called as a command.

Supported parameter types:
- `int`
- `float`
- `str`
- Variable arguments, e.g. `*args`.

Keyword parameters are not allowed.

An `int` parameter is converted using the same method as for other pyOCD commands. Hexadecimal and binary numbers are
allowed, digits can be separated by underscores, and so on. For variable arguments, type annotations are ignored and the
tuple passed to the function will contain strings as entered in the command invocation.

The decorated function remains accessible as a regular function in the user script namespace, and is therefore callable
from other functions within the user script. This is true even if the function definition is not compatible with the
command decorator, for instance if it has invalid parameter types.

Help for the new command can be specified by passing a `help` argument to the `@command` decorator.

Example:

```py
@command(help="Decode and print the first few vectors")
def vectable(base: int):
    vecs = target.read_memory_block32(base, 4)
    print(f"Initial SP:     {vecs[0]:#010x}")
    print(f"ResetHandler:   {vecs[1]:#010x}")
    print(f"NMI:            {vecs[2]:#010x}")
    print(f"HardFault:      {vecs[3]:#010x}")
```


## Script globals

A number of useful symbols are made available in the global namespace of user scripts. These include
both target related objects, as well as parts of the pyOCD Python API.

| Symbol | Description |
|--------|-------------|
| `__file__` | Path to the user script. |
| `__name__` | Module name of the user script. |
| `aps` | Dictionary of CoreSight Access Port (AP) objects. The keys are the APSEL value. |
| `board` | The `Board` object. |
| `BreakpointType` | Enumeration of breakpoint types. |
| `command` | Decorator for defining new commands. See [user-defined commands](#user_defined_commands) for details. |
| `DeviceRegion` | Device-type memory region class. |
| `debug` | Log a debug message. |
| `dp` | The CoreSight Debug Port (DP) object. |
| `Error` | The base class for all pyOCD exceptions. |
| `Event` | Enumeration of notification event types. |
| `exceptions` | Module containing the exception classes. |
| `error` | Output an error log. |
| `info` | Output an info-level log message. |
| `FileProgrammer` | Utility class to program files to target flash. |
| `FlashEraser` | Utility class to erase target flash. |
| `FlashLoader` | Utility class to program raw binary data to target flash. |
| `FlashRegion` | Flash memory region. |
| `HaltReason` | Enumeration of halt reasons. |
| `LOG` | `Logger` object for the user script. |
| `MemoryMap` | Class representing the device's memory map. |
| `MemoryType` | Memory region type enumeration. |
| `options` | The session options dictionary. |
| `probe` | The connected debug probe object. |
| `pyocd` | The root pyOCD package. |
| `RamRegion` | RAM memory region. |
| `ResetType` | Reset type enumeration. |
| `RomRegion` | ROM memory region. |
| `RunType` | Enumeration of types of run operations (step or run). |
| `SecurityState` | Enumeration of core security states. |
| `session` | The session object, which is the root of the connection object graph. |
| `State` | Enumeration of target state. |
| `Target` | Base class, mostly useful for numerous constants that are defined within the class. |
| `target` | The `CoreSightTarget` or subclass instance representing the MCU. |
| `TransferError` | Exception class for all transfer errors. |
| `TransferFaultError` | Exception subclass of `TransferError` for bus faults. |
| `VectorCatch` | Namespace class containing bit mask constants for vector catch options. |
| `warning` | Log a warning. |
| `WatchpointType` | Enumeration of watchpoint types. |


## Delegate functions

This section documents all functions that user scripts can provide to modify pyOCD's behaviour. Some are simply
notifications, while others allow for overriding of default behaviour. Collectively, these are called delegate functions.

All parameters of script delegate functions are optional. Parameters can be declared in any order, and that are not
needed can be excluded. In fact, most parameters are not necessary because the same objects are available as script
globals, for instance `session` and `target`.

Those parameters that are present must have names matching the specification below, and there must not be unspecified,
required parameters (those without a default value). (Extra optional parameters are allowed but will never be passed any
value other than the default, unless you call the function yourself from within the script.)


### will_connect

Pre-init notification for the board.
```
will_connect(board: Board) -> None
```

**Parameters** \
*board* - A `Board` instance that is about to be initialized. \
**Result** \
Ignored.

### did_connect

Post-initialization notification for the board.
```
did_connect(board: Board) -> None
```

**Parameters** \
*board* - A `Board` instance. \
**Result** \
Ignored.

### will_init_target

Hook to review and modify init call sequence prior to execution.
```
will_init_target(target: SoCTarget, init_sequence: CallSequence) -> None
```

**Parameters** \
*target* - An `SoCTarget` object about to be initialized. \
*init_sequence* - The `CallSequence` that will be invoked. Because call sequences are
  mutable, this parameter can be modified before return to change the init calls. \
**Result** \
Ignored.

### did_init_target

Post-initialization notification.
```
did_init_target(target: SoCTarget) -> None
```

**Parameters** \
*target* - An `SoCTarget` object. \
**Result** \
Ignored.

### will_start_debug_core

Hook to enable debug for the given core.
```
will_start_debug_core(core: CoreTarget) -> Optional[bool]
```

**Parameters** \
*core* - A `CoreTarget` object about to be initialized. \
**Result** \
*True* Do not perform the normal procedure to start core debug. \
*False/None* Continue with normal behaviour.

### did_start_debug_core

Post-initialization hook.
```
did_start_debug_core(core: CoreTarget) -> None
```

**Parameters** \
*core* - A `CoreTarget` object. \
**Result** \
Ignored.

### will_stop_debug_core

Pre-cleanup hook for the core.
```
will_stop_debug_core(core: CoreTarget) -> Optional[bool]
```

**Parameters** \
*core* - A `CoreTarget` object. \
**Result** \
*True* Do not perform the normal procedure to disable core debug. \
*False/None* Continue with normal behaviour.

### did_stop_debug_core

Post-cleanup notification for the core.
```
did_stop_debug_core(core: CoreTarget) -> None
```

**Parameters** \
*core* - A `CoreTarget` object. \
**Result** \
Ignored.

### will_disconnect

Pre-disconnect notification.
```
will_disconnect(target: SoCTarget, resume: bool) -> None
```

**Parameters** \
*target* - An `SoCTarget` object. \
*resume* - The value of the `disconnect_on_resume` option. \
**Result** \
Ignored.

### did_disconnect

Post-disconnect notification.
```
did_disconnect(target: SoCTarget, resume: bool) -> None
```

**Parameters** \
*target* - An `SoCTarget` object. \
*resume* - The value of the `disconnect_on_resume` option. \
**Result** \
Ignored.

### will_reset

```
will_reset(core: CoreTarget, reset_type: Target.ResetType) -> Optional[bool]
```
Pre-reset hook.

**Parameters** \
*core* - A `CoreTarget` instance. \
*reset_type* - One of the `Target.ResetType` enumerations. \
**Result** \
*True* The hook performed the reset.  \
*False/None* Caller should perform the normal reset procedure.

### did_reset

Post-reset notification.
```
did_reset(core: CoreTarget, reset_type: Target.ResetType) -> None
```

**Parameters** \
*core* - A `CoreTarget` instance. \
*reset_type* - One of the `Target.ResetType` enumerations. \
**Result** \
Ignored.

### set_reset_catch

Hook to prepare target for halting on reset.
```
set_reset_catch(core: CoreTarget, reset_type: Target.ResetType) -> Optional[bool]
```

**Parameters** \
*core* - A `CoreTarget` instance. \
*reset_type* - One of the `Target.ResetType` enumerations. \
**Result** \
*True* This hook handled setting up reset catch, caller should do nothing. \
*False/None* Perform the default reset catch set using vector catch.

### clear_reset_catch

Hook to clean up target after a reset and halt.
```
clear_reset_catch(core: CoreTarget, reset_type: Target.ResetType) -> None
```

**Parameters** \
*core* - A `CoreTarget` instance. \
*reset_type* - One of the `Target.ResetType` enumerations. \
**Result** \
Ignored.

### mass_erase

Hook to override mass erase.
```
mass_erase(target: SoCTarget) -> Optional[bool]
```

**Parameters** \
*target* - An `SoCTarget` object. \
**Result** \
*True* Indicate that mass erase was performed by the hook. \
*False/None* Mass erase was not overridden and the caller should proceed with the standard mass erase procedure.

### trace_start

Notification to prepare for tracing the target.
```
trace_start(target: SoCTarget, mode: int) -> None
```

**Parameters** \
*target* - A `CoreSightTarget` object. \
*mode* - The trace mode. Currently always 0 to indicate SWO. \
*Result* - Ignored.

### trace_stop

Notification to clean up after tracing the target.
```
trace_stop(target: SoCTarget, mode: int) -> None
```

**Parameters** \
*target* - A `CoreSightTarget` object. \
*mode* - The trace mode. Currently always 0 to indicate SWO. \
**Result** \
Ignored.
