<div align="center">

Pintos on Windows 11 with WSL2

A working Pintos development environment using Ubuntu or Debian, WSL2, GCC and QEMU

</div>

About This Repository

This repository is my fork of the JHU CS318 Pintos distribution.

I use it to learn and experiment with operating-system concepts such as:

Kernel threads

Process management

CPU scheduling

Synchronization

System calls

Virtual memory

File systems

Interrupt handling

Kernel debugging

The original Pintos code was designed for older Linux toolchains. This README documents the WSL2 setup I used for Pintos on Windows 11 with Ubuntu or Debian.

This method does not require:

Installing Linux as the main operating system

Dual booting Windows and Linux

Creating disk partitions

Building the old GCC 6.2 cross-compiler

Tested Environment

This setup was tested with:

Component

Version

Host operating system

Windows 11

Linux environment

Debian 13 or Ubuntu 26.04 LTS through WSL2

Architecture

x86-64 host, 32-bit x86 Pintos target

Compiler

Distribution-provided GCC

Build tools

GNU Make and Binutils

Emulator

QEMU

Pintos source

JHU CS318 Pintos fork

The full Pintos boot and alarm-single test were verified on Debian 13. The same source, compatibility flags and build process were also verified on Ubuntu 26.04 LTS.

How the Environment Works

Windows 11
    ↓
WSL2
    ↓
Ubuntu or Debian development environment
    ↓
GCC builds a 32-bit x86 Pintos kernel
    ↓
QEMU emulates an x86 computer
    ↓
Pintos boots inside the emulated computer

Pintos itself does not run as a normal Windows, Ubuntu or Debian application.

The source code is compiled inside the selected WSL2 Linux distribution, while QEMU creates the virtual hardware on which the Pintos kernel runs.

Installation

1. Install WSL2 with Ubuntu or Debian

Open PowerShell as Administrator and install one of the following distributions.

Ubuntu

wsl --install -d Ubuntu

Debian

wsl --install -d Debian

Restart Windows if requested.

After restarting, open the installed distribution from the Start menu and create a Linux username and password.

Check that it is using WSL2:

wsl -l -v

Example output:

NAME      STATE      VERSION
Ubuntu    Running    2

or:

NAME      STATE      VERSION
Debian    Running    2

If the selected distribution is using WSL1, convert it by using its exact name:

wsl --set-version Ubuntu 2

or:

wsl --set-version Debian 2

2. Update Ubuntu or Debian

Open the Ubuntu or Debian terminal and run:

sudo apt update
sudo apt full-upgrade -y

3. Install the Required Packages

sudo apt install -y \
  build-essential \
  gcc-multilib \
  binutils \
  git \
  make \
  perl \
  qemu-system-x86 \
  file

These packages provide:

GCC and Make for compiling Pintos

32-bit x86 compilation support

GNU Binutils for linking and inspecting binaries

Perl for the Pintos launcher scripts

QEMU for x86 hardware emulation

Git for source-code management

4. Create a Workspace

Keep the Pintos project inside the Linux filesystem instead of /mnt/c.

mkdir -p ~/cs2043
cd ~/cs2043

Keeping the project under the Linux home directory avoids slower file operations and Windows/Linux permission problems.

5. Clone This Repository

git clone https://github.com/shashika-mora/pintos-wsl.git pintos
cd pintos

GCC Compatibility Fix

Modern Ubuntu and Debian compilers may generate position-independent executables by default.

Pintos is an old fixed-address kernel and should not be compiled as position-independent code. Without disabling PIE and PIC, Pintos may compile successfully but repeatedly reboot when QEMU tries to run it.

Open:

src/Make.config

Find:

CFLAGS = -m32 -g -msoft-float -O0

Change it to:

CFLAGS = -m32 -g -msoft-float -O0 -fno-pie -fno-pic

The same change can be applied using:

sed -i \
  's/^CFLAGS = -m32 -g -msoft-float -O0$/CFLAGS = -m32 -g -msoft-float -O0 -fno-pie -fno-pic/' \
  src/Make.config

Verify the change:

grep '^CFLAGS' src/Make.config

Expected output:

CFLAGS = -m32 -g -msoft-float -O0 -fno-pie -fno-pic

Building Pintos

1. Build the Pintos Utilities

cd ~/cs2043/pintos/src/utils
make clean
make

Make sure the Pintos launcher script is executable:

chmod +x ~/cs2043/pintos/src/utils/pintos

Add the Pintos utilities directory to PATH:

echo 'export PATH="$HOME/cs2043/pintos/src/utils:$PATH"' >> ~/.bashrc
source ~/.bashrc

Verify that the pintos command is available:

which pintos

The output should end with:

cs2043/pintos/src/utils/pintos

This builds host-side tools such as:

pintos
pintos-mkdisk
setitimer-helper
squish-pty
squish-unix

These programs run inside the WSL2 Linux environment and are used to start QEMU, create disk images and control Pintos tests.

2. Build the Threads Kernel

cd ~/cs2043/pintos/src/threads
make clean
make

A successful build creates files such as:

build/kernel.o
build/kernel.bin
build/loader.bin

The build process is approximately:

Pintos C and assembly source
            ↓
32-bit object files
            ↓
kernel.o
            ↓
kernel.bin
            ↓
QEMU loads and executes the kernel

Some warnings from modern GCC are expected because Pintos was written for older compilers.

Warnings are acceptable as long as the build finishes without:

make: *** Error

Running Pintos

Basic Pintos Boot

The following is the basic command sequence used in the Pintos installation guide.

Build the threads kernel:

cd ~/cs2043/pintos/src/threads
make

Move into the build directory:

cd build

