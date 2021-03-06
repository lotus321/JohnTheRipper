## Overview

- sha512crypt for ZTEX 1.15y board allows candidate passwords up to
64 bytes, equipped with on-board mask generator and comparator.
The board computes 640 keys in parallel.


## Computing units

- sha512crypt invokes SHA512 in different ways, sometimes that look
very complex. To accomplish the task, a CPU-based computing unit
was implemented.
- Each unit consists of 4 cores, CPU, memory and I/O subsystem.
The approximate schematic of a computing unit is shown at fig.1.

```
       --------------+-------------+-------------+-------
      /             /             /             /        \
 +--------+    +--------+    +--------+    +--------+     |
 |        |    |        |    |        |    |        |     |
 | SHA512 |    | SHA512 |    | SHA512 |    | SHA512 |     |
 |  core  |    |  core  |    |  core  |    |  core  |     |
 |   #0   |    |   #1   |    |   #2   |    |   #3   |     |
 |        |    |        |    |        |    |        |     |
 +--------+    +--------+    +--------+    +--------+     |
      ^            ^            ^             ^           |
       \           |            |            /            |
        ---+-------+------------+------------             |
           |                                              |
 . . . . . | . . . . . . . . . . . . . . . . . . . . . . .| .
           |                       sha512_engine          |
           |                                              |
  +---------------+         ---------                     |
  |               |        /         \     +--------+    /
  | process_bytes |       /  "main"   \    | memory |<---
  |               |<--+--|   memory    |<--| input  |
  +---------------+   |   \ (16x256b) /    | mgr    |<-
     |      ^         |    \         /     +--------+  \
 +-------+  |        /      ---------                   |
 | procb |  |       /                            +------------+
 | saved |  |      /    +-------------------+    | unit_input |
 | state |  |     /     | thread_state(x16) |    +------------+
 +-------+  |    |      +-------------------+             ^
            |    |                                        |
  +-----------+   \   +-----------------------------+     |
  | procb_buf |    -->|          C.P.U.             |     |
  +-----------+       |  +-----------------------+  |     |
          ^           |  | integer ops. incl.    |  |     |
           \          |  | 16x16 registers(x32b) |  |     |
            \         |  +-----------------------+  |     |
             ---------|                             |     |
                      |  +----------------------+   |     |
                      |  | thread switch (x16)  |   |     |
    +--------+        |  +----------------------+   |     |
    | unit   |        |  | instruction pointers |   |     |
    | output |<-------|  +----------------------+   |     |
    | buf    |        |  | instruction memory   |   |     |
    +--------+        |  +----------------------+   |     |
        |             +-----------------------------+     |
        |                                                /
         \                                              /
          ---> to arbiter_rx             from arbiter_tx
```

- SHA512 computations are performed using specialized circuits
("cores"). Each cycle, a core computes one of 80 algorithm rounds.
Additionally 8 cycles it's busy with additions at the end of the block.
Each core runs 2 computations at the same time. On some cycle, a round
from computation #0 is computed and on the next cycle, a round from
computation #1 is computed.
Several cycles before 2 computations are finished and output,
next 2 computations start loading Initial Values (IVs) and pre-fetching
data from core's input buffer.
- internally cores store result of SHA512, allow to use previously
stored result as IVs for subsequent block. That way, the core
is able to handle input of any length.
- So each core runs 4 independent computations in parallel, performs
4 blocks in (80+8)*4 = 352 cycles.

- CPU runs same program in 16 independent hardware threads.
Each thread has 256 bytes of "main" memory (accessible by SHA512
subsystem), 16 x 32-bit registers. Data movement, integer,
execution flow control operations are available. However there's only
a subset if operations typically implemented in general-purpose CPUs,
enough for the task.
- The reference implementation of the algorithm uses modulo
operation. Since "generic" modulo appeared to be area consuming, that was
replaced with 'if A equals to X then A=0, else A=A+1' operation.
- CPU is heavily integrated with SHA512 subsystem. It has INIT_CTX,
PROCESS_BYTES, FINISH_CTX instructions that are almost equivalent to
init_ctx(), process_bytes(), finish_ctx() from software library.
Each instruction takes 1 cycle to execute.
- SHA512 instructions store instruction data in internal buffer
(procb_buf). A dedicated unit (process_bytes) takes intermediate data,
fetches input data from the memory, creates 16 x 64-bit data blocks
for cores, adds padding and total where necessary. It saves
the state of an unfinished computation, switches to the next core
after each block.
- The program for the CPU is available <a href='https://github.com/magnumripper/JohnTheRipper/blob/bleeding-jumbo/src/ztex/fpga-sha512crypt/sha512crypt/sha512unit/cpu/program.vh'>here</a>.


