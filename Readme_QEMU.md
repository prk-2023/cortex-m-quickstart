# Rust No embedded:

Reference: https://www.youtube.com/watch?v=_sYnzFe9A6E


- One of the biggest Roadblock for learning embedded software is the Hardware.
There is a confusing amount of options available, and non of which are free ( free in open-ness )

So getting HW to learn on is offen a hard first step to overcome.
This is where QEMU tries to fill some gaps. Which allow us to skip the hardware and learn the embedded code on your desktop.

## Goals:

Setup for no HW embedded proj: Setup and run the code without access to a development board.

- Project Setup: 
- Embedded Rust basics: 
- compiling for QEMU:
- Running with QEMU:

Setup: Start with a github repo : https://github.com/rust-embedded/cortex-m-quickstart
This is a project developed and maintained by Cortec-M team who are the maintainers of the Cortex-M crate ecosystem.

This project is a part of Rust on Embedded devices work group:
An org working to improve end-to-end experience of using Rust on resource-constrained envs and non-traditional platforms.

The group is also an official working group of Rust langauge.

For more on this checkout there news letter:   http://blog.rust-embedded.org

The above git project cortex-m-quickstart is a cargo repo that is setup with some dependencies to allowing the 
developer to quickly produce binaries that are able to run on the coretx- M0  M3 Processor.
Luckely this project also has roles in Cargo files to do QEMU emulation with those binaries if we do not have the a development
board.

Fist task is to how to use this repo and setup to do development.

Step1: Clone the repo:

$ git clone https://github.com/rust-embedded/cortex-m-quickstart.github rust-no-hardware
$ cd rust-no-hardware

Note: there is a .vscode folder that contains the template to launch configurations for debugging Cortex-M programs 
using visual studio code. For more read .vscode/README.md for more information.
You can ignore this if not using vs code.

Step2: there are few changes that are required for the project to be able to run as it currently setup.

Filename: Cargo.toml:

    Edit:
    Under [package] ==> authors = ["day"]
                        name = "day_app"  # this is the name of the proj 
          [[bin]] ==>   name = "day_app"

    With this changes we can acutally build the program for cortex arm M3 processor
    
Step3: check the main.rs to check what the program is about:

Filename: src/main.rs 

    ```
    1  #![no_std]
    2  #![no_main]
    3
    4  // pick a panicking behavior
    5  use panic_halt as _; // you can put a breakpoint on `rust_begin_unwind` to catch panics
    6
    7  // use panic_abort as _; // requires nightly
    8  // use panic_itm as _; // logs messages over ITM; requires ITM support
    9  // use panic_semihosting as _; // logs messages to the host stderr; requires a debugger
    10
    11 use cortex_m::asm;
    12 use cortex_m_rt::entry;
    13
    14 #[entry]
    15 fn main() -> ! {
    16    asm::nop(); // To not have main optimize to abort in release mode, remove when you add code
    17
    18    loop {
    19        // your code goes here
    20    }
    21  }
    ```
The above code looks different from the regular Rust app we develop on linux.
There are major changes to this structural program the current document is a walkthrough that line by line.

code analysis:

1  #![no_std]
    Normally when we build a program in Rust we use the std libraray which brings libc and other functinality 
    into scope of the project. This part is what an embedded chip can not handle. 
    The first line replaces the std library with the core library that is the light weight version of Rust backend.

2  #![no_main] 
    In a normal Rust program with std library the start flow is to look for the "main()" to start the program.
    For an embedded platfrom we will not be needing the main() function and we will take care of that.

5  use panic_halt as _; // you can put a breakpoint on `rust_begin_unwind` to catch panics 
    Setup a panic handler, when the program has a panic i.e an event that it can not handle by error handling, 
    we need to tell what function we are going to run. And this line we use the panic halt function that comes
    from the cortex-m library. 

10 use cortex_m::asm;
    dependence cortex-m-asm  ==> for assembly inline like assembly nop()
11 use cortex_m_rt::entry;
    Cortex M entry point ( this is required as we have disabled the main() function call when using std lib flow)
    since we have defined "no_main" in line 2. meaning we will handle the entry point of the board. 

