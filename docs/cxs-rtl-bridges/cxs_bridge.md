# CXS Bridge Architecture

## Introduction

This document covers the microarchitecture details of the CXS Bridge. The Bridge is designed to exchange CCIX packets with maximum TLP size of 4K. The bridge architecture is primarily designed to support the link layer features of CXS protocol, while the TLP to Flit Conversion are soft coded in TLM Models. The CXS Interface is an intermediate interface between the CCIX Transaction Layer and CCIX Protocol Layer, where the Hardware Cache Coherency management happens at protocol layer which is a CHI protocol , the soft coding of which is done in through TLM Models. The link layer essentially transmits and receives transactions through well formed data called FLITs. These FLITs are programmed and sent and received from software over AXI channel into the bridge.

## Reference:

The System Cache Controller with CXS Interface has been designed with Porter-CML CXS Interface 1.1 as Reference. The CXS Bridge is proposed to be designed with same reference document.

## Features Supported:

-   Link layer features: data width(cache line size) :256,512,1024 
-    Control Width :14,36,44( it implies maximum packets from 2 to 4).
-   Number of link layer credits :15
-   Low Power Mode
-   AXI interface features: AXI4-Lite, Data Width :32, Address :32
-   Error TLP Reception Reporting

## Features not supported

-   No Replication
-   No Data Check :

## Block diagram of CXS bridge
```
           +---------------------------------------------------------+
           |                  +----------------+ +----------------+  |
           |            +---->+ REGISTERS      | |  +-----------+ |  |CXS_VALID_TX
           | +--------------+ +----------------+ |  |CXS TRANSMIT+------------->
           | |              | +---------------+  |  |&          | |  |CXS_CNTL_TX
           | |              | | +-----------+ |  |  |CREDIT LINK+-------------->
    CLK    | |              | | |TRANSMIT   | |  |  |           | |  |
   +-------> |              | | |FLIT MEM   +------>+MANAGER    | |  |CXS_DATA_TX
           | |  AXI SLAVE   +-> +-----------+ |  |  |           +-------------->
   RESETN  | |  &           | | +-----------+ |  |  |           | |  |CXS_CRDRTN_TX
  +--------> |  ADDRESS     | | |TRANSMIT   | |  |  |           +-------------->
           | |  DECODER     | | |CNTL MEM   +------>+           | |  |CXS_CRDGNT_TX
AXI4_LITE  | |              | | +-----------+ |  |  |           +<-------------+
 <-------->+ |              | |               |  |  +-----------+ |  |
           | |              | |               |  |  +-----------+ |  |CXS_ACTIVEREQ_TX
           | |              | |               |  |  |           +-------------->
           | |              | |               |  |  |LINK LAYER | |  |
           | |              | |               |  |  |           | |  |CXS_ACTIVEACK_TX
           | |              | |               |  |  |STATE      +<-------------+
           | |              | |               |  |  |           | |  |
           | |              | | +-----------+ |  |  |           | |  |
           | |              <-+ |RECEIVE    | |  |  |MACHINE    | |  |CXS_ACTIVEACK_RX
           | |              | | |FLIT MEM   | |  |  |           +------------->
           | |              | | +-----------+ |  |  |           | |  |
           | |              | | +-----------+ |  |  |           | |  |CXS_ACTIVEREQ_RX
           | +----------^---+ | |RECEIVE    | |  |  |           <--------------+
           |            |     | |CNTL MEM   | |  |  +-----------+ |  |
           |            +-----+ +-----------+ |  |                |  |
           |                  +---------------+  |  +-----------+ |  |CXS_VALID_RX
           | +-------------+         | |  |      |  |CXS RECEIVE+<----------------+
           | |             |         | |  |      |  |&          | |  |
           | | INTERRUPT   |         | |  +---------+           | |  |CXS_CNTL_RX
           | |             |         | |         |  |CREDIT LINK<-----------------+
           | |             |         | |         |  |           | |  |
           | | HANDLER     <---------+ |         |  |MANAGER    | |  |CXS_DATA_RX
           | |             |           +------------+           <------------------+
           | |             |                     |  |           | |  |
           | |             |                     |  |           | |  |CXS_CRDRTN_RX
           | |             |                     |  |           +<-----------------+
           | +-------------+                     |  |           | |  |CXS_CDRGNT_RX
           |                                     +--------------------------------->
           +---------------------------------------------------------+


```

## Hardware Block Description:

The CXS Bridge consists of the following sub-blocks:

-   Link layer state machine,
-   Credit managers for transmit and receive channels.
-   Memory for storing transmit and receive Flits and Control information related to Flits.
-   Register block for programming the CXS bridge.

The CXS Bridge transmits and receives TLPs converted to FLITs. The Flit width size ranges from 256 to 1024 bits. TLPs of sizes ranging from 4 bytes to 512 bytes are embedded within each Flit. So there can be multiple TLPs within a Flit or multiple Flits for a single TLP based on the size of TLP. Since maximum TLP size for CCIX packets is 512 bytes, the memory names and sizes are assigned accordingly. Each Flits along with it has an auxiliary Control Field consisting of Start ,End packet information along with pointers of Start ,End and Error if any within the packet. Data sent on transmit and receive channels is controlled by a link layer state machine working in tandem with the DUT side. Block diagram shows I/O signals mainly participating in CXS transactions.

### Clocks

The Bridge IP requires a single clock "clk". This clock typically is connected with XDMA IP's clk for Xilinx use case.

| clock name	 | associated interface |
|--|--|
| clk |  s_axi |
|clk | CXS  |

#### Resets
|name| associated interface |  description 
|--|--|--
|resetn  | S_AXI |Design uses only one ACTIVE LOW reset. The reset is synchronized with clk.
  resetn |CXS|Design uses only one ACTIVE LOW reset. The reset is synchronized with clk.
 |usr_resetn[usr_rst_num-1:0]||ACTIVE LOW user soft reset. This reset is generated by ANDing resetn and RESET_REG


### Port Map

Bridge top level and block level port map can be found below:

| Name | Width |I/O |description 
|--|--|--|--
|clock  |  1| input | clk
|resetn|1|input|Reset  Active Low
|usr_resetn|[usr_rst_num-1:0]|output|User Reset Active Low
|irq_out|1|output|Interrupt to XDMA
|irq_ack|1|input|Interrupt Acknowledgement from XDMA
|h2c_intr_out|[127:0]|output|Host to Card interrupt
|h2c_gpio_out|[255:0]|output|gpio output
|c2h_intr_in|[63:0]|input|card to host interrupt
|c2h_gpio_in|[255:0]|input|gpio input
|s_axi_* |--|--|AXI4-Lite Slave interface
|cxs_* |--|--|CXS Interface

### RTL Hi

### RTL Hierarchy

-   cxs_bridge_top.v
-   u_cxs_bridge (cxs_bridge.v)
    -   u_cxs_channel_if (cxs_channel_if.v)
        -   u_CXS_TX_Credit (cxs_link_credit_manager.v)
        -   u_CXS_RX_Credit (cxs_link_credit_manager.v)
        -   u_regs (cxs_register_interface.v)
            -   u_cxs_txflit_tx_mgmt (chi_txflit_mgmt.v)
            -   u_cxs_txflit_tx_ram(chi_txflit_ram.v)
            -   u_cxs_txflit_rx_ram (chi_rxflit_ram.v)
            -   u_cxs_intr_handler (chi_intr_handler)

### Hardware Block Description

#### Register Block
```
           +--------------------------------------
           |                  +------------------|
           |            +---->+ REGISTERS      |-|
           | +--------------+ +------------------|
           | |              | +---------------+ ||
           | |              | | +-----------+ | ||
    CLK    | |              | | |TRANSMIT   | | || CXS_DATA_TX
   +-------> |              | | |FLIT MEM   +-------->
           | |  AXI SLAVE   +-> +-----------+ | ||
   RESETN  | |  &           | | +-----------+ | ||
  +--------> |  ADDRESS     | | |TRANSMIT   | | || CXS_CNTL_TX
           | |  DECODER     | | |CNTL MEM   +-------->
AXI4_LITE  | |              | | +-----------+ | ||
 <-------->+ |              | |               | ||
           | |              | |               | ||
           | |              | |               | ||
           | |              | |               | ||
           | |              | |               | ||
           | |              | |               | ||
           | |              | |               | ||
           | |              | | +-----------+ | || CXS_DATA_RX
           | |              <-+ |RECEIVE    +<------+
           | |              | | |FLIT MEM   | | ||
           | |              | | +-----------+ | ||
           | |              | | +-----------+ | ||
           | +----------^---+ | |RECEIVE    | | || CXS_CNTL_RX
           |            |     | |CNTL MEM   +<-------+
           |            +-----+ +-----------+ | ||
           |                  +---------------+ ||
           | +-------------+         |          ||
           | |             |         |          ||
           | | INTERRUPT   |         |          ||
           | |             |         |          ||
           | |             |         |          ||
           | | HANDLER     <---------+          ||
           | |             |                    ||
           | |             |                    ||
           | |             |                    ||
           | |             |                    ||
           | +-------------+                    ||
           |                                    ||
           +--------------------------------------
```
The Register block implements the register space as per CXS bridge requirements. The Register block is accessible via AXI4-Lite interface. The bridge register space is mapped to one of the PCIE BASE Address Registers (BARs). The overall memory requirement for bridge is 128KB and depends mainly on size of the Transmit and Receive CXS channel memories. Below is the mention of memory size requirement per channel. The register offsets are defined for each of the registers. The register map sheet elaborates on address spaces of each of these. The Register block consists of programmable Registers, status Registers and memories for transmitting and receiving Flits to/from link layer interface. register and memory access from system side is done through AXI4-Lite interface, where axi transactions has a AXI4 Slave FSM from which address and data are passed onto the Address Decoder. Both Address Decoder and AXI4 Slave FSM are not separate sub-blocks but built in within the Register Interface block.

