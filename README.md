<div align="center">

# Pintos on Windows 11 with WSL2

### A working Pintos development environment using Debian, WSL2, GCC and QEMU

</div>

---

## About This Repository

This repository is my fork of the JHU CS318 Pintos distribution.

I use it to learn and experiment with operating-system concepts such as:

- Kernel threads
- Process management
- CPU scheduling
- Synchronization
- System calls
- Virtual memory
- File systems
- Interrupt handling
- Kernel debugging

The original Pintos code was designed for older Linux toolchains. This README documents the setup that I successfully used on **Windows 11 with Debian running through WSL2**.

This method does not require:

- Installing Ubuntu as the main operating system
- Dual booting Windows and Linux
- Creating disk partitions
- Building the old GCC 6.2 cross-compiler

---

## Tested Environment

This setup was tested with:

| Component | Version |
|---|---|
| Host operating system | Windows 11 |
| Linux environment | Debian 13 through WSL2 |
| Architecture | x86-64 host, 32-bit x86 Pintos target |
| GCC | Debian GCC 14 |
| GNU Make | 4.4 |
| QEMU | 10 |
| Pintos source | JHU CS318 Pintos fork |

---

## How the Environment Works

```text
Windows 11
    ↓
WSL2
    ↓
Debian development environment
    ↓
GCC builds a 32-bit x86 Pintos kernel
    ↓
QEMU emulates an x86 computer
    ↓
Pintos boots inside the emulated computer
```

Pintos itself does not run as a normal Windows or Debian application.

The source code is compiled inside Debian, while QEMU creates the virtual hardware on which the Pintos kernel runs.

---

# Installation

## 1. Install WSL2 with Debian

Open **PowerShell as Administrator** and run:

```powershell
wsl --install -d Debian
```

Restart Windows if requested.

After restarting, open Debian from the Start menu and create a Linux username and password.

Check that Debian is using WSL2:

```powershell
wsl -l -v
```

Expected output:

```text
NAME      STATE      VERSION
Debian    Running    2
```

If Debian is using WSL1, convert it:

```powershell
wsl --set-version Debian 2
```

---

## 2. Update Debian

Open the Debian terminal and run:

```bash
sudo apt update
sudo apt full-upgrade -y
```

---

## 3. Install the Required Packages

```bash
sudo apt install -y \
  build-essential \
  gcc-multilib \
  binutils \
  git \
  make \
  perl \
  qemu-system-x86 \
  file
```

These packages provide:

- GCC and Make for compiling Pintos
- 32-bit x86 compilation support
- GNU Binutils for linking and inspecting binaries
- Perl for the Pintos launcher scripts
- QEMU for x86 hardware emulation
- Git for source-code management

---

## 4. Create a Workspace

Keep the Pintos project inside the Linux filesystem instead of `/mnt/c`.

```bash
mkdir -p ~/cs2043
cd ~/cs2043
```

Keeping the project under the Linux home directory avoids slower file operations and Windows/Linux permission problems.

---

## 5. Clone This Repository

```bash
git clone https://github.com/shashika-mora/pintos-wsl.git
cd pintos-wsl
```

---

# GCC Compatibility Fix

Modern Debian compilers generate position-independent executables by default.

Pintos is an old fixed-address kernel and should not be compiled as position-independent code. Without disabling PIE and PIC, Pintos may compile successfully but repeatedly reboot when QEMU tries to run it.

Open:

```text
src/Make.config
```

Find:

```makefile
CFLAGS = -m32 -g -msoft-float -O0
```

Change it to:

```makefile
CFLAGS = -m32 -g -msoft-float -O0 -fno-pie -fno-pic
```

The same change can be applied using:

```bash
sed -i \
  's/^CFLAGS = -m32 -g -msoft-float -O0$/CFLAGS = -m32 -g -msoft-float -O0 -fno-pie -fno-pic/' \
  src/Make.config
```

Verify the change:

```bash
grep '^CFLAGS' src/Make.config
```

Expected output:

```text
CFLAGS = -m32 -g -msoft-float -O0 -fno-pie -fno-pic
```

---

# Building Pintos

## 1. Build the Pintos Utilities

```bash
cd ~/cs2043/pintos-wsl/src/utils
make clean
make
```

This builds host-side tools such as:

```text
pintos
pintos-mkdisk
setitimer-helper
squish-pty
squish-unix
```

These programs run inside Debian and are used to start QEMU, create disk images and control Pintos tests.

---

## 2. Build the Threads Kernel

```bash
cd ~/cs2043/pintos-wsl/src/threads
make clean
make
```

A successful build creates files such as:

```text
build/kernel.o
build/kernel.bin
build/loader.bin
```

The build process is approximately:

```text
Pintos C and assembly source
            ↓
32-bit object files
            ↓
kernel.o
            ↓
kernel.bin
            ↓
QEMU loads and executes the kernel
```

Some warnings from modern GCC are expected because Pintos was written for older compilers.

Warnings are acceptable as long as the build finishes without:

```text
make: *** Error
```

---

# Running Pintos

Move into the build directory:

```bash
cd ~/cs2043/pintos-wsl/src/threads/build
```

Run the `alarm-single` kernel test:

```bash
../../utils/pintos -v --qemu -- -q run alarm-single
```

Arguments:

| Argument | Meaning |
|---|---|
| `../../utils/pintos` | Pintos launcher script |
| `-v` | Disable the separate graphical VGA window |
| `--qemu` | Use QEMU as the emulator |
| `--` | Pass the remaining arguments to Pintos |
| `-q` | Shut down after the test finishes |
| `run alarm-single` | Run the `alarm-single` thread test |