14 #[entry]
    the entry point that we want to run is follows below ( fn main() )

15 fn main() -> ! {
    defintion of the main fun that is defined to run on line 14
    Note: ther return value is "!" which means the program does not return.

16    asm::nop(); // To not have main optimize to abort in release mode, remove when you add code
    we run the assembly nop to give main() some code to contain and we end with a infinite loop.
    Because in a embedded board the program does not exit and runs the code forever.
    ( as we do not have the luxary of a scheduler and all other features of a traditional operating system gives to a program)


Step4: 
    In order for the code to do some thing we can add some meaning full stuff to the infinite loop.

Step5: As its not comfortable to work with asm: we can actually change:
    11 use cortex_m::asm; 
    to 
    11 use cortex_m_semihosting; 
--- 
    Reference: https://docs.rust-embedded.org/book/start/semihosting.html

    From Rust Embedded book:
        semihosting is a mechanism that lets embedded devices do I/O on the host and is mainly used to log messaged to the 
        host console.
        semihosting requires a debug session and pretty much nothing else ( no extra wires) making it very convinent to use.
        The downside is that semihosting is slow, with each write operation taking several milliseconds depending on 
        hardware (eg: ST-link)
        cortex_m_semihosting: crate provides API to do semihosting opeartions on Cortex-M devices.
        example program:

        #![no_main]
        #![no_std]

        use panic_halt as _;

        use cortex_m_rt::entry;
        use cortex_m_semihosting::hprintln;

        #[entry]
        fn main() -> ! {
            hprintln!("Embedded Rust!! ").unwrap();
            loop {}
        }
        
        Run the above program on HW and we will see the message within the OpenOCD logs.
        $openocd
        (..)
        Embedded Rust!
        (..)

        we need to enable semihosting in OpenOCD from GDB first:
        (gdb) monitor arm semihosting enable 
        semihosting enabled

--- 
    QEMU also understrand semihosting and the above program will also work with qemu-system-arm without having to start
    a debug session.
    Note: we only have to pass the -semihosting-config flag to QEMU to enable semihosting support.
    these flags should be included in Cargo.toml 

Back to our prog: we replace cortex_m::asm with cortex_m_semihosting.
with cortex_m_semihosting we are intrested in two things: debug and hostprintline ( hprintln! macro )

    with debug : allowing us to do exit or return from the board which in embedded do not actually exista
    so when we are semihosting we are able to say that this location end the program cleanly.
    and hprintln: is host print line which tell if we are doing a semihosting env where you are hosting it in a emulated env 
    on the host forward to the host a line that we are going to print.. which is the idea of semihosting.

    so our main () will not change to 

    fn main() -> ! {
        hprintln!("embedded Rust with semihosting \n").unwrap(); 
        //unwrap() makes sure that it did not return no value

        // next we do the semihosting debug and call the exit function 
        debug::exit(debug::EXIT_SUCCESS);
        // all this does is in the event we are doing in a semi hosted env (in QEMU)
        // this will end the program here instead of going into a infinite loop.
        ..
    }



    ```
        #![no_std]
        #![no_main]

        // pick a panicking behavior
        use panic_halt as _; // you can put a breakpoint on `rust_begin_unwind` to catch panics
                             // use panic_abort as _; // requires nightly
                             // use panic_itm as _; // logs messages over ITM; requires ITM support
                             // use panic_semihosting as _; // logs messages to the host stderr; requires a debugger

        //use cortex_m::asm;
        use cortex_m_rt::entry;
        use cortex_m_semihosting::{debug, hprintln};

        #[entry]
        fn main() -> ! {
            //asm::nop(); // To not have main optimize to abort in release mode, remove when you add code

            hprintln!("---> Embedded Rust On QEMU <---\n").unwrap();

            debug::exit(debug::EXIT_SUCCESS);
            //This line will ternimate the program execution if running in QEMU env instead of going to infinite loop below

            loop {
                // your code goes here
            }
        }
    ```
Step6: ( how to build the above code ) this is a proj build for  cortex M0 processor.
so we need to make sure we have the build chain in rustup to compile down to that arch.

So install the required toolchain using rustup:

$ rustup target add thumbv7m-none-eabi
info: downloading component 'rust-std' for 'thumbv7m-none-eabi'
info: installing component 'rust-std' for 'thumbv7m-none-eabi'

Now we are ready to use cargo to build for a target as below:

$ cargo build --target thumbv7m-none-eabi
   Compiling semver-parser v0.7.0
   Compiling typenum v1.17.0
   Compiling proc-macro2 v1.0.85
   Compiling cortex-m v0.7.7
   Compiling semver v0.9.0
   Compiling version_check v0.9.4
   Compiling rustc_version v0.2.3
   Compiling unicode-ident v1.0.12
   Compiling bare-metal v0.2.5
   Compiling nb v1.1.0
   Compiling nb v0.1.3
   Compiling generic-array v0.14.7
   Compiling vcell v0.1.3
   Compiling syn v1.0.109
   Compiling void v1.0.2
   Compiling embedded-hal v0.2.7
   Compiling quote v1.0.36
   Compiling generic-array v0.13.3
   Compiling generic-array v0.12.4
   Compiling volatile-register v0.2.2
   Compiling cortex-m-rt v0.6.15
   Compiling bitfield v0.13.2
   Compiling stable_deref_trait v1.2.0
   Compiling as-slice v0.1.5
   Compiling cortex-m-semihosting v0.3.7
   Compiling cortex-m v0.6.7
   Compiling aligned v0.3.5
   Compiling r0 v0.2.2
   Compiling day_app v0.1.0 (/home/reactor/daybreak/WFH/2024/Rust/NoEmbeddedRust/rust-no-harware)
   Compiling panic-halt v0.2.0
   Compiling cortex-m-rt-macros v0.6.15
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 8.91s

Step7: 
$ cd target/thumbv7m-none-eabi/debug
ls -l day_app
-rwxr-xr-x 2 daybreak daybreak 743556 Jun  7 10:30 day_app

$ file day_app
day_app: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), statically linked, with debug_info, not stripped

