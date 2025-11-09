---
description: Creating a cycle-accurate Game Boy emulator in Rust
---
## GameCrab Dev Log 0: Emulating the Game Boy's CPU
# Overview
GameCrab is an in-development emulator for the Nintendo Game Boy written in Rust. The emulator is open source and dedicated to the public domain under the CC0 license.
This post covers how I went about emulating the Game Boy's CPU. I'll discuss how the CPU works and show the code I wrote to emulate it along the way.
# Components of the Console
## The CPU
The CPU object contains two fields, a register object which holds all of the different CPU registers, and a flag which toggles whether interrupts are enabled.
```Rust
pub(crate) fn new() -> CPU {
    CPU {
        registers: Registers::new(),
        enable_interrupts: false,
    }
}
```
### Registers
The register object simply holds all of the registers which compose the CPU.
```Rust
pub(crate) struct Registers {
    pub a: u8,
    pub b: u8,
    pub c: u8,
    pub d: u8,
    pub e: u8,
    pub f: u8,
    pub h: u8,
    pub l: u8,
    pub sp: u16,
    pub pc: u16,
    pub bus: u16,
    pub x: u8,
    pub y: u8,
}
```
## The Clock
The Game Boy's CPU has a clock which ticks at a rate of approximately 4 MHz. The clock can be broken down into M cycles and T cycles.
T cycles represent a single tick of the clock, and the CPU is able to perform one command each tick. The RAM takes 4 T cycles to access, so each M cycle is comprised of 4 T cycles.
## Instructions and Commands
Within the emulator, the terms instruction and command represent two separate yet related concepts:
### Instructions
Instructions refer to the opcodes for the CPU. A byte in RAM is fetched and decoded to determine the instruction to execute.
```Rust
// Each variant of instruction enum represents a category of instructions, and each variant contains another enum which disambiguates the category into a specific instruction
pub enum Instruction {
    CB,
    Control(ControlOps),
    Load16(Ld16),
    Push(PushPop),
    Pop(PushPop),
    Load8(Ld8, Ld8),
    Arithmetic16(A16Ops),
    Arithmetic8(A8Ops),
    JumpRelative(JR),
    Jump(JP),
    Restart(u8),
    Return(Ret),
    Call(Calls),
    BitOp(BitOps),
}
```
```Rust
// For example, The Control variant of the Instruction enum takes a ControlOps enum as its field, each representing a different control instruction
// The hex numbers represent the opcodes byte for each instruction
pub enum ControlOps {
    NOP,  //0x00
    STOP, //0x10
    HALT, //0x76
    DI,   //0xF3
    EI,   //0xFB
    DAA,  //0x27
    SCF,  //0x37
    CPL,  //0x2F
    CCF,  //0x3F
}
```
### Commands
Commands are the steps performed to execute an instruction. Each command takes one clock cycle to execute, and multiple commands can be queued in order to execute an instruction. Commands come in two forms, read and update. An enum with two possible states is used to represent the commands.
#### Read
The first state is Read, which covers reading a byte between the RAM and registers. It takes two arguments, a source and destination. Each can either reference a CPU register or an address in RAM. As the names suggest, the value of the source is read and then written to the destination.
#### Update
The second state is Update, which handles updating the internal state of the console. This can come in many forms, such as performing arithmetic between CPU registers or copying a 16-bit value between register pairs. The update command takes a pointer to a function with a mutable reference to the console as an argument. This often comes in the form of a lambda expression, allowing functions with different signatures to be passed as a command.
```Rust
// The Command enum has two variants, one for read and one for update
// Read has fields for the source and destination,
// and Update takes a function with a mutable reference to the console as an argument
pub(crate) enum Command {
    Read(Source, Destination),
    Update(fn(&mut Console)),
}
```
## Fetch-Decode-Execute Loop
The CPU operates using a fetch-decode-execute loop. It begins by fetching a byte from RAM, decoding it into an instruction for the CPU, and finally executing the instruction.
### Fetch
When it is time for the CPU to run the next instruction, it starts by fetching a byte in RAM at the address specified by the Program Counter(PC). This byte is then passed to the decoder to determine what instruction should be executed.
```Rust
// The program counter points to an address in memory and the corresponding byte is fetched
// self is referencing the console
self.ram.fetch(self.cpu.get_register_16(&Register16::Pc))
```
### Decode
The byte that was fetched is then used to determine which of the 500 CPU instructions to run. Because a single byte can only represent 256 values, the instruction 0xCB is treated as a prefix, meaning that the following byte in memory will represent a different set of 256 possible instructions.
```Rust
// The decoder has two functions which are very similar.
// There is decode and decode_cb. Which function is run depends on the value of the CB flag, which is set to true when the previous command was 0xCB
// The fetched byte from RAM is passed into the function, and the decoder matches the low and high nibble of the byte in order to return the corresponding instruction
// decode_cb works in the exact same way, just with different instructions mapped to each byte
pub fn decode(byte: u8) -> Result<Instruction, String> {
    let high_nibble = byte & 0xF0;
    let low_nibble = byte & 0x0F;

    match high_nibble {
        0x0 => match low_nibble {
            0x0 => Ok(Control(NOP)),
            0x1 => Ok(Load16(BCU16)),
            0x2 => Ok(Load8(Ld8::BC, Ld8::A)),
            0x3 => Ok(Arithmetic16(A16Ops::Inc(A16Args::BC))),
            0x4 => Ok(Arithmetic8(A8Ops::Inc(A8Args::B))),
            0x5 => Ok(Arithmetic8(A8Ops::Dec(A8Args::B))),
            0x6 => Ok(Load8(Ld8::B, Ld8::U8)),
            0x7 => Ok(BitOp(RLCA)),
            0x8 => Ok(Load16(U16SP)),
            0x9 => Ok(Arithmetic16(A16Ops::Add(A16Args::BC))),
            0xA => Ok(Load8(Ld8::A, Ld8::BC)),
            0xB => Ok(Arithmetic16(A16Ops::Dec(A16Args::BC))),
            0xC => Ok(Arithmetic8(A8Ops::Inc(A8Args::C))),
            0xD => Ok(Arithmetic8(A8Ops::Dec(A8Args::C))),
            0xE => Ok(Load8(Ld8::C, Ld8::U8)),
            0xF => Ok(BitOp(RRCA)),
            _ => Err(log_error(byte)),
        },
    // The function continues on like this for all 256 values
```
### Execute
The execute step is where things start to get interesting. As discussed earlier, instructions for the Game Boy are composed of a series of commands executed over multiple clock cycles. The commands are pushed onto the execution queue and executed over the following clock cycles.
```Rust
// This function handles all 3 steps of the fetch, decode, execute loop
// self is referencing the console
pub(crate) fn fetch_decode_execute(&mut self) {
    // The 0xCB instruction sets the cb_flag to true.
    // The value of the cb_flag determines which decoder should be used when reading the byte
    let decoder = match self.cb_flag {
        true => decode_cb,
        false => decode,
    };

    // The decode methods return a Result<instruction, String> because certain bytes do not map to any instruction.
    // The fetch method returns the byte at the addres represented by the program counter, and it is passed into the appropriate decoder to return an instruction
    let instruction = match decoder(self.ram.fetch(self.cpu.get_register_16(&Register16::Pc))) {
        Ok(value) => value,
        Err(error) => panic!("{error}"),
    };

    // After the instruction has been decoded, we can set the cb_flag back to false so that we are ready for the next instruction
    self.cb_flag = false;

    // Every instruction for the Game Boy increments the program counter on the second tick, so we go ahead and schedule that here rather than duplicate it for every instruction
    self.push_command(1, Command::Update(Self::command_increment_pc));

    // Finally, we run the execute method, which takes an instruction and pushes all of the commands that compose that instruction onto the execution queue
    self.execute(instruction);
}
```