#### Address Decoder

The address decoder block converts the AXI4 transactions into register reads and writes. The AXI-4Lite transactions are targeted to one of the following regions of the register space.

-   General Registers: These Registers which are not part of the Data Path of CXS Bridge, but are used in Bridge Identification, External interrupts, GPIO interface.
-   Protocol Specific Registers: Protocol Specific registers: contain many configuration and status registers which pertain to datapath of CXS Bridge
-   Memories: CHI Bridge uses memories for Flit storage where Flits are sent from system side to the CXS link layer, and are read over on AXI4-Lite interface as Flits are received from CHI link layer interface. This happens only when CXS link is UP. Register Block works in close relation to Link Layer Interface block as described later. Register Interface also interfaces with Interrupt Handler block for Interrupt generation.

#### Transmit and Receive Data CXS Memory Calculation

Maximum Flit Width supported = 1024, For the given width there are 32 32-bit parallel memories that are 15 deep. Maximum TLP size is 512 Bytes, so both Transmit and Receive must at least contain, transmit and receive at any given point of time. However there can be maximum of 15 credits for 15 Flits sent/received. 15 Flits with multiple TLPs could also be transmitted/received. So

So max memory requirement for each channel must be according to credit limit and maximum Flit width = 1024*15= 15K

for Transmit and Receive Channels , the size would be 32K.

For 512 bits, the memory requirement = 16K,

for 256 bits flit width, memory requirement = 8K.

#### Transmit and Receive Control CXS Memory Calculation:

Maximum Flit Control Width = 44 for Flit width of 1024 bits. So there are 2 32 bit-parallel memories that are 15 deep for each channel.

max memory requirement for control channels = 64*15=320 bits.

  

#### Link Layer Interface
```
+---------------------------------------+
|-----------------+ +----------------+  |
|| REGISTERS      | |  +-----------+ |  |CXS_VALID_TX
|-----------------+--->+CXS TRANSMIT+------------->
|----------------+ ||  |&          | |  |CXS_CNTL_TX
|| +-----------+ | ||  |CREDIT LINK+-------------->
|| |TRANSMIT   | | ||  |           | |  |
|| |FLIT MEM   +------>+MANAGER    | |  |CXS_DATA_TX
|> +-----------+ | ||  |           +-------------->
|| +-----------+ | ||  |           | |  |CXS_CRDRTN_TX
|| |TRANSMIT   | | ||  |           +-------------->
|| |CNTL MEM   +------>+           | |  |CXS_CRDGNT_TX
|| +-----------+ | ||  |           +<-------------+
||               | ||  +-----------+ |  |
||               | ||  +-----------+ |  |CXS_ACTIVEREQ_TX
||               | ||  |           +-------------->
||               | ||  |LINK LAYER | |  |
||               | ||  |           | |  |CXS_ACTIVEACK_TX
||               | |-->+STATE      +<-------------+
||               | ||  |           | |  |
|| +-----------+ | ||  |           | |  |
|| |RECEIVE    | | ||  |MACHINE    | |  |CXS_ACTIVEACK_RX
|| |FLIT MEM   | | ||  |           +------------->
|| +-----------+ | ||  |           | |  |
|| +-----------+ | ||  |           | |  |CXS_ACTIVEREQ_RX
|| |RECEIVE    | | ||  |           <--------------+
|| |CNTL MEM   | | ||  +-----------+ |  |
|| +-----------+ | ||                |  |
+----------------+ ||  +-----------+ |  |CXS_VALID_RX
|         |  |     ||  |CXS RECEIVE+<----------------+
|         |  |     ||  |&          | |  |
|         |  +---------+           | |  |CXS_CNTL_RX
|         |        ||  |CREDIT LINK<-----------------+
|         |        ||  |           | |  |
|         |        |--^+MANAGER    | |  |CXS_DATA_RX
|         +------------+           <------------------+
|                   |  |           | |  |
|                   |  |           | |  |CXS_CRDRTN_RX
|                   |  |           +<-----------------+
|                   |  |           | |  |CXS_CDRGNT_RX
|                   +--------------------------------->
+---------------------------------------+
```
A Central Link Layer state machine keeps running in tune with DUT transmit and receive CXS signals. The Link Layer interface is tightly coupled with Register interface. The Link Layer interface implements the link layer of the CXS Layer where it has a central state machine that controls the transmit and receive channels. The state machine keeps the link state based on grant/request mechanism based on the readiness of the upper layers, once the Link comes up, Flits are exchanged based on credit exchange for Flit transfers and reception. The Credit Levels for both receive and transmit per channel are maintained in Credit Manager Block. Upto 15 credits are transmitted and received for receive and transmit channels respectively. Other auxiliary signals of Cxs layer are also controlled from Link Layer Interface. The Link Layer Interface works closely with Register interface into receiving and transmitting Flits from application layer or the software. The Block Diagram shows the Link Layer interface block.