The day_app is build for ARM EABI version 5 

Now we can use the regular arm build chain ( note: not the rust build-chain )
Install the following:
arm-none-eabi-binutils-cs.x86_64 : GNU Binutils for cross-compilation for arm-none-eabi target
arm-none-eabi-gcc-cs.x86_64 : GNU GCC for cross-compilation for arm-none-eabi target
arm-none-eabi-gcc-cs-c++.x86_64 : Cross Compiling GNU GCC targeted at arm-none-eabi
arm-none-eabi-newlib.noarch : C library intended for use on arm-none-eabi embedded systems

$ sudo dnf install arm-none-eabi-* 

Now dump the application using objdump to check what the dey_app is doing:

$ arm-none-eabi-objdump -d dey_app | less
day_app:     file format elf32-littlearm

Disassembly of section .text:

00000400 <Reset>:
     400:       b580            push    {r7, lr}
     402:       466f            mov     r7, sp
     404:       f000 fc8a       bl      d1c <DefaultPreInit>
     408:       f240 0000       movw    r0, #0
     40c:       f2c2 0000       movt    r0, #8192       @ 0x2000
     410:       f240 0108       movw    r1, #8
     414:       f2c2 0100       movt    r1, #8192       @ 0x2000
     418:       f000 fb55       bl      ac6 <_ZN2r08zero_bss17h135fe6805b809593E>
     41c:       f240 0000       movw    r0, #0
     420:       f2c2 0000       movt    r0, #8192       @ 0x2000
     424:       f240 0100       movw    r1, #0
     428:       f2c2 0100       movt    r1, #8192       @ 0x2000
     42c:       f641 42f8       movw    r2, #7416       @ 0x1cf8
     430:       f2c0 0200       movt    r2, #0
     434:       f000 fb76       bl      b24 <_ZN2r09init_data17h6458d8322c3901c7E>
     438:       f000 f815       bl      466 <main>

-> objdump shows the app has a "<Reset>" vector this is when the board starts up it does setting up the board cortex-m3 librbary
   <DefaultPreInit>... Once the setting up of the board is done we go to the <main> function 
    
    .......
    .......
00000466 <main>:
     466:       b580            push    {r7, lr}
     468:       466f            mov     r7, sp
     46a:       f000 f800       bl      46e <_ZN7day_app18__cortex_m_rt_main17hcf06a62a58ed3694E>

