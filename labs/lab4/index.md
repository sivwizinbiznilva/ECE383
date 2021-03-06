# Lab 4 - Remote Terminal

## Lab Overview

In the first portion of this lab, you will interface a number of peripherals to a simple PicoBlaze processor.  By using a USB-to-UART bridge, you will create a program that can take a command written over your computer's serial port (i.e., remote terminal) and read or write to any one of your input or output peripherals.  Specifically, you will need to control the following on your development board: LEDs and switches.

In the second part of the lab, you will implement the same logic, but use the MicroBlaze processor instead.  You will also add a VGA output peripheral. 

## System Overview

Your software will read in three-digit commands along with optional parameters.  The list of commands you must implement is provided in Table 1.  The commands are executed by your code as soon as the user finishes typing (i.e., they do not have to press "Enter").

| Command | Description |
| :-: | :-: |
| `led ##` | Write the hex value "##" to the LEDs |
| `swt` | Read in the current switch value and write to the terminal as a two hex values |

**Table 1**: List of commands that must be implemented in this laboratory assignment.  See Figure 1 for an example terminal session.

![Figure 1](figure1.jpg)

**Figure 1**: Sample terminal session that shows all the features needed for this lab.  This session set the LEDs to `0x75` and shows that the switches are currently set to the value `0x25`.

## Prelab Assignment

Create a PicoBlaze design that meets the following requirements:

1. Defines the following ports:
  1. Port `0xAF` - Read switch inputs
  2. Port `0x07` - Read push button inputs
  3. Port `0x07` - write button values to upper-four LEDs, and lower-four switch values to the lower-four LEDs.
2. Constantly read in the push buttons and lower-four bits of the switches and writes those values out to the LEDs.

Turn in a hard copy of your software and VHDL code along with simulation screenshots to demonstrate your design works correctly.

You should use the openPICIDE software to write and simulate your assembly code, as shown in class.  Make sure you have the following settings:

- Settings => Processor
  - Processor: Xilinx PicoBlaze Family
  - Processor
    - Derivate: PicoBlaze 6
    - Scratchpad size: 64
    - Memory bank count: 1
    - Memory bank size: 1024 instructions
    - Shared memory location: Low address range
    - Interrupt vector: 1023
  - Compiler
    - Entity name: custom_rom (or whatever you want to call it)
    - VHDL template file: ROM_form.vhd (from the PicoBlaze files zip)

## Required Functionality (PicoBlaze)

- Demonstrate the ability to echo a character that is sent using UART (i.e., you recieve the character with `uart_rx` and send back the same character through `uart_tx`).  Do this using the provided Xilinx UART modules.  **DO THIS FIRST**
- Demonstrate the ability to handle the `swt` command
- Demonstrate the ability to handle the `led` command

## Required Functionality (MicroBlaze)

With MicroBlaze, recreate the same functionality as in the first part of this lab.

_Note_: Never connect any of your ports to the `GCLK` signal, or you will get a PAR error and your design will not work correctly.

## A Functionality (MicroBlaze)

Add a `vga` command that allows the user to specify the background color to send to the VGA monitor.

## Lab Hints

- Define your ports as constants in your PicoBlaze and VHDL code.  This will make your code much more readable.
- You'll need the USB-UART driver - [download it here!](http://www.exar.com/common/content/default.aspx?id=10296)
  - The UART port is the microUSB connector labeled as such - it's close to the programming port on your board
  - Use the same cable you program the board with to communicate over UART
- Tera Term is freeware software that can communicate over your computer's serial ports.   [Click here to download!](http://en.sourceforge.jp/projects/ttssh2/downloads/60733/teraterm-4.82.exe/)
  - Configure the speed to Tera Term by using the "Setup" => "Serial Port" menu option
  - Select the USB-UART serial port (check your computer Device Manager if you are unsure of the port).
  - The baud rate should be 9600 for this lab

![Figure 2](figure2.jpg)

**Figure 2** - Example serial configuration for Tera Term.

