# nEXT2shell: Interactive EXT2 Filesystem Shell

An interactive user-space shell and filesystem manipulation tool written in C/C++ that parses, navigates, and modifies raw disk images formatted with the Second Extended Filesystem (**EXT2**) without mounting them into the host operating system.

---

## 📖 About / What It Is

`nEXT2shell` is an educational, user-space shell application designed to inspect and manipulate an EXT2 virtual disk image (`myext2image.img`) at byte-level precision. 

Rather than relying on kernel drivers, OS-level loop devices, or system utilities (such as `mount`, `system()`, or `exec()`), `nEXT2shell` directly reads from and writes to raw disk offsets via standard low-level I/O system calls (`open`, `lseek`, `read`, `write`). It parses fundamental on-disk structures defined by the EXT2 specification, including the superblock, block group descriptors, block/inode bitmaps, inode tables, and directory entries.

---

## 🎯 Purpose / Why It Was Created

This project was developed as an academic practical project for an **Operating Systems** course in a **Computer Science** (B.Sc.) degree program. 

The primary objective was to gain an in-depth, hands-on understanding of filesystem internals, low-level disk layouts, and storage management mechanisms, including:
- **On-Disk Metadata Structures:** Decoding the superblock, block group descriptor table, and inode tables.
- **Allocation & Deallocation:** Managing block and inode allocation bitmaps during file/directory creation and deletion.
- **Hierarchical Path Resolution & Directory Layout:** Parsing variable-length linked directory entries (`ext2_dir_entry_2`) and updating record lengths (`rec_len`).
- **Data Block Addressing & Indirection:** Traversing direct block pointers and single-level indirect block tables to read and free file contents.

For further theoretical and technical details on the EXT2 filesystem architecture and specification, refer to the reference documentation and source examples provided in the `aditional reading` directory:
- 📚 [Additional Reading Materials](aditional%20reading/) — contains foundational reference guides (*Design and Implementation of the Second Extended Filesystem*, *The Ext2 Filesystem*, *The Second Extended File System*) as well as reference implementations and headers (`ext2_fs.h`).

---

## 📦 Critical Dependency: GNU Readline (`readline/readline.h`)

> [!IMPORTANT]
> **Prerequisite Header & Library: GNU Readline (`<readline/readline.h>`)**
>
> This program relies on the **GNU Readline** library (`<readline/readline.h>` and `<readline/history.h>`) to provide an interactive shell experience with inline command editing, line navigation, auto-refreshing prompts showing the current directory path, and command history persistence (`add_history`).

If the Readline development headers are not installed on your Linux system, compilation will fail with:
```text
fatal error: readline/readline.h: No such file or directory
```

### Installing GNU Readline:
On Debian, Ubuntu, Linux Mint, or other APT-based distributions, install the development package before compiling:

```bash
sudo apt update
sudo apt install libreadline-dev
```

On Fedora/RHEL/CentOS:
```bash
sudo dnf install readline-devel
```

On Arch Linux:
```bash
sudo pacman -S readline
```

---

## 💻 Implemented Commands

Once `nEXT2shell` is running, it starts at the root directory (`/`) of the virtual disk and displays a dynamic prompt: `[/]$> `. The following commands are supported:

### 1. Filesystem & Metadata Inspection
- **`info`**
  - **Syntax:** `info`
  - **Description:** Reads the primary superblock and displays essential volume and filesystem statistics, including:
    - Volume name and total image size (in bytes)
    - Free disk space (in KiB)
    - Total and free inodes / blocks
    - Block size and inode structure size
    - Block group counts, blocks/inodes per group, and inode table size.

- **`attr <file | dir>`**
  - **Syntax:** `attr <filename>` or `attr <dirname>`
  - **Description:** Displays metadata and inode attributes for a specific file or directory located in the current working directory. Decodes `i_mode` permissions (e.g., `frw-r--r--` for files or `drwxr-xr-x` for directories), user ID (`uid`), group ID (`gid`), size in bytes/KiB, and formatted last modification timestamp (`i_mtime`).

### 2. Navigation & Directory Listing
- **`ls`**
  - **Syntax:** `ls`
  - **Description:** Traverses the directory entries stored within the current directory's data block. For each entry, it prints:
    - Entry name (including `.` and `..`)
    - Inode number
    - Record length (`record length / rec_len`)
    - Name length (`name length`)
    - File type (`file type`: 1 = regular file, 2 = directory).

- **`cd <path>`**
  - **Syntax:** `cd <dir_name>` or `cd ..`
  - **Description:** Changes the current working directory within the virtual disk. Supports stepping into subdirectories or returning to the parent directory (`..`). Updates the interactive shell prompt with the new path.

- **`pwd`**
  - **Syntax:** `pwd`
  - **Description:** Prints the current absolute path in the virtual disk hierarchy (e.g., `/documentos/` or `/livros/classicos/`).

### 3. File & Directory Viewing
- **`cat <file>`**
  - **Syntax:** `cat <filename>`
  - **Description:** Opens a regular file from the current directory, reads its contents block by block, and prints the text content directly to the terminal.