```Rust
// This method takes an instruction as input and translates them to the corresponding method to execute that instructions
// The emulator is still early in devlopment, which is why there are so many todo macros :)
// self is referencing the console
pub fn execute(&mut self, instruction: Instruction) {
    // Every instruction method returns an Option<u64>
    // This Option parameter tells the console how many ticks to wait before fetching the next instruction
    if let Some(next_instruction_offset) = match instruction {
        CB => self.instruction_cb(),
        Control(control_op) => self.instruction_control(control_op),
        Load16(ld16) => self.instruction_load16(ld16),
        Push(push_pop) => todo!("push not implemented"),
        Pop(push_pop) => todo!("pop not implemented"),
        Load8(to, from) => self.instruction_load8(to, from),
        Arithmetic16(a16_ops) => todo!("arith16 not implemented"),
        Arithmetic8(a8_ops) => todo!("arith8 not implemented"),
        JumpRelative(jr) => todo!("jumpRelative not implemented"),
        Jump(jp) => todo!("jump not implemented"),
        Restart(arg) => todo!("restart not implemented"),
        Return(ret) => todo!("return not implemented"),
        Call(calls) => todo!("call not implemented"),
        BitOp(bit_ops) => todo!("bitOp not implemented"),
    } {
        // This is where the next instruction gets queued, using the return value of the instruction methods
        self.queue_next_instruction(next_instruction_offset);
    }
}
```
## Execution Queue
The execution queue is a hash map where the key is the tick count and the value is a queue of commands. The CPU only ever queues one command at each tick count, but this approach allows other components such as the PPU(Pixel Processing Unit) to also schedule commands each tick.
Each tick, the execution queue is queried to see if any commands are scheduled for that tick. If so, those commands are executed.
```Rust
// The Hashmap takes a u64 as the key, which represents the tick counter
// The VecDeque is used as a queue, and each element of the queue is a Command object
// Each tick, the execution queue is passed the current tick count, and it pops off the queue of commands if one exists
pub(crate) struct ExecutionQueue {
    map: HashMap<u64, VecDeque<Command>>,
}
```
### Queueing Commands
Once the instruction to execute has been determined, we must queue all of the commands needed to execute the instruction. 
```Rust
// The method most instructions use is the push_command method
// This method takes a tick offset and the command to queue as arguments
// Tick offset is how many ticks in the future the command will be queued relative to the current tick count
// self is referencing the console
pub(super) fn push_command(&mut self, tick_offset: u64, command: Command) {
    // The method adds the tick offset to the current tick count and pushes the command onto the queue using the push_command_absolute method
    self.execution_queue
        .push_command_absolute(self.tick_counter + tick_offset, command);
}

// This method is responsible for actually queueing the commands
// The command is scheduled at the actual tick count rather than a relative count
// self is referencing the execution queue
pub(crate) fn push_command_absolute(&mut self, tick: u64, command: Command) {
    // First we query if a queue already exists at the specified tick
    match self.map.get_mut(&tick) {
        // If so, we push the command to the back of the queue
        Some(cmds) => {
            cmds.push_back(command);
        }
        // If not, we instantiate a new queue at the specified tick
        None => {
            self.map.insert(tick, VecDeque::from([command]));
        }
    }
}
```
# The Console Object
Now that we understand the elements of the console, it's time to create the Console object.
```Rust
// The Console is a composition of all the components discussed previously
// This allows the console to have methods which allow communication between components
pub struct Console {
    pub(crate) cpu: CPU,
    pub(crate) rom: ROM,
    pub(crate) ram: RAM,
    pub(crate) tick_counter: u64,
    pub(crate) execution_queue: ExecutionQueue,
    pub(crate) cb_flag: bool,
}
```
```Rust
// When the console is instantiated, it calls the instantiation method for each of its components
// This allows components to start off in their default states
pub fn new() -> Console {
    Console {
        cpu: CPU::new(),
        rom: ROM::new(),
        ram: RAM::new(),
        tick_counter: 0,
        execution_queue: ExecutionQueue::new(),
        cb_flag: false,
    }
}
```
```Rust
// For example, the register object, contained inside of the CPU, has default starting values that need to be set for each register
pub fn new() -> Registers {
    Registers {
        a: 0x01,
        b: 0xFF,
        c: 0x13,
        d: 0x00,
        e: 0xC1,
        f: 0x00,
        h: 0x84,
        l: 0x03,
        sp: 0xFFFE,
        pc: 0x0100,
        x: 0x0,
        y: 0x0,
        bus: 0x0100,
    }
}
```
# Putting It All Together
Now that everything has been explained, we can go step-by-step through an example to show how everything works in action. We will be fetching, decoding, and executing the instruction 0x01. This instruction is Load BC U16, which takes the two bytes of RAM following the instruction and reads the values into the BC register pair.
## Unit Test
We will start by looking at the unit test for this instruction, which starts by instantiating a new console object and populating the RAM with the data necessary to run the instruction. After that, it simulates the clock by repeatedly running the tick command. Finally we test to make sure the values stored in the registers are what we expect.
```Rust
// This function handles the creation of the console for the unit tests
fn init(memory_map: Vec<(u8, u16)>) -> Console {
    // first we instantiate a new Console object
    let mut console = Console::new();

    // The memory_map variable represents a vector of tuples where the u8 value is the byte to store in RAM, and the u16 is the address to store that byte
    for memory in memory_map {
        console.ram.set(memory.0, memory.1);
    }

    // Finally we return the console to the calling function
    console
}

#[test]
fn bc_u16() {
    // (0x01, 0x100) is the bc_u16 instruction stored at address 0x100
    // We start at address 0x100 because that is the initial value of the program counter
    // The next two tuples represent the following bytes of memory, and the values are what we will read from RAM into the register
    // The Game Boy is little endian, so the first byte following the instruction is the value to read into register C(50), and the second register reads into register B(45)
    let mut console = init(vec![(0x01, 0x100), (50, 0x101), (45, 0x102)]);

    // The Load BC U16 instruction requires 12 clock cycles to execute, so we will run the tick method 12 times 
    for n in 0..12 {
        console.tick();
    }

    // Now we test to ensure that the correct values have been read into each register
    assert_eq!(console.cpu.get_register(&Register8::B), 45);
    assert_eq!(console.cpu.get_register(&Register8::C), 50);
}
```
## The Tick Method
After initializing the console with the test data in RAM, the next step is to run the tick method. 
```Rust
// self is referencing the console
pub fn tick(&mut self) {
    // The first time the tick method is run, it calls the fetch_decode_execute instruction in order to queue the first instruction
    if self.tick_counter == 0 {
        self.fetch_decode_execute();
    }

    // Now query the execution queue at the current tick and pop off the queue if one exists
    let map_option = self.execution_queue.pop(&self.tick_counter);
    // If a command queue exists, we iterate through each command and execute it
    if let Some(queue) = map_option {
        for command in queue {
            command.execute_command(self);
        }
    }

    // Finally, we increment the tick counter so that we already for the next time the tick method is called
    self.tick_counter += 1;
}
```
## Queueing Commands
In order to execute a command, we must first queue it. Let's take a look at examples for both Read and Update commands.
### Read Commands
To handle read commands, we call the read method which reads the value from the source and writes it to the destination.
```Rust
fn read(console: &mut Console, source: &Source, destination: &Destination) {
    // First we determine if the source is a Register or address in RAM and store the value into the value variable
    let value = match source {
        Source::Register(register) => console.cpu.get_register(register),
        Source::RamFromRegister(register16) => {
            console.ram.fetch(console.cpu.get_register_16(register16))
        }
    };

    // Now we write the value variable into the appropriate register or RAM address
    match destination {
        Destination::Register(register) => console.cpu.set_register(value, register),
        Destination::RamFromRegister(register16) => console
            .ram
            .set(value, console.cpu.get_register_16(register16)),
    }
}
```
Let's take a look at an example of a read command used in the Load BC U16 instruction:
```Rust
// In this example, we are scheduling a read command 5 ticks from the current ticks
// The source of the read is the byte of RAM at the address specified by the Bus register
// The destination is the register "high," which refers to the higher register in the register pair BC, which is the C register
// self is referencing the console
self.push_command(
    5,
    Read(
        Source::RamFromRegister(Register16::Bus),
        Destination::Register(high),
    ),
);
```
### Update commands
Now let's take a look at two different update commands, one which passes a method directly, and one that uses a lambda expression. Remember that the Update command expects a function whose signature contains one argument, a mutable reference to the console object.
```Rust
// Since command_increment_pc has a signature matching that which Update commands expect, we are able to directly pass it as a reference
// self is referencing the console
self.push_command(4, Update(Self::command_increment_pc));
```
```Rust
// command_increment_pc's signature
pub(crate) fn command_increment_pc(&mut self)
```