0000046e <_ZN7day_app18__cortex_m_rt_main17hcf06a62a58ed3694E>:
     46e:       b580            push    {r7, lr}
     470:       466f            mov     r7, sp
     472:       b082            sub     sp, #8
     474:       f241 6084       movw    r0, #5764       @ 0x1684
     478:       f2c0 0000       movt    r0, #0
     47c:       211a            movs    r1, #26
     47e:       f000 f943       bl      708 <_ZN20cortex_m_semihosting6export11hstdout_str17h8a0303b2cffd9640E>
     482:       f807 0c02       strb.w  r0, [r7, #-2]
     486:       f817 0c02       ldrb.w  r0, [r7, #-2]
     48a:       07c0            lsls    r0, r0, #31
     48c:       b190            cbz     r0, 4b4 <_ZN7day_app18__cortex_m_rt_main17hcf06a62a58ed3694E+0x46>
     48e:       e7ff            b.n     490 <_ZN7day_app18__cortex_m_rt_main17hcf06a62a58ed3694E+0x22>
     490:       f241 60ac       movw    r0, #5804       @ 0x16ac

-> the obj dump for main function which then called the semihosting  and hprintln .. 


Step8: Emulating our program: ( to run the program with out the board do the below hack  to cortex-m-quickstart project )
    Since our idea is to run the program on QEMU we need to change the .cargo/config.toml to the cortex-m-quickstart project
    to run on QEMU.

Filename: .cargo/config.toml
chnage the runner command under the [target.thumbv7m-none-eabi] 

In Rust runners are the once that get triggered by "cargo run"
so now we need to change the runnuner to run the program ona  QEMU system as below: uncomment the line 

runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"

Now we are ready to run the target as 
$ cargo run --target thumbv7m-none-eabi
   Compiling day_app v0.1.0 (/home/reactor/daybreak/WFH/2024/Rust/NoEmbeddedRust/rust-no-harware)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.40s
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel target/thumbv7m-none-eabi/debug/day_app`
Timer with period zero, disabling
---> Embedded Rust On QEMU <---
 

Step9: Debugging our program:

Debugging flow here explains: how to debug when some thing goes wrong.

QEMU has a nice interface to use gdb in our program:

for which we modify the runner command above to :


runner = "qemu-system-arm -cpu cortex-m3 \
        -gdb tcp::3333 -S \    
        -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"

-S : freezes QEMU when it starts up so we can connect to gdb

$ cargo run --target thumbv7m-none-eabi
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.03s
     Running `qemu-system-arm -cpu cortex-m3 -gdb 'tcp::3333' -S -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel target/thumbv7m-none-eabi/debug/day_app`
Timer with period zero, disabling

at this point the QEMU is waiting for gdb connection from host pc.

Open an other terminal now to connect to the GDB server running on the QEMU:

$ cd target/thumbv7m-none-eabi/debug/
$ gdb-multiarch ./day_app
GNU gdb (GDB) Fedora 12.1-2.fc36
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./day_app...
(gdb) target remote localhost:3333
Remote debugging using localhost:3333
warning: Remote gdbserver does not support determining executable automatically.
RHEL <=6.8 and <=7.2 versions of gdbserver do not support such automatic executable detection.
The following versions of gdbserver support it:
- Upstream version of gdbserver (unsupported) 7.10 or later
- Red Hat Developer Toolset (DTS) version of gdbserver from DTS 4.0 or later (only on x86_64)
- RHEL-7.3 versions of gdbserver (on any architecture)
cortex_m_rt::Reset () at /home/daybreak/.cargo/registry/src/index.crates.io-6f17d22bba15001f/cortex-m-rt-0.6.15/src/lib.rs:497
497     pub unsafe extern "C" fn Reset() -> ! {
(gdb) layout split  
(gdb) layout src <== view the program
(gdb) lay next   <== to toggle between code view and assembly version, register state ...
(gdb) c   <== continue the program run
Continuing.
[Inferior 1 (process 1) exited normally]
(gdb) quite
$


we can use the above method to get things stared and start debugging ... 