#### Link Credit Manager

The Link Credit Manager maintains the number of credits required to transmit/receive flits on CXS channel. At a time maximum 15 credits can be sent or received, the credits are implied by virtue of pulse width of CRDGNT interface of CXS. when the link comes up, a default of 15 credits are sent on receive channel over CRDGNT output to the DUT . So the internal credit counter in receive side credit manager is 0. As Flits are received, credits are incremented in receive side for the given channel and they are decremented when software programs the ownership_flip register explained later in data flow description. On transmit side credits are received on transmit channel from DUT over CRDGNT input signal, so internal credit counter for respective channel is incremented.When OWNERSHIP_FLIP Register is programmed for a given channel, Flits are transmitted with which credits are decremented into corresponding channel credit Manager. It is incremented again when credits are received from the DUT.

More on the Link Credits management, CXS has an additional interface called CRDRTN which sends remaining credits to the other side when nothing is to be sent, remaining credits are received on Receive side. This can be achieved by programming a bit in the Register asking to send remaining credits.

#### Interrupt Handler

For propagating interrupts from x86 Host to DUT in the FPGA and vice-versa, the CHI Bridge has provision for generating interrupts. Interrupts from Host to the FPGA DUT are called Host to Card (H2C) interrupts and from FPGA DUT to x86 Host are called Card to Host (C2H) interrupts.  
The H2C interrupts generated by Host can be connected to DUT by using h2c_intr_out ports. The software driver needs to program C2H_INTR_REG to generate C2H interrupts. The DUT generated interrupts should be connected to c2h_intr_in ports. These interrupts can be translated to Legacy/MSI or MSI-X interrupts over PCIe controller to Host. The Bridge also generates interrupts to Host for indicating CXS Bridge Receive Transaction occurrences by virtue of INTR_STATUS_REG with field 0 which indicates occurrence of transaction of event on any of the Transmit or Receive Channel. Any Error in the received Flit(s) along with End of Packet is also used for raising Error Interrupt indicated by virtue of INTR_STATUS_REG with field 1 indicating the occurrence of Error. The interrupt handler block streamlines the C2H interrupts and its own transaction event interrupts and forwards to PCIe controller solution. The bridge uses “irq_out” to generate interrupts and waits for an acknowledgement “irq_ack” before sending the next interrupt. The “irq_ack” is sent by the PCIe controller after the “irq_out” translates to a PCIE Legacy interrupt Message TLP on the PCIe link. The interrupt clear and mask registers control enablement and disablement of the interrupts as per software’s discretion.Interrupt Handler is not used in Polling Mode.

  

#### Link Initialization:

The General Data Flow Description describes how Flit flows across the Bridge in detail. The FLIT width configuration is read initially to help program the FLITs for transmission and reception. Any configuration from DUT side would be known beforehand and reflected in the parameters of CXS Bridge. The BRIDGE_CONFIGURE_REG is set to ‘1’ to initiate the CXS Link Bring up process. Upon seeing the BRIDGE_CONFIGURE_REG bit set, the link layer state machine activates and eventually goes to RUN state for both transmit and receive side and collects credits on transmit channels and sends credits equal to number of RX_ALLOW_CREDITS(typ.15) from receive channel to/from the neighboring DUT for all the channels participating. The status of the link and the individual channel can be checked from CHI_BRDIGE_CHN_TX_STS_REG and CHI_BRIDGE_CHN_RX_STS_REG.

