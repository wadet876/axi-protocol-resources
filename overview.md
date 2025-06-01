AXI Protocol Breakdown by Wade
Source: AMBA® AXI Protocol Specification version H and K

Note: I will try to highlight the difference between AXI4 (H) and AXI5 (K) throughout. This is a need-to-know summary of the AXI protocol but I encourage you, if time permits, to use this document as a reading guide to follow along as you go through the original AXI doc.

Vocabs:
Signal: Can be single bit or multiple bit. A signal is a wire (single bit) or a group of wire
Assert: To bring active (e.g. to assert a reset is to reset)
Deassert: To deactivate (e.g. to deassert a reset is to deactivate)
Active-low/Active-high: Active when low/active when high
Drive: To generate a safety voltage

# Specification
## 1. Architecture Overview
In general, AXI has 5 channels. Each channel has signals beginning in a letter. You can treat each channel as a group of signals functioning towards a single purpose:
| AXI4  | AXI5  |
|--|--|
|Write Data (W)| Same
|Read Data (R)| Same
|Write Response (B) | Same
| Read Address (AR) | Read Request (AR) |
| Write Address (AW) | Write Request (AW) |

The general transaction remains the same across both protocols. These images shows write and read respectively. 
![](https://i.gyazo.com/dba5f9d90596b15e3ceee89cd10fbdff.png)
![](https://i.gyazo.com/926a4055f0feb68f1b0ce2c6433cb749.png)
Note: Older AXI and protocol definitions might use Master/Slave instead of Manager/Interface. Please try to use the updated terminologies. Alternatively, use Controller/Peripheral.
## Channel Definition
Each channel, or signal group, consists of information signals and a **VALID** (used by source) and **READY**(used by destination) to establish a handshake mechanism. The **LAST** signal indicates the transfer of the final data item.

### WRITE AND READ REQUEST/ADDRESS CHANNELS:
Carry required control information for a transaction (read or write)
###  WRITE DATA CHANNEL:
Carries the actual write data. Consists of:
 - Data Width, WDATA, which can be 8, 16, 32, 64, 128, 256, 512, or 1024 bits wide (DATA_WIDTH)
 - A byte lane strobe signal for every eight data bits
 
 The byte lane strobe signal (WSTRB) has one bit for each 8 bits (1 byte) of the width. So a 128 bit width would warrant a 16 bit WSTRB signal, and a 64 bit width would mean an 8bit WSTRB. The WTSRB light to 1 for every valid byte. 
![](https://i.gyazo.com/8d8bf77b859139407d08d299a151581f.png)
Note that each cell/square is a  byte. 0xFC is 11111100 and 0x3C is 00111100.

###  WRITE RESPONSE CHANNEL:
Destination uses this channel to signal the end of a **complete, full** transaction.

### Read data channel
Carries the actual read data. Consists of:
 - RDATA which can be 8, 16, 32, 64, 128, 256, 512, or 1024 bits wide (DATA_WIDTH).
 - A read response signal

## Interface and interconnect
![](https://i.gyazo.com/ca86b90cb92cc04a7d30813c8aac5913.png)
The AXI protocol provides a *single* interface definition for the interfaces between: 
- A Manager and the interconnect 
- A Subordinate and the interconnect 
- A Manager and a Subordinate
That is to say, the AXI protocol can be point-to-point (Manager AXI <-> Subordinate AXI) or bus-based (Manager AXI <-> Interconnect AXI <-> Subordinate AXI). The interconnect, or bus, can be treated as its own AXI device.
### Typical system topologies
Most systems use one of three interconnect topologies, from least to most complicated:
1. Shared request and data channels
2. Shared request channel and multiple data channels
3. Multilayer, with multiple request and data channels

In most systems, since the request channel bandwidth requirement (request channel carries control information) is much less than the data channel, topology #2 strikes a good balance of complexity and performance. 
When the request channel is shared between a manager and multiple subordinates, AWADDR/ARADDR (Write/Read Address) is used to determines which device fulfills the request.

### Register slices
Since each channel carries data in one direction (i.e., handshaking is not done on a bidirectional channel) and there is no fixed timing relationship across different channels, register slices can be easily added at any point in any channel.
Register slicing is the practice of adding registers to pipeline a transaction, allowing the designer to break up long logic chains into multiple cycles if needed.

## 2. Signal List
See the spec for signal list. Chapter A2, pg 27-35.
## 3. AXI Transport
This section details some specifics about AXI transactions. Very useful for implementation
### Clock and Reset
#### Clock 
Each AXI interface has a single clock signal, ACLK. All input signals are sampled on the rising edge of ACLK. All output signal changes can only occur after the rising edge of ACLK.

Any output should be registered before being outputted. There should be no purely combinational path between input and output.

### Reset
ARESETn is the single active-**low** reset signal for an interface. Reset can be brough active (i.e. resetting) asynchronously (combinational reset) but should only be deasserted synchronously (on posedge of ACLK).

These behaviors apply during reset:
- A Manager interface must drive AWVALID, WVALID, and ARVALID LOW.
- A Subordinate interface must drive BVALID and RVALID LOW.
- All other signals DC.

The earliest point after reset that a Manager is permitted to begin driving AWVALID, WVALID, or ARVALID HIGH is at a rising ACLK edge after ARESETn is HIGH. 
![](https://i.gyazo.com/db6ed87b349858a6f0d558864b37ef3c.png)
### Channel Handshake
**All** AXI channels use the same VALID/READY (their own signals) handshake process to transfer address, data, and control information. This mean all parties can control the rate that information flows. VALID comes from the source to indicate that address/data/control information is available. Destination generates a READY signal to indicate that it is ready to accept the info. Transfer only occurs when both VALID and READY are HIGH.

On Manager and Subordinate interfaces, there must be no combinatorial paths between input and output signals. That is to say **all data must be registered before being outputted**.

![](https://i.gyazo.com/82e06022647b2c3cb29965245e420a14.png)
The source presents information after edge 1 and asserts the VALID. The destination asserts the READY signal after edge 2. The source must keep its information stable until the transfer occurs at edge 3, when this assertion is recognized.

Some things to note:
- A source is **NOT ALLOWED** to wait until READY is asserted before asserting VALID (i.e., VALID should be asserted whenever a source can send control/data/address info)
- A destination **IS ALLOWED** to assert READY without VALID being asserted to indicate that the destination is ready to accept information whenever available.
- A source is **NOT ALLOWED** to deassert VALID after it has been asserted unless a READY is asserted. (source cannot change its mind)
- A destination can deassert READY before a VALID is asserted (i.e., a destination can change its mind)
-  VALID must stay asserted until READY is asserted and registered at a clock edge
- Handshake happens when VALID and READY are both asserted at a clock edge
- 
![](https://i.gyazo.com/7d21c007547a092596d17d07a23ff695.png)
Valid During Ready
![](https://i.gyazo.com/4d42ceb9db3c9113b3065b7a2471547a.png)
Valid after Ready

### Write and Read Channels
This section describes the AXI write and read channels. The channels are:
• A3.3.1 Write request channel (AW)
• A3.3.2 Write data channel (W)
• A3.3.3 Write response channel (B)
• A3.3.4 Read request channel (AR)
• A3.3.5 Read data channel (R)
For interfaces that use A15.3 DVM messages, there are two additional channels:
• A3.6.1 Snoop request channel (AC)
• A3.6.2 Snoop response channel (CR)
#### Write Request Channel (AW)
![](https://i.gyazo.com/3d91b2d5e628a40c5e002e736747901e.png)
The Manager can assert the AWVALID signal only when it drives a valid request. When asserted, AWVALID must remain asserted until the rising clock edge after the Subordinate asserts AWREADY

The *recommended* default state for AWREADY is HIGH. This saves a cycle since you do not need to assert AWREADY

#### Write Data Channel (W)
![](https://i.gyazo.com/25d8d9a4fc33f694945d893600de94cc.png)
Once again, VALID must stay on when information can be sent from the source and stay on until READY is asserted. 
The default state of READY can be HIGH but only if subordinate can accept data in a single cycle (i.e. no register slicing on data path).
The Manager must assert the WLAST signal while it is driving the final write transfer in the transaction.
It is recommended that WDATA is driven to zero for inactive byte lanes.
A Subordinate that does not use WLAST can omit the input from its interface.
The property WLAST_Present is used to determine if the WLAST signal is present. (this is not relevant to us but might be relevant if you are working with an IP)

#### Write Response Channel (B)
![](https://i.gyazo.com/abd9b3b71c7ae265599b88e806005a03.png)
The default state of BREADY can be HIGH, but only if the Manager can always accept a write response in a single cycle.

#### Read request channel (AR)
![](https://i.gyazo.com/84a2725684770b8da075254d01f378fc.png)
The *recommended* default state for ARREADY is HIGH. Consult the Channel Handshake section for the rules regarding READY/VALID.

#### Read data channel (R)
![](https://i.gyazo.com/68897c162cbeeb828f5d35e10baf22cd.png)
A subordinate must only assert RVALID in response to a request.
A manager uses RREADY to signals that it can accepts the data. The default state of RREADY can be high if the manager is able to accept data immediately after it signals a read transaction.
The Subordinate must assert the RLAST signal when it is driving the final read transfer in the transaction.
It is recommended that RDATA is driven to zero for inactive byte lanes.
A Manager that does not use RLAST can omit the input from its interface.
The property RLAST_Present is used to determine if the RLAST signal is present.

### Relationships between channel
The AXI protocol requires the following relationships to be maintained:
-  A write response must always follow after write transfers
- Read data and responses must always follow the read request. (0 can be used for data if read failed)
-  Channel handshakes must conform to the dependencies defined in Dependencies between channel
handshake signals.
- When a Manager issues a write request, it must be able to provide all write data for that transaction immediately.
- When a Manager has issued a write request and all write data, it must be able to accept all responses for that transaction without waiting for another transaction.
-  When a Manager has issued a read request, it must be able to accept all read data for that transaction without waiting on any other transaction.
- Note that a Manager can rely on read data returning in order from transactions that use the same ID, so the Manager only needs enough storage for read data from transactions with different IDs.
- A Manager is permitted to wait for one transaction to complete before issuing another transaction request. This is not to say a manager can issue another transaction conditional to the original transaction.
- A Subordinate is permitted to wait for one transaction to complete before accepting or issuing transfers for
another transaction.
- A Subordinate must not block acceptance of data-less write requests due to transactions with leading write
data.
The protocol does not define any other relationship between the channels.
The lack of relationship means, for example, that the write data can appear at an interface before the write request
for the transaction. This can occur if the write request channel contains more register stages than the write data
channel. Similarly, the write data might appear in the same cycle as the request.
When the interconnect (see previous section about AXI bus) is required to determine the destination address space or Subordinate space, it must realign
the request and write data. This realignment is required to assure that the write data is signaled as being valid only
to the Subordinate that it is destined for.