## Design overview

```
       ------------------+--------+--+--+--------+---------
      /                 /        /  /  /        /          \
 +-----------+   +-----------+             +-----------+    |
 |           |   |           |   X  X  X   |           |    |
 | Computing |   | Computing |             | Computing |    |
 |  Unit #0  |   |  Unit #1  |             |  Unit N   |    |
 |           |   |           |   X  X  X   |           |    |
 +-----------+   +-----------+             +-----------+    |
       ^               ^         ^  ^  ^         ^          |
       |               |         |  |  |        /           |
       +---------------+---------+--+--+--------            |
       |                                                    |
 . . . | . . . . . . . . . . . . . . . . . . . . . . . . . .| .
       |                                                    |
       |                                                    |
  +-----------------+          +----------------+          /
  |     Arbiter     |          |     Arbiter    |<---------
  | (transmit part) |<-------->| (receive part) |
  +-----------------+          +----------------+
       ^                                    |
       |                    +------------+  |  +------
       |      +---------+   |            |<-+->| mode \
       |      | cmp.    |-->| comparator |     | cmp   |--
       |      | config. |   |            |---->| ?    /   \
       |      +---------+   +------------+     +------     |
       |          ^                                        |
 . . . | . . . . .|. . . . . . . . . . . . . . . . . . . . | .
       |          |      Communication framework           |
       |          |                                       /
  +-----------+   |                                      /
  | candidate |   |                                     /
  | generator |   |                                    /
  +-----------+---------+----------------------+      /
  | input pkt. handling | output pkt. creation |<-----
  +---------------------+----------------------+
  | input FIFO          | output FIFO          |
  +---------------------+----------------------+
  |  Prog. clocks  | USB I/O   |  FPGA reset   |
  +--------------------------------------------+

```
fig.2. Overview, FPGA application

- Each FPGA has 10 computing units, that's 40 cores, 160 keys are
computed in parallel. 
- On some hashes, dependent on key length and salt length,
performance degradation up to 10% is observed. The cause is
not-so-effective iteraction between CPU and process_bytes unit.
- Communication framework was mostly taken from previous descrypt-ztex
and bcrypt-ztex projects. The only difference is variable-length
candidate generator (bcrypt and descrypt have fixed-length inputs).


## Design Placement and Routing

- Attention was paid for optimal placement of individual components.
Available area was manually divided into 51 equal rectangles. Each unit
occupies 5 rectangles: 4 for cores and one for the CPU and the rest.


## How to run simulation using ISIM from ISE Design Suite

- Make sure you have ```define SIMULATION``` in sha512.vh uncommented.
- For behavioral simulation of 1 of 10 units, run <a href='https://github.com/magnumripper/JohnTheRipper/blob/bleeding-jumbo/src/ztex/fpga-sha512crypt/sha512crypt/sha512unit/sha512unit_test.v'>sha512unit_test</a>.
Uncomment/add what you're testing. The result of the 1st computation
appears in the Unit's Output Buffer (unit_output_buf) and in rows 48-63 of
"main" memory (sha512unit.engine.mem_main.inst.native_mem_module.memory).
- For simulation of full design with data as arrives from USB controller,
use <a href='https://github.com/magnumripper/JohnTheRipper/blob/bleeding-jumbo/src/ztex/fpga-sha512crypt/sha512crypt/sha512crypt_test.v'>sha512crypt_test</a>.
Output packets (defineed in <a href='https://github.com/magnumripper/JohnTheRipper/blob/bleeding-jumbo/src/ztex/pkt_comm/inpkt.h'>inpkt.h</a>) appear in
output_fifo.fifo_output0.ram exactly as before they leave FPGA.