#### Receiving Flits on Bridge:

After the link up, software can receive Flits from DUT which could be any Read/Write/Snoop Request from upper layer of CCIX contained within the DUT.

The Bridge receives the Data Flit and Control Flit which is stored in the RX CXS Data Memory and RX CXS Control Memory. The Flit Valid bit is used to set the RX_OWNERSHIP bit ,side-by-side the bit in INTR_FLIT_RXN_STATUS_REG register corresponding to RX Flit received is set , the RXDAT_OWNERSHIP bit is set to ‘1’ for the count of the Flit(s) received thereafter too, and an interrupt is raised by virtue of INTR_FLIT_RXN_STATUS_REG register if INTR_FLIT_RXN_ENABLE_REG is set. At this point, the credit that is consumed for Request Flit is not sent back to DUT until the software has read the Data CXS Flit and the Control CXS Flit.Software upon seeing the interrupt clears the interrupt bit by virtue of INTR_FLIT_TXN_CLEAR_REG. Thereafter it reads the data, after which RX_OWNERSHIP_FLIP bit set by software. The FLIP bit resets the RXDAT_OWNERSHIP register for as many bits set in the FLIP register. Setting the FLIP bits on also makes the credits consumed in Flit reception be sent back to DUT.

#### Transmitting Flits on Bridge:

After the link is up, software can send Cache Coherency based requests if the Host is on remote side to downstream remote Cache Interconnect DUT or it can send response/Data based on requests coming from DUT.

Though for CXS interface there is no distinguishing element, it is sent as Flit within which TLP{s) are packaged.

Each Buffer is addressable through AXI4-Lite Bus with an offset address meant for a particular channel. Based on the number of credits available which can be checked through the CHI_BRIDGE_TXREQ_CUR_CREDITS_REG, The TLP{s) are loaded into the Transmit CXS Data Memory through AXI4-Lite bus, Along with the TLP(s), Control information related to packaged TLP(s) is also stored in Transmit CXS Control Memory. The TLP(s) in parallel access of the Transmit CXS Memory become a Flit. In order to transmit this over the CXS Transmit channel, software must set the TX_OWNERSHIP_FLIP bits in the TX_OWNERSHIP_FLIP_REG for the number of Flits to be sent. Upon sending a Flit, the status bit in INTR_FLIT_TXN_STATUS_REG. Based on the number of credits available, software can choose to send more requests. Because of the limitations of the buffer space within the credit limit of 15, non-continuous mode of transfer is enabled for both transmit and receive, which means continuous TLP(s) of bigger sizes can get staggered in receive and transmit based on credit availability. Example: if Flit Width is 256 and TLP to be transmitted is 512 bytes, which means 16 credits required, the TLP will not be sent in a single event since the max number of credits supported is 15.

So , software shall have to check the Credit value and then transmit the remaining portion, setting the TX_OWNERSHIP_FLIP bits according to the number of Flits to be sent.

#### Credit Return and Low Power from /to Bridge:

When there is no Data/Request to be sent, software can program the CREDIT_RETURN_TX_REG bit in the CXS_LOW_POWER_REG register to send back the remaining credits back to the DUT, simultaneously, DUT can also send remaining credits to the Bridge when there is no Information to be sent from its side. Thereafter, software can choose to put Bridge on Receive side and thus the DUT in Low Power by virtue of setting LOW_POWER_RX bit set to '1' in CXS_LOW_POWER_REG which essentially sends the Low power request to DUT to put its State machine in Low Power while the local Receive state machine also goes to Low Power, DEACTIVATE state.

The Transmit side link layer goes to low power only on such request from DUT through the CXS Low Power CXS_DEACT_HINT signal assertion.

It comes back to normal RUN only when normal link up procedure Flow is followed.

#### Error Reporting:

Any Error response received in the Flits will be raised as an interrupt on Interrupt Error Status Register called as INTR_ERROR_STS_REG in field ERR_0 , An Interrupt will be raised if INTR_ERROR_EN_REG bit 0 is set.

Upon receiving the interrupt, software shall clear the interrupt by setting the INTR_ERROR_CLEAR_REG bit '0'.