- Note: Your "serial in" - i.e., where the computer is sending you data for the FPGA to read - pin is A16.  Look at the datasheet or master UCF to find the "serial out" pin location.
- To avoid crossing clock domains, all your synchronous elements should be based on the same `clk`
- Take advantage of the simulator in the openPICIDE.
- Use the Xilinx-provided `uart_tx6` and `uart_rx6` modules rather than writing your own UART controller.
  - They are available in the PicoBlaze files zip in the `UART_and_PicoTerm` folder
  - The reference manual for these modules is available in the folder as well
  - You must create a module, `clk_to_baud`, that generates an enable signal that pulses high for one clock cycle.  Based on the time between these pulses, your baud rate is 16 times that value.  For example, if you have an enable pulse once every 1 ms, your baud rate is (1 / (16 * 1ms)) = 62,500 baud.
  - You will probably need to manually create the `write_buffer` signal rather
    than using PicoBlaze's `write_strobe` signal due to timing issues.  The
same applies for the `read_buffer` signal.
- Build converter modules (e.g., `ascii_to_nibble`, `nibble_to_ascii`, etc.).
 Start small and work larger.  Here is a sample approach to this problem:
  1. Write your `clk_to_baud` module.  A good example is provided in the Xilinx UART manual.  Simulate this design to ensure it runs correctly.
  2. Ensure the UART configuration is correct on your FPGA and drivers are
installed on your computer.  Do a hardware loopback on the UART module
(`serial_out <= serial_in;`).
  3. Connect the `uart_tx6` and `uart_rx6` modules to a simple PicoBlaze program that takes the serial input (if available) and writes it to the serial output.
  4. Expand your PicoBlaze code to accept three-character commands, then go to the next line.
    - Going to the beginning of the next line requires a New Line and Carriage Return
  5. Expand your PicoBlaze code to process the `swt` command.
  6. Expand your PicoBlaze code to process the `led ##` command.
- When simulating, ensure you provide valid `serial_in` data - see the UART manual for details.  Here's a screenshot of my simulation - note the time scale:

![Simulation Screenshot](lab4_testbench.jpg)

## Free Code

```vhdl
-- useful for the led command
entity ascii_to_nibble is
    port ( ascii : in std_logic_vector(7 downto 0);
           nibble  : out std_logic_vector(3 downto 0)
         );
end ascii_to_nibble;

-- useful for the swt command
entity nibble_to_ascii is
    port ( nibble : in std_logic_vector(3 downto 0);
           ascii  : out std_logic_vector(7 downto 0)
         );
end nibble_to_ascii;

entity clk_to_baud is
    port ( clk         : in std_logic;
           reset       : in std_logic;
           baud_16x_en : out std_logic
        );
end clk_to_baud;

entity atlys_remote_terminal_pb is
    port (
             clk        : in  std_logic;
             reset      : in  std_logic;
             serial_in  : in  std_logic;
             serial_out : out std_logic;
             switch     : in  std_logic_vector(7 downto 0);
             led        : out std_logic_vector(7 downto 0)
         );
end atlys_remote_terminal_pb;
```

**Code Listing 1** - Entity templates for the lab to ensure consistency between student designs.

## Grading

| Item | Grade | Points | Out of | Date | Due |
|:-: | :-: | :-: | :-: | :-: |
| Prelab | **On-Time:** 0 ---- Check Minus ---- Check ---- Check Plus | | 10 | | BOC L23 |
| Required Functionality (PicoBlaze) | **On-Time**------------------------------------------------------------------**Late:** 1Day ---- 2Days ---- 3Days ---- 4+Days | | 30 | | COB L24 |
| Required Functionality (MicroBlaze) | **On-Time** ------------------------------------------------------------------ **Late:** 1Day ---- 2Days ---- 3Days ---- 4+Days| | 20 | | COB L28 |
| A Functionality (MicroBlaze) | **On-Time** ------------------------------------------------------------------ **Late:** 1Day ---- 2Days ---- 3Days ---- 4+Days| | 10 | | COB L28 |
| Use of Git / Github | **On-Time:** 0 ---- Check Minus ---- Check ---- Check Plus ---- **Late:** 1Day ---- 2Days ---- 3Days ---- 4+Days| | 5 | | COB L29 |
| Code Style | **On-Time:** 0 ---- Check Minus ---- Check ---- Check Plus ---- **Late:** 1Day ---- 2Days ---- 3Days ---- 4+Days| | 5 | | COB L29 |
| README | **On-Time:** 0 ---- Check Minus ---- Check ---- Check Plus ---- **Late:** 1Day ---- 2Days ---- 3Days ---- 4+Days| | 20 | | COB L29 |
| **Total** | | | **100** | | |
