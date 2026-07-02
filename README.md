# Treasure Hunt

> A multi-process C application that demonstrates core operating-system concepts — `fork`/`exec`, signal-based IPC, anonymous pipes, and binary file I/O — through a command-line treasure-hunt management system.

![Language](https://img.shields.io/badge/language-C-blue)
![Build](https://img.shields.io/badge/build-GNU%20Make-orange)
![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20WSL-lightgrey)
![License](https://img.shields.io/badge/license-Educational-green)

---

## Overview

Treasure Hunt is a university systems-programming project that manages geolocation-based treasure hunts stored as packed binary files. Four cooperating executables are compiled from a single codebase: a user-facing **hub**, a background **monitor** daemon, a **treasure manager** for CRUD operations, and a **score calculator**. Every hub command travels through a signal-and-pipe IPC chain before touching disk, making the data flow an explicit demonstration of POSIX process coordination rather than a library abstraction.

Each hunt is a directory under `treasure_hunts/` containing a flat binary file of `Treasure_T` records and a plain-text audit log. A symlink in the working directory provides a shortcut to each log. The entire stack uses only POSIX system calls — no third-party libraries.

---

## Key Features

- **Four-binary architecture** — `treasure_hub`, `treasure_monitor`, `treasure_manager`, and `treasure_calculator` are built as separate executables and composed at runtime via `fork` + `execl`.
- **Signal-based command dispatch** — the hub writes a command to `monitor_command.txt`, then wakes the monitor with `SIGUSR1`; the monitor's main loop is a `pause()` loop that processes one command per signal.
- **Pipe-based output routing** — when `start_monitor` is issued, the hub opens an anonymous pipe; the monitor inherits the write end (passed as `argv[1]`) and funnels all child-process stdout back through it to the hub's terminal.
- **Binary flat-file storage** — treasures are stored as fixed-size `Treasure_T` structs written contiguously to `treasure.bin`; removal is implemented as a temp-file copy-and-rename to avoid partial writes.
- **Per-hunt audit logging** — every add, list, view, and remove operation appends a timestamped entry to `logged_hunt.txt`; a symlink `logged_hunt-<hunt_id>` is created in the working directory on first write.
- **Score aggregation** — `treasure_calculator` reads the binary file and builds a singly-linked list of `(username, totalScore)` nodes, accumulating values in a single sequential pass.
- **Zombie prevention** — the hub installs a `SIGCHLD` handler with `SA_NOCLDSTOP` that calls `waitpid(-1, WNOHANG)` in a loop, reaping all finished child processes including transient `treasure_manager` forks.

---

## System Architecture & Data Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                    treasure_hub  (foreground process)                │
│                                                                      │
│  write("> ")  read(stdin)  ──►  get_command_type()                  │
│                                        │                             │
│                              execute_command()                       │
│                                        │                             │
│            ┌───────────────────────────┤                             │
│            │  start_monitor            │  other commands             │
│            │  fork() + execl()         │  send_command_to_monitor()  │
│            │         │                 │    1. write → monitor_command.txt
│            │         │                 │    2. kill(monitor_pid, SIGUSR1)
│            │         │                 │    3. read(fd_for_pipe) ◄──┐│
│            │         ▼                 │                            ││
│            │  [pipe read end kept]     │                            ││
└────────────┼──────────────────────────┼────────────────────────────┼┘
             │                          │                             │
             │ fork/execl               │                             │
             ▼                          │                             │
┌───────────────────────────────────────┼─────────────────────────────┤
│             treasure_monitor          │   (child of hub)            │
│                                       │                             │
│   setup_signal_handlers(SIGUSR1)      │                             │
│   while(1) { pause() }                │                             │
│        │                              │                             │
│     SIGUSR1 ──► is_running=1          │                             │
│                    │                  │                             │
│              process_command()        │                             │
│              read(monitor_command.txt)│                             │
│                    │                  │                             │
│        ┌───────────┴───────────┐      │                             │
│        │                       │      │                             │
│   list_hunts()    exec_treasure_manager() / exec_calculate_score()  │
│   (opendir)       fork() + execl(), pipe stdout                     │
│        │                       │                                    │
│        └──────────────── write(output_fd_pipe) ────────────────────►│
└────────────────────────────────────────────────────────────────────-┘
                          │ fork/execl (per command)
               ┌──────────┴──────────┐
               ▼                     ▼
   ┌─────────────────────┐  ┌──────────────────────┐
   │   treasure_manager  │  │  treasure_calculator  │
   │  --add    <hunt_id> │  │  <hunt_id>            │
   │  --list   <hunt_id> │  │                       │
   │  --view   <hunt_id> │  │  reads treasure.bin   │
   │           <id>      │  │  builds linked list   │
   │  --remove_treasure  │  │  prints user scores   │
   │  --remove_hunt      │  └──────────────────────┘
   └─────────────────────┘
```

### Data layout on disk

```
treasure_hunts/
└── <hunt_id>/
    ├── treasure.bin        # sequential packed Treasure_T records (binary)
    └── logged_hunt.txt     # append-only plain-text audit log

logged_hunt-<hunt_id>       # symlink → treasure_hunts/<hunt_id>/logged_hunt.txt
monitor_command.txt         # transient IPC file; overwritten before each SIGUSR1
```

### `Treasure_T` record layout (56 bytes on typical x86-64)

| Field                     | Type     | Max length |
| ------------------------- | -------- | ---------- |
| `id`                      | `char[]` | 16         |
| `userName`                | `char[]` | 16         |
| `GPSCoordinate.latitude`  | `float`  | —          |
| `GPSCoordinate.longitude` | `float`  | —          |
| `clueText`                | `char[]` | 128        |
| `value`                   | `int`    | —          |

---

## Project Structure

```
Treasure_Hunt/
├── bin/                          # Compiled executables (git-ignored)
│   ├── treasure_hub
│   ├── treasure_monitor
│   ├── treasure_manager
│   └── treasure_calculator
│
├── src/
│   ├── treasure_hub/
│   │   ├── Hub/                  # Hub entry point and command dispatch
│   │   │   ├── hub.c / hub.h
│   │   │   └── Libraries/
│   │   │       ├── hub_commands/ # Command parsing and routing (get_command_type)
│   │   │       ├── hub_control/  # Process lifecycle: fork, exec, pipe, kill
│   │   │       └── hub_signal_handler/ # SIGCHLD handler (zombie reaping)
│   │   ├── Monitor/              # Monitor entry point (pause loop + SIGUSR1)
│   │   │   ├── monitor.c / monitor.h
│   │   │   └── Libraries/
│   │   │       ├── list_hunts/   # Direct directory scan for list_hunts command
│   │   │       ├── monitor_commands/ # Command router: exec_treasure_manager, exec_calculate_score
│   │   │       └── monitor_signal_handler/ # SIGUSR1 handler
│   │   └── Calculate_Score/      # Score calculator entry point + linked-list accumulator
│   │       ├── calculate_score.c / calculate_score.h
│   │
│   └── treasure_manager/         # Standalone CRUD tool (invoked by monitor via execl)
│       ├── treasure_manager.c / treasure_manager.h   # Entry point, operation enum, Treasure_T
│       └── Libraries/
│           ├── manage_argv/      # CLI flag parsing (--add, --list, --view, etc.)
│           ├── manage_operations/
│           │   ├── add/          # Interactive treasure creation + binary write
│           │   ├── list/         # Sequential read + pretty-print, view by ID
│           │   └── remove/       # Temp-file copy-and-rename removal
│           └── helper_functions/ # Shared: create_treasure(), DIR/FILE helpers, symlink_create()
│
├── treasure_hunts/               # Runtime data (one subdirectory per hunt)
│   ├── Hunt001/
│   │   ├── treasure.bin
│   │   └── logged_hunt.txt
│   └── Hunt003/
│       ├── treasure.bin
│       └── logged_hunt.txt
│
├── OLD/                          # Superseded prototypes (not built)
│   ├── incercare_0/              # First attempt: single-file treasure_manager
│   ├── incercare_01/             # Second attempt: standalone monitor
│   ├── old_hub/
│   └── old_monitor/
│
├── makefile                      # Build all four executables into bin/
├── monitor_command.txt           # Runtime IPC file (hub → monitor command channel)
└── Docs.pdf                      # Project specification
```

---

## Tech Stack

| Layer           | Technology            | Role                                                                     |
| --------------- | --------------------- | ------------------------------------------------------------------------ |
| Language        | C (C99, POSIX.1-2008) | Entire codebase — direct access to system calls                          |
| Build           | GNU Make + gcc        | Single `make build` compiles all four targets with `-Wall`               |
| Process model   | `fork` / `execl`      | Spawning monitor and per-command tool invocations                        |
| IPC — commands  | `SIGUSR1` + file      | Hub signals monitor; command string is in `monitor_command.txt`          |
| IPC — output    | Anonymous pipe        | Monitor writes results back to hub over a pipe opened at `start_monitor` |
| IPC — lifecycle | `SIGTERM` / `SIGCHLD` | Graceful monitor shutdown; zombie reaping via `waitpid`                  |
| Storage         | Binary flat file      | Fixed-size struct records in `treasure.bin`; no external database        |
| Audit trail     | Append-only text file | Every operation logged to `logged_hunt.txt`                              |
| Shortcuts       | `symlink(2)`          | `logged_hunt-<hunt_id>` created in working directory on first add        |
| Platform        | Linux / WSL           | Requires POSIX — does not build natively on Windows                      |

---

## Getting Started

### Prerequisites

- **gcc** — any version supporting C99
- **GNU Make**
- **Linux** or **WSL** (Windows Subsystem for Linux) — the code uses `fork`, `sigaction`, `pipe`, and `execl`

### Build

```bash
# from the repository root
make build
```

This compiles four executables into `bin/`:

```
bin/treasure_hub
bin/treasure_monitor
bin/treasure_manager
bin/treasure_calculator
```

### Clean

```bash
make clean   # removes bin/treasure_manager only (see Assumptions to verify)
```

### Run

All commands must be run from the **repository root** — the executables use relative paths (`bin/treasure_manager`, `treasure_hunts/`) that assume the working directory is the project root.

```bash
./bin/treasure_hub
```

---

## Usage

### Hub shell

`treasure_hub` presents a `> ` prompt. The monitor must be running for most commands.

```
> start_monitor
Monitor started. Type commands to execute the treasure manager.
> list_hunts
Hunt: Hunt001 | Treasures: 3
Hunt: Hunt003 | Treasures: 1
> list_treasures Hunt001
Hunt name: Hunt001
...
ID: Daria
User: Daria_Ciobanu
Latitude: 44.430000
Longitude: 26.100000
Clue: Under the oak tree near the fountain
Value: 150
> view_treasure Hunt001 Anisia
Searching for treasure with ID: Anisia
ID: Anisia
Username: Anisia_Ciobanu
...
> calculate_score Hunt001
--------------
Hunt directory: Hunt001
---------------
Daria_Ciobanu: 300
Anisia_Ciobanu: 100
> stop_monitor
Monitor stopped.
> exit
```

| Command                                 | Description                                                |
| --------------------------------------- | ---------------------------------------------------------- |
| `start_monitor`                         | Forks and execs `bin/treasure_monitor`; opens the IPC pipe |
| `stop_monitor`                          | Sends `SIGTERM` to the monitor and waits for it to exit    |
| `list_hunts`                            | Lists all hunt directories with their treasure count       |
| `list_treasures <hunt_id>`              | Prints all treasure records in the named hunt              |
| `view_treasure <hunt_id> <treasure_id>` | Prints a single treasure by ID                             |
| `calculate_score <hunt_id>`             | Prints total score per user in the named hunt              |
| `help`                                  | Prints available commands                                  |
| `clear`                                 | Clears the terminal                                        |
| `exit`                                  | Exits (blocked until `stop_monitor` is called first)       |

### Treasure Manager (direct CLI)

`treasure_manager` can also be called directly, bypassing the hub entirely:

```bash
# Interactively add a treasure to Hunt001
./bin/treasure_manager --add Hunt001

# List all treasures in Hunt001
./bin/treasure_manager --list Hunt001

# View a specific treasure by ID
./bin/treasure_manager --view Hunt001 <treasure_id>

# Remove a specific treasure
./bin/treasure_manager --remove_treasure Hunt001 <treasure_id>

# Remove an entire hunt (deletes all files, directory, and symlink)
./bin/treasure_manager --remove_hunt Hunt001
```

`--add` reads all fields interactively from stdin:

```
Enter treasure data:
ID (max 15 chars): T007
Username (max 31 chars): Alice
Latitude: 44.43
Longitude: 26.10
Clue text: Behind the old library clock
Value: 250
```

### Score Calculator (direct CLI)

```bash
./bin/treasure_calculator Hunt001
```

Output:

```
--------------
Hunt directory: Hunt001
---------------
Alice: 250
Daria_Ciobanu: 300
```

---

## License & Contributing

This project was developed as a university systems-programming assignment and is shared here for portfolio and educational purposes. It is not open for external contributions at this time.

---

## Assumptions to Verify

The following are inferences made during reverse-engineering. Verify before sharing widely:

- **`make clean` scope** — the `clean` target in `makefile` only removes `bin/treasure_manager`; the other three binaries are not cleaned. This appears to be an oversight.
- **`userName` field size** — `Treasure_T.userName` is declared with `MAX_ID_LEN` (16 chars) rather than `MAX_USERNAME_LEN` (32 chars); the `--add` prompt says "max 31 chars" but only 15 chars will be stored without truncation.
- **5-second monitor shutdown delay** — `process_command()` sleeps 5 seconds when it receives `stop_monitor` before exiting; this may have been a course requirement or debug artifact.
- **Single in-flight command** — `monitor_command.txt` holds exactly one command at a time with no locking. The protocol serialises this naturally (hub blocks on `read(pipe)` until monitor responds), but rapid concurrent calls would corrupt the file.
- **Relative path dependency** — hardcoded paths (`bin/treasure_manager`, `treasure_hunts/`) require the process working directory to be the repository root at all times.
