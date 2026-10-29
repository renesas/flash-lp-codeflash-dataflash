# Flash LP Software Building Block

## Overview

This Software Building Block (SBB) demonstrates basic use of the Renesas Flash LP (`r_flash_lp`) driver for code flash and data flash operations. The reference application initializes Flash LP and lets the user select flash blocks from a serial-terminal menu, then demonstrates blank check, erase, write, read/verify, and code-flash access-window operations.

This SBB supports these boards:

- FPB-RA0L1
- FPB-RA2E3

## What use case does this SBB demonstrate?

The SBB demonstrates a basic nonvolatile-memory workflow using Flash LP:

1. Open the Flash LP driver.
2. Select data flash or code flash from the demo terminal.
3. Select a valid flash block.
4. Blank-check the selected area.
5. Erase the block when it is not blank.
6. Blank-check it again after erase.
7. Read and display the selected flash contents.
8. Write a known 64-byte pattern.
9. Read the data back with a memory copy.
10. Compare the read data with the written data.
11. For code flash, additionally demonstrate setting and clearing an access window.

The demo builds its write buffer using byte values `0x00` through `0x3F`, which makes the read-back data easy to inspect in the terminal.

## Hardware requirements

The supplied documentation requires one supported Renesas RA board and an appropriate USB cable for programming and debugging.

### FPB-RA0L1

Place jumper **J9 on pins 1-2** to use the on-board debug functionality.

### FPB-RA2E3

Close **E25, E28, E46, and E47** to use J-Link On-Board VCOM for the serial terminal.

## What does the SBB core code do?

The reusable SBB layer is implemented by:

- `flash_lp_ep.c`
- `flash_lp_ep.h`

It wraps the Flash LP driver and provides the operations needed by the reference application:

- Open and close the Flash LP instance.
- Blank-check an address range.
- Erase one flash block.
- Write 64 bytes to a flash address.
- Set a code-flash access window.
- Clear a code-flash access window.
- Handle Flash LP callback events used by background operation (BGO).
- Expose the most recent blank-check result.
- Define flash block sizes, address calculations, and board-dependent ranges used by the example.

The implementation uses callback flags for `FLASH_EVENT_NOT_BLANK`, `FLASH_EVENT_BLANK`, `FLASH_EVENT_ERASE_COMPLETE`, and `FLASH_EVENT_WRITE_COMPLETE`. When data-flash BGO is configured, the SBB waits for the corresponding callback state before returning from relevant operations.

## Demo/reference application versus reusable SBB code

### Reusable SBB code

`flash_lp_ep.c` and `flash_lp_ep.h` are the reusable Flash LP layer. They contain the Flash LP API wrappers, BGO callback handling, data/code flash address calculations, block-size definitions, and access-window wrappers.

### Demo/reference application code

`flash_lp_example_app.c` and `flash_lp_example_app.h` implement the interactive demonstration. They are responsible for:

- Initializing the UART console.
- Displaying SBB/FSP information.
- Presenting data-flash and code-flash menu choices.
- Accepting and validating block numbers.
- Building the demonstration write pattern.
- Sequencing blank check, erase, write, read, and compare operations.
- Printing data in hexadecimal.
- Disabling interrupts during code-flash program/erase operations in the demonstrated flow.
- Demonstrating code-flash access-window set/erase/write/clear behavior.
- Error reporting and breakpoint handling.

The application's entry point is:

```c
void flash_lp_example_app_run(void);
```

The source identifies this as the single function called from `hal_entry.c`.

## Which public functions should I look at first?

### `flash_lp_init()`

```c
fsp_err_t flash_lp_init(void);
```

Opens the generated Flash LP instance using `g_flash_ctrl` and `g_flash_cfg`.

### `flash_lp_blank_check()`

```c
fsp_err_t flash_lp_blank_check(uint32_t block_addr);
```

Checks `BLOCK_SIZE` bytes beginning at `block_addr`. The result is exposed through `g_blank_check_result`. If a BGO blank check is active, the wrapper waits for the blank/not-blank callback event.

### `flash_lp_erase()`

```c
fsp_err_t flash_lp_erase(uint32_t block_addr);
```

Erases `NUM_BLOCKS`, which is configured as one block in this SBB. For data flash with BGO enabled, the wrapper waits for `FLASH_EVENT_ERASE_COMPLETE`.

### `flash_lp_write()`

```c
fsp_err_t flash_lp_write(uint8_t * src_address,
                         uint32_t flash_address);
```

Writes `BLOCK_SIZE` bytes from the source buffer to the selected flash address. For data flash with BGO enabled, the function waits for the write-complete callback.

### `flash_lp_access_window_set()`

```c
fsp_err_t flash_lp_access_window_set(uint32_t block_addr);
```

Sets a Flash LP code-flash access window using addresses calculated by `FLASH_CF_START_ADDR()` and `FLASH_CF_END_ADDR()`.

### `flash_lp_access_window_clear()`

```c
fsp_err_t flash_lp_access_window_clear(void);
```

Clears the configured code-flash access window.