### 4. File Creation & Modification
- **`touch <file>`**
  - **Syntax:** `touch <new_filename>`
  - **Description:** Creates a new empty regular file.
    - Searches the inode bitmap for the first available free inode.
    - Marks the inode as allocated in the bitmap.
    - Initializes an `ext2_inode` struct (setting file mode to regular file with default permissions, zero size, link count = 1, and current timestamps).
    - Appends the new `ext2_dir_entry_2` record into the current directory block, adjusting the previous entry's `rec_len` accordingly.

- **`mkdir <dir>`**
  - **Syntax:** `mkdir <new_dirname>`
  - **Description:** Creates a new empty subdirectory.
    - Allocates a free inode and a free data block from their respective bitmaps.
    - Initializes the directory data block with default `.` (pointing to self) and `..` (pointing to parent directory) entries.
    - Adds the directory record to the current directory block and updates inode link counts and directory counters.

- **`rename <file> <newfilename>`**
  - **Syntax:** `rename <old_filename> <new_filename>`
  - **Description:** Renames an existing file or directory inside the current directory. Adjusts directory entry records and realigns offsets within the directory data block to maintain contiguous structure integrity.

### 5. File & Directory Deletion
- **`rm <file>`**
  - **Syntax:** `rm <filename>`
  - **Description:** Deletes a regular file from the filesystem.
    - Locates the file's inode and frees all associated data blocks (including direct blocks and blocks referenced through indirect pointers) by clearing their bits in the block bitmap.
    - Marks the file's inode as free in the inode bitmap.
    - Removes the directory entry from the parent directory block, expanding the previous record's `rec_len` to cover the removed slot.

- **`rmdir <dir>`**
  - **Syntax:** `rmdir <dirname>`
  - **Description:** Deletes a directory, provided that it is empty (contains only `.` and `..`). Deallocates its directory data block, frees its inode in the inode bitmap, decrements parent directory link counts, and removes the directory entry.

### 6. Exporting Files to Host OS
- **`cp <source_file> <target_path>`**
  - **Syntax:** `cp <filename_in_image> <destination_path_on_host>`
  - **Description:** Copies a file out of the EXT2 virtual disk into the host operating system's filesystem. Reads the file data from the disk image and writes it to the designated external destination file.

### 7. Shell Session Control
- **`exit`**
  - **Syntax:** `exit`
  - **Description:** Terminates the `nEXT2shell` interactive session and returns to the host terminal.

---

## ⚙️ Environment & Technical Constraints

- **Host Platform:** Linux (developed and tested on **Linux Mint 20.3 Una 64-bit**, Kernel `5.4.0` / Ubuntu).
- **Target Image Format:** EXT2 disk image with a fixed block size of **1024 bytes** (`-b 1024`), 64 MiB total volume size.
- **Known Limitations & Simplifications:**
  - Commands accept space-separated arguments; filenames containing whitespace characters are not supported.
  - New inode/block allocations (`touch`, `mkdir`) prioritize and allocate from block group 0.
  - Directory entries are expected to fit within a single 1024-byte directory block.
  - Files supported up to 64 MiB using direct and single-level indirect blocks.

---

## 🚀 How to Build and Execute

### 1. Install Prerequisites
Ensure you have `g++`, `make`, and the GNU Readline development library installed:

```bash
sudo apt update
sudo apt install build-essential libreadline-dev
```

### 2. Verify or Prepare the Disk Image
The shell expects a raw EXT2 image named `myext2image.img` inside the `implementation` directory:
- A pre-configured disk image (`myext2image.img`) is already provided inside `implementation/`.
- A clean backup copy is also preserved in `implementation/image_copy/myext2image.img`.

*(Optional)* If you wish to create a fresh 64 MiB EXT2 image from scratch:
```bash
cd implementation
dd if=/dev/zero of=./myext2image.img bs=1024 count=64K
mkfs.ext2 -b 1024 ./myext2image.img
cd ..
```

### 3. Build the Application
Navigate into the `implementation` directory and compile using `make`:

```bash
cd implementation
make
```
This compiles `nEXT2shell.cpp` with `g++ -Wall` and links against `-lreadline`, creating the executable `./nEXT2shell`.

### 4. Run the Shell
Execute the binary from within `implementation`:

```bash
./nEXT2shell
```

Upon launching, the interactive prompt will appear:
```text
[/]$> info
[/]$> ls
[/]$> cd documentos
[/documentos/]$> ls
[/documentos/]$> cat alfabeto.txt
[/documentos/]$> cd ..
[/]$> exit
```

### 5. Other Makefile Targets
From within the `implementation` directory:
- **Run in Debug Mode (Compile, Run & Clean):**
  ```bash
  make debug
  ```
- **Clean Build Artifacts:**
  ```bash
  make clean
  ```
- **Restore Clean Virtual Disk Image:**
  If the disk image becomes corrupted or you wish to reset changes, you can restore from `image_copy`:
  ```bash
  cp image_copy/myext2image.img ./myext2image.img
  ```

### 6. Validating Filesystem Integrity
From within the `implementation` directory, you can verify the consistency of the virtual disk at any time using Linux's standard `e2fsck` utility:

```bash
e2fsck myext2image.img
```

---

## ⚖️ Academic Context & Disclaimer

This software was developed solely for academic and educational purposes to demonstrate fundamental operating systems principles and filesystem structure mechanics.