---

## Expected Successful Output

A successful boot should contain output similar to:

```text
Pintos booting with 3,968 kB RAM...
367 pages available in kernel pool.
367 pages available in user pool.
Calibrating timer...
Boot complete.

Executing 'alarm-single':
(alarm-single) begin
(alarm-single) Creating 5 threads to sleep 1 times each.
(alarm-single) thread 0: duration=10
(alarm-single) thread 1: duration=20
(alarm-single) thread 2: duration=30
(alarm-single) thread 3: duration=40
(alarm-single) thread 4: duration=50
(alarm-single) end

Execution of 'alarm-single' complete.
Powering off...
```

This confirms that:

- The bootloader executed
- The Pintos kernel was loaded
- Memory initialization succeeded
- Timer initialization succeeded
- Kernel threads were created
- The thread test completed
- Pintos shut down normally

---

# Understanding the Alarm Test

The `alarm-single` test creates five kernel threads.

```text
Thread 0 → sleeps for 10 timer ticks
Thread 1 → sleeps for 20 timer ticks
Thread 2 → sleeps for 30 timer ticks
Thread 3 → sleeps for 40 timer ticks
Thread 4 → sleeps for 50 timer ticks
```

The expected wake-up order is:

```text
Thread 0
Thread 1
Thread 2
Thread 3
Thread 4
```

This test connects to operating-system concepts such as:

```text
Timer interrupt
    ↓
Running thread goes to sleep
    ↓
Thread enters a blocked state
    ↓
Scheduler runs another thread
    ↓
Wake-up time is reached
    ↓
Thread returns to the ready state
```

---

# Common Problems

## Pintos Repeatedly Reboots

Example:

```text
Loading...
Kernel command line: -q run alarm-single
Pintos booting with Pintos hda1
Loading...
Kernel command line: -q run alarm-single
...
```

This usually means the kernel was compiled using incompatible position-independent code.

Confirm that `src/Make.config` contains:

```makefile
CFLAGS = -m32 -g -msoft-float -O0 -fno-pie -fno-pic
```

Then rebuild from a clean state:

```bash
cd ~/cs2043/pintos-wsl/src/threads
make clean
make
```

Run the test again:

```bash
cd build
../../utils/pintos -v --qemu -- -q run alarm-single
```

---

## `gcc -m32` Does Not Work

Install the 32-bit compiler support:

```bash
sudo apt install -y gcc-multilib
```

Test it:

```bash
printf 'void test(void) {}\n' |
gcc -m32 -ffreestanding -fno-pie -x c -c -o /tmp/pintos32.o -
```

Inspect the output:

```bash
file /tmp/pintos32.o
```

Expected:

```text
ELF 32-bit LSB relocatable, Intel 80386
```

---

## `file: command not found`

```bash
sudo apt install -y file
```

---

## `qemu-system-i386: command not found`

```bash
sudo apt install -y qemu-system-x86
```

Verify:

```bash
qemu-system-i386 --version
```

---

## Stop a Running Pintos Test

Press:

```text
Ctrl + C
```

If QEMU has captured the keyboard or mouse, use:

```text
Ctrl + Alt + G
```

to release it.

---

# Useful Commands

## Rebuild the Threads Kernel

```bash
cd ~/cs2043/pintos-wsl/src/threads
make clean
make
```

## Run `alarm-single`

```bash
cd ~/cs2043/pintos-wsl/src/threads/build
../../utils/pintos -v --qemu -- -q run alarm-single
```

## Inspect the Kernel Binary

```bash
file kernel.o
file kernel.bin
```

## Check Repository Changes

```bash
cd ~/cs2043/pintos-wsl
git status
git diff
```

## Open the Current WSL Directory in Windows Explorer

```bash
explorer.exe .
```

---

# Important Source Directories

```text
src/
├── threads/       Kernel threads, scheduling and synchronization
├── userprog/      User processes and system calls
├── vm/            Virtual-memory implementation
├── filesys/       File-system implementation
├── devices/       Timer, keyboard, disk and shutdown devices
├── lib/           Pintos libraries and data structures
├── tests/         Automated Pintos tests
├── utils/         Launcher and host-side utility programs
└── misc/          Toolchain and support scripts
```

---

# Learning Path

I plan to use this repository in the following order:

1. Understand the Pintos directory structure
2. Trace the kernel boot process
3. Study `threads/init.c`
4. Study kernel-thread creation
5. Understand timer interrupts
6. Study thread scheduling
7. Study synchronization primitives
8. Learn kernel debugging with GDB
9. Move into processes and system calls
10. Explore virtual memory and file systems

---

# Notes

- This setup uses the normal Debian compiler.
- Building the old GCC 6.2 Pintos cross-compiler is not required for this setup.
- The `-fno-pie` and `-fno-pic` flags are necessary when using modern Debian GCC versions.
- Pintos should be stored inside the WSL Linux filesystem.
- Headless QEMU mode is recommended because it keeps all output inside the terminal.
- This setup was tested with the JHU CS318 Pintos distribution.

---

# Credits

Pintos was originally developed at Stanford University as a teaching operating system.

This repository is based on the JHU CS318 Pintos distribution:

- [JHU CS318 Pintos](https://github.com/jhu-cs318/pintos)
- [Original Stanford Pintos Documentation](https://web.stanford.edu/class/cs140/projects/pintos/pintos.html)

---

<div align="center">

Built as part of my Operating Systems learning journey.

</div>

One recommendation: commit the `src/Make.config` compatibility change into your fork. Then people cloning `pintos-wsl` will not need to apply the `sed` command manually, and the README can later describe that fix as already included.