### `flash_lp_deinit()`

```c
void flash_lp_deinit(void);
```

Closes the Flash LP instance when it is open.

### `bgo_callback()`

```c
void bgo_callback(flash_callback_args_t * p_args);
```

Converts Flash LP callback events into internal flags consumed by the blank-check, erase, and write wrappers.

## What configuration assumptions were made for the reference boards?

### General operation size

The SBB defines:

```c
#define BLOCK_SIZE  (64U)
#define NUM_BLOCKS  (1U)
```

The demonstrated blank-check and write operations therefore use 64 bytes, while erase requests one configured flash block.

### FPB-RA0L1

For FPB-RA0L1, the supplied code defines:

- Data flash start: `0x40100000`
- Data flash end: `0x401003FF`
- Data flash block size used by the address calculator: 256 bytes
- Demo data-flash block range: 0 through 3
- Code flash block size: 2048 bytes
- Code flash block 10: `0x00005000`
- Code flash end: `0x0000FFFF` (block 31)
- Demo code-flash block range: 10 through 31
- Access-window block range: 10 through 30

### FPB-RA2E3

For FPB-RA2E3, the supplied code defines:

- Data flash start: `0x40100000`
- Data flash end: `0x401007FF`
- Data flash block size used by the address calculator: 1024 bytes
- Demo data-flash block range: 0 through 1
- Code flash block size: 2048 bytes
- Code flash block 10: `0x00005000`
- Code flash end: `0x0000FFFF` (block 31)
- Demo code-flash block range: 10 through 31
- Access-window block range: 10 through 30

### Data-flash BGO

The SBB checks `g_flash_cfg.data_flash_bgo`. When BGO is enabled for data flash, blank check, erase, and write completion depend on the callback configured for `g_flash_cfg` and handled by `bgo_callback()`.

### Input assumptions

The application's input buffer is 16 bytes. The README specifies that user input should not exceed **15 bytes** and states that operation is not guaranteed for non-integer or incorrectly formatted input.

## Important code-flash access-window warning

The supplied README warns that using a block number below 10 for code-flash operations can lock the board because of `R_FLASH_LP_AccessWindowSet()`.

The application itself validates normal code-flash and access-window selections starting at block 10.

The source also warns that a wrongly configured access window persists across a power cycle and can prevent programming of code flash. Its demonstrated access-window sequence clears the window after the erase/write operations.

This is a destructive flash demonstration. Do not repurpose the example's address ranges for application firmware or persistent product data without first validating the target memory map and protecting any flash that must be retained.

## What must I change for my board or application?

### Flash memory map

The address constants and block calculations in `flash_lp_ep.h` are device dependent. For a different MCU, validate:

- Data-flash start and end addresses.
- Data-flash erase block size.
- Code-flash block size.
- Code-flash start/end addresses used by the demo.
- Permitted block-selection ranges.
- Access-window range.

Do not rely solely on the header's generic `else` branch for a board that the supplied README does not identify as supported.

### Generated Flash LP configuration

The SBB expects generated objects named:

```text
g_flash_ctrl
g_flash_cfg
```

If data-flash BGO is enabled, the generated configuration must provide callback behavior compatible with `bgo_callback()`.

### Flash contents and reserved regions

The demo erases and writes the selected block. An integrating application must choose addresses that do not destroy firmware, configuration, boot information, calibration data, or other content that must be retained.

### Code-flash execution considerations

The reference application disables interrupts before performing its code-flash operation and re-enables them later. The source explains that this is done to prevent vector-table access while code flash is in program/erase mode.

If the code-flash flow is changed, the execution and interrupt arrangement must continue to satisfy the target Flash LP requirements.

### Write size

The SBB's current wrapper writes exactly 64 bytes (`BLOCK_SIZE`). Change the wrapper and caller consistently if the product requires a different write amount, while respecting the target Flash LP programming requirements.

### Clocks and timing

The supplied hand-written files do not specify flash clock configuration or measured erase/write timing. Any target-specific clock requirements must be established from the generated FSP configuration and the target Flash LP requirements.

### Memory

The reference application allocates two local 64-byte buffers for its flash operation: one read buffer and one write buffer. The supplied files do not provide the complete RAM/ROM footprint of the SBB or generated FSP driver.

### Task priorities and execution model

The supplied application does not define an RTOS or task priorities. BGO completion is handled by callback flags, but the wrappers wait in polling loops for those flags. An RTOS-based integration must define its own scheduling and synchronization approach.

### Console middleware

The terminal UI uses `console/uart_console.h` and standard I/O. These are demo dependencies. The reusable Flash LP wrappers can be called by another application control layer without retaining the interactive menu.

## Dependencies

### Renesas FSP Flash LP

The reusable SBB uses:

- `R_FLASH_LP_Open()`
- `R_FLASH_LP_BlankCheck()`
- `R_FLASH_LP_Erase()`
- `R_FLASH_LP_Write()`
- `R_FLASH_LP_AccessWindowSet()`
- `R_FLASH_LP_AccessWindowClear()`
- `R_FLASH_LP_Close()`

