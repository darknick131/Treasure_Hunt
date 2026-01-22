# Treasure Hunt Project 🏴‍☠️

**Treasure Hunt** is a sophisticated C-based application designed to simulate and manage treasure hunt activities using advanced operating system concepts. It employs a multi-process architecture to coordinate interactions between a central **Hub**, a background **Monitor**, and a **Treasure Manager**, ensuring robust data handling and real-time responsiveness.

---

## 🚀 Key Features

*   **Multi-Process Architecture**:  
    Utilizes `fork()` to create a distinct separation of concerns:
    *   **Hub**: The user-facing command-line interface (CLI) for control and interaction.
    *   **Monitor**: A background daemon-like process that manages system state and handles signal-based communication.
    *   **Treasure Manager**: A standalone executable invoked for CRUD operations on hunt data.
*   **Inter-Process Communication (IPC)**:
    *   **Signals**: Custom signal handlers (`handle_sigchld`, generic signal handlers) manage process lifecycles and notifications.
    *   **Pipes**: standard file descriptors are used to pipe data between components, ensuring seamless logic flow.
*   **Command-Line Interface**: A custom shell-like environment supporting commands for managing hunts, viewing treasures, and calculating scores.
*   **Score Calculation**: Integrated module to evaluate performance in hunts.
*   **Robust Error Handling**: Safely manages process termination and zombie processes suitable for UNIX-like environments (Linux/WSL).

---

## 🛠️ Installation & Build

### Prerequisites
*   **C Compiler**: `gcc` is required.
*   **Make**: GNU Make for build automation.
*   **Operating System**: Linux or WSL (Windows Subsystem for Linux), as the project relies on POSIX system calls (`<unistd.h>`, `<sys/wait.h>`).

### Building the Project
From the root directory, run the following command to compile all components:

```bash
make build
```

This will create an `init` step and standard executables in the `bin/` directory:
*   `bin/treasure_hub`
*   `bin/treasure_monitor`
*   `bin/treasure_manager`
*   `bin/treasure_calculator`

To clean up build artifacts:
```bash
make clean
```

---

## 💻 Usage

### Starting the Hub
The main entry point is the **Treasure Hub**. Start the application with:

```bash
./bin/treasure_hub
```

### Available Commands
Once inside the Hub shell (`> ` prompt), you can use the following commands:

| Command | Description |
| :--- | :--- |
| `start_monitor` | Launches the background Monitor process to handle active sessions. |
| `stop_monitor` | Safely shuts down the Monitor process. |
| `list_hunts` | Displays all available treasure hunts. |
| `list_treasures <hunt_name>` | Lists all treasures associated with a specific hunt. |
| `view_treasure <hunt_name> <treasure>` | Displays detailed information about a specific treasure. |
| `calculate_score <hunt_name>` | Calculates and displays the score for a given hunt. |
| `clear` | Clears the terminal screen. |
| `help` | Shows the list of available commands. |
| `exit` | Exits the Hub. (Note: Monitor must be stopped first). |

### Example Session
```bash
> start_monitor
Monitor started. Type commands to execute the treasure manager.
> list_hunts
[Displays list of hunts...]
> list_treasures Hunt001
[Displays treasures in Hunt001...]
> stop_monitor
> exit
```

---

## 📂 Project Structure

```plaintext
Treasure_Hunt/
├── bin/                  # Compiled executables
├── src/
│   ├── treasure_hub/     # Source code for the Hub & Monitor processes
│   │   ├── Hub/          # Main CLI logic (Command parsing, Loop)
│   │   ├── Monitor/      # Background process logic (Signal handling)
│   │   └── Calculate_Score/ # Logic for scoring mechanism
│   └── treasure_manager/ # Independent logic for data management (CRUD)
├── treasure_hunts/       # Data storage for specific hunts
└── makefile              # Build configuration
```

## ⚙️ Technical Details

This project is an excellent demonstration of **Systems Programming** in C. Key technical highlights include:
*   **Process Creation**: `fork()` and `exec` families are used to spawn the Monitor and Manager tools.
*   **Signal Handling**: `sigaction` and `signal` are used to handle `SIGCHLD` (avoiding zombies) and custom signals for graceful shutdowns.
*   **File I/O**: Low-level `read()` and `write()` calls are used for CLI interaction and file processing, bypassing some standard library buffers for precise control.

---

## 📜 License
This project is for educational and personal use.
