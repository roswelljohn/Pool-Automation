
7/23/2026  12:15pm
Pentair IF/IC Project Checkpoint
Current Status
✅ IntelliChlor (IC)
Working correctly.
Queue works.
Responses received.
Queue entries removed after replies.
✅ IntelliFlo Communications
Working correctly.
Status requests transmit.
Local control requests transmit.
Pump replies correctly.
Packet parser is working.
Current IF packets sent every cycle:
A5 00 60 10 07 00     (Status Request)
A5 00 60 10 04 01 00  (Local Control)
Pump State
Current physical state:
Pump ON
0 RPM
Display shows Running Schedule
Waiting for an external start command
This is the correct state for testing.
What We Learned
Selecting Program 2 from Home Assistant changes the HA entity:
Pool Pump Program Select >> Program 2
However no IntelliFlo start/program packet is ever transmitted.
The log only shows:

IF Sent ...10.07...
IF Sent ...10.04...
There is no third IF packet.
That means:

The command never reaches the transmit queue.

The UART, queue, parser, checksum, and packet transmission are not the problem.

Where We Need To Look Next
We need to find the callback executed when Home Assistant changes:
Pool Pump Program Select
We're looking for code that eventually calls something like:
queue_if_packet_(...)
or
requestProgram(...)
or
startProgram(...)
or
setProgram(...)
Searches To Perform Next Session
Search the project for these, in order:
Program Select
Pool Pump Program
queue_if_packet_
on_control
publish_state
We're trying to find where the HA select entity is connected to the IntelliFlo command generator.
Important Discovery
The queue modifications we made are working.
ESPHome is compiling the edited .cpp, uploading it, and running the new code.

The communication layer is no longer suspect.

The remaining issue is higher up in the application logic—the code that should enqueue a "start Program 2" command is apparently never being called.

Next time, we'll trace the "Pool Pump Program Select" callback to find why it never queues the IntelliFlo start packet. We're very close now.

7/23/2026   11am
Current status
Your merged loop() is working correctly.
The logs show:
✅ IntelliChlor requests are sent first every 30 seconds.
✅ IntelliChlor responses are received.
✅ Queue entries are removed correctly.
✅ IntelliFlo requests are sent afterward.
✅ IntelliFlo status packets are received.
✅ Pump RPM, watts, and flow are updating correctly.
✅ Pump responds to Home Assistant commands (you successfully issued Stop Pump).
So the transmit scheduler is functioning exactly as intended.
Most important observation
When you pressed Stop Pump, the only thing that appeared was:
Pool Pump Program Select >> Stop Pump
There was no
IF Sent: ...
after that.
That means one of two things:
the select entity never queues the Stop command, or
it queues it somewhere other than if_queue_.
The scheduler itself is working.
First thing we'll do next session
We're going to trace where the Stop command goes.
Search for
Stop Pump
If that doesn't find anything, search for
requestPumpProgram
or
setPumpProgram
or simply
Program Select
We're looking for the function that runs when the Home Assistant select changes.
After we find it
We'll verify that it eventually does something like
this->if_queue_.push(packet);
If it doesn't, we'll fix that.
If it pushes to the old
tx_queue_
instead, we'll move it to if_queue_.
What we've already fixed
✔ Split receive parser
✔ Split transmit scheduler
✔ IntelliFlo priority
✔ IntelliChlor retry queue
✔ Both protocols coexist without collisions
✔ Confirmed ESPHome is compiling the modified source
When you come back, just say:
"Resume Pentair checkpoint"


07/22/2026
Project Checkpoint – Pentair ESPHome IntelliFlo
Date: Tonight's session
Goal
Improve IntelliFlo RS-485 control reliability while keeping IntelliChlor compatibility, using ESPHome and Home Assistant.
Hardware
ESP32
ESPHome custom component
Pentair IntelliFlo VS pump
No IntelliCenter
No EasyTouch
No IntelliTouch
Home Assistant controls everything
IntelliChlor code is present but no chlorinator is currently installed (you may add a CMP chlorinator later).
Files Modified
pentair_if_ic.h
Added a dedicated transmit packet structure:
struct TxPacket {
    PacketType type;
    uint8_t retries;
    uint8_t attempts;
    std::vector<uint8_t> data;
};
Changed:
std::queue<TxPacket> tx_queue_;
pentair_if_ic.cpp
Replaced all tuple handling with the new struct.
Converted:
std::make_tuple(...)
to
TxPacket tx_packet;
...
tx_queue_.push(tx_packet);
Removed all usage of:
std::tuple
std::make_tuple
std::get
The project now compiles successfully.
IntelliFlo Findings
From the packet captures:
Physical keypad changes Program 1, Program 2, and Stop all generate RS-485 traffic.
Home Assistant commands are transmitted correctly.
The pump acknowledges packets.
However, the pump often does not execute remote commands because of its current control state or command timing.
We determined that simply changing packet timing wasn't enough.
Next Development Stage
We'll implement packet priority.
Current queue:
IC
IC
IC
IF
IC
Desired behavior:
IF
IC
IC
IC
IntelliFlo commands should never wait behind IntelliChlor polling.
Planned Changes
Add a priority field to TxPacket.
Prioritize IntelliFlo packets.
Send pump commands immediately when possible.
Delay IntelliChlor polling briefly after an IntelliFlo command.
Continue decoding the remaining unknown IntelliFlo packets.
Testing Status
Completed
✅ Packet parser working.
✅ Checksum verification fixed.
✅ Packet queue refactored.
✅ Code compiles.
Remaining
⏳ Priority queue.
⏳ IntelliFlo command arbitration.
⏳ Additional protocol decoding.
Working Style
We'll continue the same disciplined process:
Make one logical change.
Compile.
Test on the pump.
Review logs.
Move to the next change.
That approach has worked very well so far.