```Rust
// Here, we want to call the method set_register_16, which takes 3 arguments, a mutable reference to the CPU, a 16 bit value, and a Register16, which is a register pair explained earlier
// Because this method does not match the signature expected by an Update command, we instead pass a lambda expression whose signature matches that expected by the Update command. Inside of that lambda expression, we are able to use the console argument passed into it to call the set_register_16 method with all of the arguments it requires.
// self is referencing the console
self.push_command(
    3,
    Update(|console: &mut Console| {
        console.cpu.set_register_16(
            console.cpu.get_register_16(&Register16::Pc),
            &Register16::Bus,
        )
    }),
);
```
```Rust
// The signature for set_register_16
// self is referencing the CPU object
pub(crate) fn set_register_16(&mut self, value: u16, register: &Register16) {
```
## Executing Commands
Now it's time to execute a command. As seen earlier, the tick method calls the execute_command method for each command queued at the current tick.
```Rust
// self is referencing the command object
pub(crate) fn execute_command(&self, console: &mut Console) {
    // First we determine what variant of command we are working with, Read or Update
    match self {
        // If the Command is of type Read, we call the read method below, passing the source and destination
        Command::Read(source, destination) => Self::read(console, source, destination),
        // If we have an Update command, we call the function it points to and pass the console as a mutable reference
        Command::Update(func) => func(console),
    }
}
```
## The Load BC U16 Instruction
Finally, we can take a look and undertand how the Load BC U16 instruction works. It's a long method, but most of it is just repeatedly queueing commands.
```Rust
// This method is used for multiple instructions which read 2 bytes from RAM into a register pair
// For this example, the register variable represents register pair BC
// self is referencing the console
fn u16_to_register(&mut self, register: Register16) -> Option<u64> {
    // This function takes a register pair and returns the 8-bit registers composing it
    // the low variable is register B, while high is register C
    let (low, high) = register16_to_register8(register);

    // We begin by pushing an Update command 3 ticks in the future
    // This command copies the value of the program counter register pair into the Bus register
    self.push_command(
        3,
        Update(|console: &mut Console| {
            console.cpu.set_register_16(
                console.cpu.get_register_16(&Register16::Pc),
                &Register16::Bus,
            )
        }),
    );

    // Next we push a command to increment the program counter by 1
    self.push_command(4, Update(Self::command_increment_pc));

    // Now we use a Read command to read the RAM at the address specified by the Bus register into the high register(C)
    // The Game Boy is little endian, which is why the first byte is the high register and the second byte is the low register
    self.push_command(
        5,
        Read(
            Source::RamFromRegister(Register16::Bus),
            Destination::Register(high),
        ),
    );

    // Now we repeat the same commands, except read into the low register(B)
    // Once again, we start by loading the program counter's value into the Bus
    self.push_command(
        6,
        Update(|console: &mut Console| {
            console.cpu.set_register_16(
                console.cpu.get_register_16(&Register16::Pc),
                &Register16::Bus,
            )
        }),
    );

    // We again increment the program counter by 1
    self.push_command(7, Update(Self::command_increment_pc));

    // Finally we read the Read the RAM address specified by the Bus into the low register(B)
    self.push_command(
        8,
        Read(
            Source::RamFromRegister(Register16::Bus),
            Destination::Register(low),
        ),
    );

    // The last step for all instructions is to return an integer specifying how many ticks the instruction takes to run
    // This allows us to schedule the next fetch, decode, and execute cycle, in this case, 12 ticks from the current tick
    Some(12)
}
```
# Conclusion
As is pretty obvious from the length of this article, a lot goes into executing an instruction in the emulator. While this approach takes a lot of setup to get running, I think it provides a pretty elegant way of implementing each instruction; you simply have to queue the commands composing the instruction using the `console.push_command` method. All of the complexity that comes with executing an instruction doesn't have to be considered when you are working on the implementation. That's a good thing too, considering the Game Boy has 500 CPU instructions!
## What's next
Compared to my previous emulation project, the CHIP-8, The Game Boy is a whole different beast. I have only partially implemented the CPU, and already the project is more than 3x the lines of code of the CHIP-8 emulator. As for what's next to do, I have to finish implementing all the remaining CPU instructions, which is much easier said than done. Thankfully, I have finished laying the groundwork and can focus solely on the instruction implementations now. I plan to post more updates as I progress with the emulator and come across new interesting challenges.
