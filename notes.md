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