Boot Pintos:

pintos --

If the installation is working, Pintos should boot and eventually display:

Boot complete.

The basic command may open a separate QEMU window. Press Ctrl + C in the Ubuntu or Debian terminal when you need to stop it.

Run Pintos Inside the Terminal

To keep the QEMU output inside the Ubuntu or Debian terminal and disable the separate VGA window, use:

pintos -v --qemu --

The -v option disables VGA output, while --qemu explicitly selects QEMU.

Run the alarm-single Test

From the same build directory, run:

pintos -v --qemu -- -q run alarm-single

Arguments:

Argument

Meaning

pintos

Pintos launcher script available through PATH

-v

Disable the separate graphical VGA window

--qemu

Use QEMU as the emulator

--

Pass the remaining arguments to the Pintos kernel

-q

Shut down Pintos after the command finishes

run alarm-single

Run the alarm-single kernel-thread test

Expected Successful Test Output

A successful test should contain output similar to:

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

This confirms that:

The bootloader executed

The Pintos kernel was loaded

Memory initialization succeeded

Timer initialization succeeded

Kernel threads were created

The thread test completed

Pintos shut down normally

Understanding the Alarm Test

The alarm-single test creates five kernel threads.

Thread 0 → sleeps for 10 timer ticks
Thread 1 → sleeps for 20 timer ticks
Thread 2 → sleeps for 30 timer ticks
Thread 3 → sleeps for 40 timer ticks
Thread 4 → sleeps for 50 timer ticks

The expected wake-up order is:

Thread 0
Thread 1
Thread 2
Thread 3
Thread 4

This test connects to operating-system concepts such as:

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

Common Problems

pintos: Permission denied

Give the Pintos launcher script executable permission:

chmod +x ~/cs2043/pintos/src/utils/pintos

Verify the permission:

ls -l ~/cs2043/pintos/src/utils/pintos

The permission string should contain x, for example:

-rwxr-xr-x

Then retry from the threads build directory:

cd ~/cs2043/pintos/src/threads/build
../../utils/pintos -v --qemu -- -q run alarm-single

pintos: command not found

Make sure the Pintos utilities were built:

cd ~/cs2043/pintos/src/utils
make

Add the utilities directory to PATH:

echo 'export PATH="$HOME/cs2043/pintos/src/utils:$PATH"' >> ~/.bashrc
source ~/.bashrc

Verify the command:

which pintos

As a direct fallback, the launcher can also be run from the threads build directory using:

../../utils/pintos -v --qemu --

Pintos Repeatedly Reboots

Example:

Loading...
Kernel command line: -q run alarm-single
Pintos booting with Pintos hda1
Loading...
Kernel command line: -q run alarm-single
...

This usually means the kernel was compiled using incompatible position-independent code.

Confirm that src/Make.config contains:

CFLAGS = -m32 -g -msoft-float -O0 -fno-pie -fno-pic

Then rebuild from a clean state:

cd ~/cs2043/pintos/src/threads
make clean
make

Run the test again:

cd build
pintos -v --qemu -- -q run alarm-single

gcc -m32 Does Not Work

Install the 32-bit compiler support:

sudo apt install -y gcc-multilib

Test it:

printf 'void test(void) {}\n' |
gcc -m32 -ffreestanding -fno-pie -x c -c -o /tmp/pintos32.o -

Inspect the output:

file /tmp/pintos32.o

Expected:

ELF 32-bit LSB relocatable, Intel 80386

file: command not found

sudo apt install -y file

qemu-system-i386: command not found

sudo apt install -y qemu-system-x86

Verify:

qemu-system-i386 --version

Stop Pintos or a Running Test

Press:

Ctrl + C

If QEMU has captured the keyboard or mouse, use:

Ctrl + Alt + G

to release it.

Useful Commands

Rebuild the Threads Kernel

cd ~/cs2043/pintos/src/threads
make clean
make

Basic Pintos Boot

cd ~/cs2043/pintos/src/threads/build
pintos --

Basic Terminal-Only Boot

cd ~/cs2043/pintos/src/threads/build
pintos -v --qemu --

Run alarm-single

cd ~/cs2043/pintos/src/threads/build
pintos -v --qemu -- -q run alarm-single

Inspect the Kernel Binary

file kernel.o
file kernel.bin

Check Repository Changes

cd ~/cs2043/pintos
git status
git diff

Open the Current WSL Directory in Windows Explorer

explorer.exe .

Important Source Directories

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

Learning Path

I plan to use this repository in the following order:

Understand the Pintos directory structure

Trace the kernel boot process

Study threads/init.c

Study kernel-thread creation

Understand timer interrupts

Study thread scheduling

Study synchronization primitives

Learn kernel debugging with GDB

Move into processes and system calls

Explore virtual memory and file systems

Notes

This setup uses the normal compiler provided by Ubuntu or Debian.

Building the old GCC 6.2 Pintos cross-compiler is not required for this setup.

The -fno-pie and -fno-pic flags are used for compatibility with modern Ubuntu and Debian GCC versions.

Pintos should be stored inside the WSL Linux filesystem.

The src/utils directory is added to PATH, allowing the guide-style pintos -- command to be used.

Terminal-only QEMU mode is useful because it keeps all output inside the Ubuntu or Debian terminal.

This setup uses the JHU CS318 Pintos distribution and supports both Ubuntu and Debian under WSL2.

Credits

Pintos was originally developed at Stanford University as a teaching operating system.

This repository is based on the JHU CS318 Pintos distribution:

JHU CS318 Pintos

Original Stanford Pintos Documentation

<div align="center">

Built as part of my Operating Systems learning journey.

</div>