### Generated FSP configuration

The code includes `hal_data.h` and uses `g_flash_ctrl` and `g_flash_cfg`.

### Callback events

BGO handling uses:

- `FLASH_EVENT_NOT_BLANK`
- `FLASH_EVENT_BLANK`
- `FLASH_EVENT_ERASE_COMPLETE`
- `FLASH_EVENT_WRITE_COMPLETE`

### Console/reference application

The reference application uses:

- `console/uart_console.h`
- `uart_console_init()`
- `uart_console_has_char()`
- `uart_console_flush()`
- `printf()`
- `fgets()`
- `atoi()`
- `memcpy()`
- `memcmp()`
- `memset()`
- `strcspn()`
- `R_FSP_VersionGet()`

It also uses `__disable_irq()`, `__enable_irq()`, and `__BKPT()` in its code-flash/error-handling flows.

## Serial terminal configuration

The supplied README specifies:

| Setting | Value |
|---|---|
| Port | Port provided by J-Link On-Board |
| Speed | 115200 |
| Data | 8 bit |
| Parity | None |
| Stop bits | 1 bit |
| Flow control | None |

For Tera Term:

- Receive newline: **CR**
- Transmit newline: **CR**
- **Local echo** enabled

## What behavior and performance were demonstrated or measured?

### Demonstrated behavior

The supplied application demonstrates:

- Flash LP initialization and cleanup.
- Selection of data-flash or code-flash operations from a terminal.
- Validation of a selected block against configured address limits.
- Blank checking before erase.
- Erasing one block when required.
- Blank checking after erase.
- Writing a 64-byte test pattern.
- Reading flash by copying memory into a local buffer.
- Verifying that data read after the write matches the source buffer.
- Data-flash BGO callback handling when enabled.
- Setting and clearing a code-flash access window.
- Disabling interrupts around the demonstrated code-flash operation.

### Performance measurements

The supplied files do **not** provide measured erase time, write time, blank-check time, CPU utilization, flash endurance, RAM/ROM footprint, BGO latency, or other performance results.

`BLOCK_SIZE = 64` describes the amount processed by the example's blank-check/write operation. It is not a performance measurement. This SBB should therefore be treated as a functional Flash LP reference rather than a performance benchmark.

## What files are safe to modify and what files are generated, preset, or demo-only?

| File | Classification | Guidance |
|---|---|---|
| `flash_lp_ep.c` | Reusable SBB implementation | Modify when intentionally changing Flash LP wrappers, BGO waiting/callback behavior, write size, or access-window operations. |
| `flash_lp_ep.h` | Reusable SBB interface and device presets | Modify carefully when changing public APIs, block/address calculations, buffer sizes, or board-device memory-map assumptions. |
| `flash_lp_example_app.c` | Demo/reference application | Appropriate location for changing menu flow, test patterns, terminal interaction, validation policy, presentation, and demo sequencing. |
| `flash_lp_example_app.h` | Demo/reference application interface/presets | Contains the demo entry point, text, commands, and displayed board-dependent block ranges. |
| `readme.txt` | Existing example documentation | Source for the validated board list, debug/VCOM setup, access-window warning, input restrictions, and serial-terminal configuration. |

The hand-written files depend on generated FSP content through `hal_data.h`. The supplied files do not enumerate the generated files themselves, so individual generated files are not classified here.

## What happens if I modify the source and later adopt a newer SBB release?

Local modifications to `flash_lp_ep.c` or `flash_lp_ep.h` create differences in the reusable SBB layer. This can be particularly important for this SBB because `flash_lp_ep.h` contains MCU-dependent flash addresses and block geometry.

To reduce upgrade conflicts:

- Keep terminal menus, test patterns, formatted output, and other demonstration behavior in `flash_lp_example_app.*`.
- Keep board/device memory map assumptions explicit and review them separately from generic Flash LP operations.
- Prefer generated FSP configuration for settings owned by the Flash LP configurator rather than duplicating those settings in hand-written source.
- Keep reusable-source modifications focused and document why each change is required.
- When adopting a newer SBB release, compare the new `flash_lp_ep.c` and `flash_lp_ep.h` with the locally modified versions before carrying changes forward.
- Revalidate flash addresses, block sizes, selected block ranges, BGO configuration, callback handling, code-flash execution behavior, and access-window behavior after an update.

The supplied files do not define an automated migration or source-compatibility mechanism for later SBB versions. Compatibility must therefore be validated by the integrating project.

## Recommended reading order

1. **`flash_lp_ep.h`** to understand flash geometry, address calculations, public functions, and board-dependent configuration.
2. **`flash_lp_ep.c`** to understand how the SBB wraps blank check, erase, write, BGO events, and access-window APIs.
3. **`flash_lp_example_app.c`** to understand the full demo sequence and destructive flash operations.
4. **`flash_lp_example_app.h`** to understand menu commands and displayed block-range limits.
5. **`readme.txt`** to review the validated hardware list, board setup, access-window warning, and terminal configuration.
