# FreeRTOS WiFi / Network Stack: Packet Processing Analysis

This document provides a detailed analysis of the WiFi and network stack in FreeRTOS, focusing on the RX/TX packet processing flow through FreeRTOS-Plus-TCP, the network interface driver layer, and WiFi-specific implementations.

## 1. Network Stack Architecture Overview

FreeRTOS does not include a built-in WiFi driver. Instead, it provides **FreeRTOS-Plus-TCP** (a complete TCP/IP stack) and defines a clean **Network Interface Driver API**. Hardware vendors implement this API for their specific WiFi/Ethernet chips. The architecture is layered:

```
┌──────────────────────────────────────────────────────┐
│  Application (uses Berkeley Sockets API)              │
│  FreeRTOS_send() / FreeRTOS_recv() / FreeRTOS_connect() │
├──────────────────────────────────────────────────────┤
│  FreeRTOS-Plus-TCP                                    │
│  ┌──────────┐ ┌──────────┐ ┌──────┐ ┌──────┐        │
│  │ TCP/UDP  │ │  DHCP    │ │ DNS  │ │ ARP  │        │
│  │ Sockets  │ │  Client  │ │      │ │/ND   │        │
│  └────┬─────┘ └────┬─────┘ └──┬───┘ └──┬───┘        │
│       └────────────┼──────────┼────────┘             │
│               ┌────▼─────────▼────┐                  │
│               │   IP-Task         │                  │
│               │ (event loop)      │                  │
│               └────────┬──────────┘                  │
├────────────────────────┼─────────────────────────────┤
│  Network Buffer Management                           │
│  (BufferAllocation_1 or BufferAllocation_2)          │
├────────────────────────┼─────────────────────────────┤
│  Network Interface Driver API                         │
│  pfInitialise / pfOutput / pfGetPhyLinkStatus        │
├────────────────────────┼─────────────────────────────┤
│  Hardware-Specific Driver (WiFi/Ethernet)             │
│  ESP32 WiFi / PIC32 WILC1000 / STM32 EMAC / etc.     │
├──────────────────────────────────────────────────────┤
│  WiFi Hardware (ESP32 / WILC1000 / etc.)              │
└──────────────────────────────────────────────────────┘
```

## 2. Network Interface Driver API

Defined in `FreeRTOS_Routing.h`, each network interface is described by `NetworkInterface_t`:

```c
typedef struct xNetworkInterface {
    const char * pcName;
    void * pvArgument;
    NetworkInterfaceInitialiseFunction_t  pfInitialise;    // Init hardware
    NetworkInterfaceOutputFunction_t      pfOutput;        // Send packet to wire
    GetPhyLinkStatusFunction_t            pfGetPhyLinkStatus; // Check link
    NetworkInterfaceMACFilterFunction_t   pfAddAllowedMAC;   // Add MAC filter
    NetworkInterfaceMACFilterFunction_t   pfRemoveAllowedMAC;// Remove MAC filter
    struct xNetworkEndPoint * pxEndPoint;  // Endpoints bound to this interface
    struct xNetworkInterface * pxNext;     // Next interface (linked list)
} NetworkInterface_t;
```

**Three functions a driver must implement:**

1. **`pfInitialise(pxInterface)`** — Initialize hardware, set MAC address, enable receiver. Returns `pdTRUE` on success. Called repeatedly until it succeeds.

2. **`pfOutput(pxInterface, pxNetworkBuffer, xReleaseAfterSend)`** — Transmit a packet. The Ethernet frame is in `pxNetworkBuffer->pucEthernetBuffer` with length `pxNetworkBuffer->xDataLength`. If `xReleaseAfterSend == pdTRUE`, driver must call `vReleaseNetworkBufferAndDescriptor()` after sending.

3. **`pfGetPhyLinkStatus(pxInterface)`** — Return `pdTRUE` if PHY link is up.

**To deliver received packets**, the driver calls into the stack:
```c
IPStackEvent_t xRxEvent = { eNetworkRxEvent, pxNetworkBuffer };
xSendEventStructToIPTask(&xRxEvent, xBlockTime);
```

### Available Driver Implementations

Located in `FreeRTOS-Plus/Source/FreeRTOS-Plus-TCP/source/portable/NetworkInterface/`:

| Driver | Target | Type |
|--------|--------|------|
| `esp32/` | ESP32 (ESP-IDF) | **WiFi** (via `esp_wifi_internal_tx`/`wlanif_input`) |
| `pic32mzef/NetworkInterface_wifi.c` | PIC32MZ + WILC1000 | **WiFi** (via `WDRV_EXT_DataSend`) |
| `STM32/` | STM32F/H (ETH MAC + PHY) | Ethernet |
| `DriverSAM/` | ATSAM4E/SAME5x (GMAC) | Ethernet |
| `Zynq/` | Xilinx Zynq (GEM) | Ethernet |
| `LPC17xx/`, `LPC18xx/`, `LPC54018/` | NXP LPC | Ethernet |
| `NXP1060/` | NXP i.MX RT1060 (ENET) | Ethernet |
| `linux/` | Linux (pcap) | Simulation |
| `WinPCap/` | Windows (WinPcap) | Simulation |
| `MPS2_AN385/`, `MPS3_AN552/` | ARM FPGA boards | QEMU simulation |
| `ksz8851snl/` | KSZ8851SNL SPI Ethernet | Ethernet (SPI) |
| `M487/` | Nuvoton M487 (EMAC) | Ethernet |

## 3. RX Packet Processing Flow (Hardware → Application)

The complete path a packet takes from WiFi/Ethernet hardware to the application socket:

```
WiFi/Ethernet Hardware
    │ (DMA or interrupt delivers raw frame)
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Hardware Interrupt / DMA Callback                    │
│   - WiFi: wlanif_input() (ESP32) or WiFi callback (PIC32)   │
│   - Ethernet: EMAC IRQ → read descriptor → get frame        │
│                                                              │
│   Actions:                                                   │
│   a) Allocate NetworkBufferDescriptor_t via                  │
│      pxGetNetworkBufferWithDescriptor(len, timeout)          │
│   b) Copy frame data into pucEthernetBuffer                  │
│   c) Set pxInterface and pxEndPoint on the descriptor        │
│   d) Enqueue to IP-task:                                     │
│      IPStackEvent_t xRxEvent = { eNetworkRxEvent, buf };    │
│      xSendEventStructToIPTask(&xRxEvent, timeout)            │
└──────────────────────┬──────────────────────────────────────┘
                       │ (FreeRTOS queue)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: IP-Task Event Loop                                   │
│   prvIPTask() → prvProcessIPEventsAndTimers()               │
│   xQueueReceive(xNetworkEventQueue, &xReceivedEvent, ...)   │
│                                                              │
│   case eNetworkRxEvent:                                      │
│     prvHandleEthernetPacket(pxNetworkBuffer)                 │
│       → prvProcessEthernetPacket(pxNetworkBuffer)            │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Ethernet Frame Dispatch                              │
│   prvProcessEthernetPacket() (FreeRTOS_IP.c:1680)           │
│                                                              │
│   Parse EthernetHeader_t:                                    │
│   switch(pxEthernetHeader->usFrameType)                      │
│     case ipARP_FRAME_TYPE (0x0806):                          │
│       → eARPProcessPacket(pxNetworkBuffer)                   │
│     case ipIPv4_FRAME_TYPE (0x0800):                         │
│     case ipIPv6_FRAME_TYPE (0x86DD):                         │
│       → prvProcessIPPacket(pxIPPacket, pxNetworkBuffer)     │
│                                                              │
│   Result → eFrameProcessingResult_t:                         │
│     eReleaseBuffer: drop, free buffer                        │
│     eReturnEthernetFrame: send back (ARP reply, ICMP echo)   │
│     eFrameConsumed: passed to upper layer, buffer owned      │
│     eWaitingResolution: set aside for ARP/ND resolution      │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 4: IP Packet Processing                                 │
│   prvProcessIPPacket() (FreeRTOS_IP.c:1976)                 │
│                                                              │
│   For IPv4:                                                  │
│     prvAllowIPPacketIPv4() — validate header, check dest IP │
│     prvCheckIP4HeaderOptions() — handle IP options           │
│                                                              │
│   For IPv6:                                                  │
│     prvAllowIPPacketIPv6() — validate, check dest            │
│     eHandleIPv6ExtensionHeaders() — extension header chain   │
│                                                              │
│   Update ARP/ND cache from source address                    │
│                                                              │
│   Dispatch by protocol (ucProtocol):                         │
│     ipPROTOCOL_ICMP (1) → process ICMP (ping reply, etc.)    │
│     ipPROTOCOL_UDP (17) → prvProcessUDPPacket()              │
│     ipPROTOCOL_TCP (6)  → xProcessReceivedTCPPacket()       │
└──────────┬────────────────────────────┬──────────────────────┘
           ▼                            ▼
┌──────────────────────┐  ┌──────────────────────────────────────┐
│ Step 5a: UDP Path    │  │ Step 5b: TCP Path                    │
│                      │  │                                      │
│ prvProcessUDPPacket()│  │ xProcessReceivedTCPPacket()          │
│ (FreeRTOS_IP.c:1871) │  │ (FreeRTOS_TCP_IP.c)                  │
│                      │  │                                      │
│ Validate UDP header  │  │ Validate TCP header, checksum        │
│ Check checksum       │  │ Find socket by (addr, port) tuple    │
│ xProcessReceived     │  │ Handle TCP state machine:            │
│   UDPPacket()        │  │   SYN → new connection               │
│ (FreeRTOS_UDP_IP.c)  │  │   ACK → advance window              │
│                      │  │   DATA → add to rxStream             │
│ Find socket by       │  │   FIN → close connection             │
│   destination port   │  │                                      │
│ Copy payload to      │  │ Data written to socket's rxStream    │
│   socket's receive   │  │   (circular buffer via               │
│   buffer or queue    │  │    uxStreamBufferAdd())              │
│ Wake receiving task  │  │ Wake receiving task via               │
│                      │  │   vSocketWakeUpUser() /              │
│                      │  │   task notification or event group   │
└──────────┬───────────┘  └──────────────┬───────────────────────┘
           ▼                             ▼
┌──────────────────────────────────────────────────────────────┐
│ Step 6: Application Receives Data                             │
│                                                               │
│ For UDP:                                                      │
│   FreeRTOS_recvfrom(xSocket, buf, len, flags)                │
│   → Reads from socket's receive buffer/queue                  │
│                                                               │
│ For TCP:                                                      │
│   FreeRTOS_recv(xSocket, buf, len, flags)                    │
│   → lTCPAddRxdata() copies from rxStream to application buf  │
│   → Returns number of bytes read                              │
│   → Blocks on event group if no data available                │
└──────────────────────────────────────────────────────────────┘
```

### ESP32 WiFi RX Example

The ESP32 driver in `portable/NetworkInterface/esp32/NetworkInterface.c` demonstrates a real WiFi RX path:

```c
// Called by ESP-IDF WiFi stack when a frame is received
esp_err_t wlanif_input(void *netif, void *buffer, uint16_t len, void *eb)
{
    // 1. Check if frame should be processed
    if (eConsiderFrameForProcessing(buffer) != eProcessBuffer) {
        esp_wifi_internal_free_rx_buffer(eb);
        return ESP_OK;
    }

    // 2. Allocate network buffer descriptor
    pxNetworkBuffer = pxGetNetworkBufferWithDescriptor(len, timeout);

    // 3. Copy frame data
    memcpy(pxNetworkBuffer->pucEthernetBuffer, buffer, len);
    pxNetworkBuffer->xDataLength = len;
    pxNetworkBuffer->pxInterface = pxMyInterface;
    pxNetworkBuffer->pxEndPoint = FreeRTOS_MatchingEndpoint(pxMyInterface, buffer);

    // 4. Send to IP-task via event queue
    IPStackEvent_t xRxEvent = { eNetworkRxEvent, pxNetworkBuffer };
    xSendEventStructToIPTask(&xRxEvent, timeout);

    // 5. Free ESP-IDF's buffer (our copy is in network buffer)
    esp_wifi_internal_free_rx_buffer(eb);
}
```

## 4. TX Packet Processing Flow (Application → Hardware)

The path data takes from the application calling `FreeRTOS_send()` to the WiFi/Ethernet hardware transmitting the frame:

```
┌──────────────────────────────────────────────────────────────┐
│ Step 1: Application Sends Data                                │
│                                                               │
│ TCP: FreeRTOS_send(xSocket, pvBuffer, uxDataLength, flags)   │
│ UDP: FreeRTOS_sendto(xSocket, pvBuffer, uxDataLength, ...)  │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ Step 2: Socket Layer                                          │
│                                                               │
│ TCP path (FreeRTOS_Sockets.c):                                │
│   FreeRTOS_send()                                             │
│     → prvTCPSendCheck() — verify socket state, connection    │
│     → prvTCPSendLoop() — copy data to socket's txStream      │
│       while(bytes remaining):                                 │
│         uxStreamBufferAdd(pxSocket->u.xTCP.txStream, data)  │
│         pxSocket->u.xTCP.usTimeout = 1                       │
│         xSendEventToIPTask(eTCPTimerEvent)                   │
│                                                               │
│ UDP path (FreeRTOS_Sockets.c):                                │
│   FreeRTOS_sendto()                                           │
│     → prvSendUDPPacket()                                      │
│       Allocate NetworkBufferDescriptor_t                      │
│       Fill in UDP header, IP header, Ethernet header          │
│       → prvSendTo_ActualSend()                                │
│         xSendEventStructToIPTask({eStackTxEvent, buffer})    │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ Step 3: IP-Task Processes TX Event                            │
│                                                               │
│ TCP: eTCPTimerEvent triggers                                  │
│   vCheckNetworkTimers() → xTCPSocketCheck(pxSocket)          │
│     → prvTCPSendRepeated() (FreeRTOS_TCP_Transmission.c)     │
│       Get data from txStream                                  │
│       Build TCP segment with sequence numbers, window, etc.  │
│       → prvTCPReturnPacket() — fills headers, checksum       │
│         Calls prvForwardTxPacket(pxNetworkBuffer, pdTRUE)   │
│                                                               │
│ UDP: eStackTxEvent triggers                                   │
│   vProcessGeneratedUDPPacket(pxNetworkBuffer)                 │
│     Fill in IP header, checksum                               │
│     → prvForwardTxPacket(pxNetworkBuffer, pdTRUE)            │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ Step 4: Forward to Network Driver                             │
│                                                               │
│ prvForwardTxPacket() (FreeRTOS_IP.c:741)                     │
│   pxNetworkBuffer->pxInterface->pfOutput(                     │
│       pxInterface, pxNetworkBuffer, xReleaseAfterSend)       │
│                                                               │
│ This calls the hardware-specific output function.             │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ Step 5: Hardware Driver Transmits                             │
│                                                               │
│ ESP32 WiFi:                                                   │
│   xESP32_Eth_NetworkInterfaceOutput()                        │
│     esp_wifi_internal_tx(ESP_IF_WIFI_STA,                    │
│       pxNetworkBuffer->pucEthernetBuffer,                    │
│       pxNetworkBuffer->xDataLength)                          │
│     if (xReleaseAfterSend)                                    │
│       vReleaseNetworkBufferAndDescriptor(pxNetworkBuffer)    │
│                                                               │
│ PIC32 WILC1000 WiFi:                                          │
│   xNetworkInterfaceOutput()                                   │
│     WDRV_EXT_DataSend(xDataLength, pucEthernetBuffer)        │
│     if (xReleaseAfterSend)                                    │
│       vReleaseNetworkBufferAndDescriptor(pxDescriptor)       │
│                                                               │
│ STM32 Ethernet:                                               │
│   xNetworkInterfaceOutput()                                   │
│     Copy to DMA TX descriptor                                 │
│     Start DMA transfer                                        │
│     Release buffer after DMA complete interrupt               │
│                                                               │
│ All drivers: release the NetworkBufferDescriptor via          │
│ vReleaseNetworkBufferAndDescriptor() when xReleaseAfterSend   │
│ is pdTRUE, returning it to the free buffer pool.              │
└──────────────────────────────────────────────────────────────┘
```

### ARP Resolution During TX

Before sending an IP packet, the destination MAC must be known:

```
prvProcessIPPacket / xProcessReceivedUDPPacket
  → xCheckRequiresResolution(pxNetworkBuffer)
    → If MAC unknown:
        eReturned = eWaitingResolution
        Buffer stored in pxARPWaitingNetworkBuffer
        ARP request sent (IPv4) or Neighbor Solicitation (IPv6)
        Timer started (ipARP_RESOLUTION_MAX_DELAY = 2000ms)
    → When ARP reply/NA received:
        eARPProcessPacket() / vNDAgeCache() resolves MAC
        pxARPWaitingNetworkBuffer is sent with resolved MAC
```

## 5. Network Buffer Management

All packet data flows through `NetworkBufferDescriptor_t`:

```c
typedef struct xNETWORK_BUFFER {
    ListItem_t xBufferListItem;             // Free list linkage
    IP_Address_t xIPAddress;                // Source/dest IP
    uint8_t * pucEthernetBuffer;            // Pointer to Ethernet frame
    size_t xDataLength;                     // Frame length
    struct xNetworkInterface * pxInterface; // Receiving/sending interface
    struct xNetworkEndPoint * pxEndPoint;   // Associated endpoint
    uint16_t usPort;                        // Source/dest port
    uint16_t usBoundPort;                   // Bound port for TX
} NetworkBufferDescriptor_t;
```

**Buffer layout** in memory:
```
+---------+-----------+---------+-------------------+
| pointer | filler[6] | ETH HDR | IP + TCP/UDP data |
| (8B)    | (+2 align)| (14B)   |                   |
+---------+-----------+---------+-------------------+
^                     ^                            ^
pucEthernetBuffer     Dest MAC                     IP header (word-aligned)
```

The `ipBUFFER_PADDING` (8 or 12 bytes depending on architecture) ensures the IP header is word-aligned for efficient processing.

**Two allocation schemes:**

| | BufferAllocation_1 | BufferAllocation_2 |
|---|---|---|
| **Size** | Fixed (`ipTOTAL_ETHERNET_FRAME_SIZE`) | Variable (requested size) |
| **Pool** | Static array + pre-allocated RAM | Dynamic `pvPortMalloc()` |
| **Fragmentation** | None | Possible (needs heap_4/5) |
| **ISR-safe** | Yes (with threshold) | Partially |
| **Resize** | No (`xBufferAllocFixedSize = pdTRUE`) | Yes (`xBufferAllocFixedSize = pdFALSE`) |

## 6. WiFi Driver Details

### 6.1 ESP32 WiFi (esp32/NetworkInterface.c)

The most complete WiFi driver implementation. Works with ESP-IDF's WiFi stack:

**Initialization:**
```c
xESP32_Eth_NetworkInterfaceInitialise()
    esp_wifi_get_mac(ESP_IF_WIFI_STA, ucMACAddress)
    FreeRTOS_UpdateMACAddress(ucMACAddress)
```

WiFi connection is managed externally by the ESP-IDF `esp_wifi` API. The driver is notified of link status changes:
- `vNetworkNotifyIFUp()` — Sets `xInterfaceState = INTERFACE_UP`
- `vNetworkNotifyIFDown()` — Sends `eNetworkDownEvent` to IP-task

**TX:** Calls `esp_wifi_internal_tx(ESP_IF_WIFI_STA, buffer, length)` — direct handoff to ESP-IDF WiFi MAC layer.

**RX:** `wlanif_input()` is registered as callback with ESP-IDF. ESP-IDF calls it with each received 802.11 frame (already converted to Ethernet format by the WiFi layer). The function copies data into a `NetworkBufferDescriptor_t` and enqueues `eNetworkRxEvent`.

### 6.2 PIC32MZ + WILC1000 WiFi (pic32mzef/NetworkInterface_wifi.c)

Uses Microchip's WILC1000 WiFi chip connected via SPI:

**Initialization:**
```c
xNetworkInterfaceInitialise()
    WIFI_On()                      // Power on WiFi module
    WIFI_ConnectAP(&xNetworkParams) // Connect to AP
```

Uses the `iot_wifi.h` API (Amazon FreeRTOS WiFi HAL abstraction) for WiFi management, and `WDRV_EXT_DataSend()` for actual packet transmission.

### 6.3 No Generic WiFi HAL

FreeRTOS-Plus-TCP does **not** define a WiFi HAL. WiFi management (scan, connect, disconnect) is handled by vendor-specific APIs:
- ESP32: `esp_wifi_*` from ESP-IDF
- PIC32: `WIFI_*` from Microchip's IoT WiFi abstraction
- Other platforms: Vendor SDK

The FreeRTOS-Plus-TCP network interface only cares about Ethernet frame exchange — it treats WiFi like any other link layer that delivers Ethernet frames.

## 7. IP-Task Event Types

The IP-task processes these events from `xNetworkEventQueue`:

| Event | Source | Handler | Purpose |
|-------|--------|---------|---------|
| `eNetworkDownEvent` | Driver or init | `prvProcessNetworkDownEvent()` | Re-initialize interface |
| `eNetworkRxEvent` | Driver ISR/callback | `prvHandleEthernetPacket()` | Process received frame |
| `eNetworkTxEvent` | Internal | `prvForwardTxPacket()` | Send packet via driver |
| `eStackTxEvent` | Socket layer (UDP) | `vProcessGeneratedUDPPacket()` | Finalize and send UDP packet |
| `eARPTimerEvent` | ARP timer | `vARPAgeCache()` | Age ARP cache entries |
| `eNDTimerEvent` | ND timer | `vNDAgeCache()` | Age Neighbor Discovery cache |
| `eDHCPEvent` | DHCP timer | `prvCallDHCP_RA_Handler()` | Run DHCP/RA state machine |
| `eTCPTimerEvent` | TCP timer | (sets flag, processed in `vCheckNetworkTimers()`) | TCP retransmit, keepalive |
| `eSocketBindEvent` | `FreeRTOS_bind()` | `vSocketBind()` | Bind socket to port |
| `eSocketCloseEvent` | `FreeRTOS_closesocket()` | `vSocketClose()` | Close and free socket |
| `eSocketSelectEvent` | `FreeRTOS_select()` | `vSocketSelect()` | Check socket readiness |
| `eTCPAcceptEvent` | `FreeRTOS_accept()` | `xTCPCheckNewClient()` | Check for new connections |

## 8. Cellular Interface as Alternative to WiFi

For devices without WiFi, FreeRTOS provides a cellular path through `FreeRTOS-Cellular-Interface`:

```
Application
    │
    ▼
coreMQTT / coreHTTP
    │
    ▼
network_transport (tcp_sockets_wrapper port: cellular)
    │
    ▼
FreeRTOS-Cellular-Interface
    │ Socket API: Cellular_SocketCreate/Connect/Send/Recv
    │ Modem AT commands via CellularCommInterface_t (UART HAL)
    ▼
Cellular Modem (BG96 / HL7802 / SARA-R4)
    │
    ▼
LTE/NB-IoT Network
```

The cellular interface provides its own socket abstraction (`Cellular_SocketHandle_t`) and the `tcp_sockets_wrapper` maps FreeRTOS+TCP socket calls to cellular socket operations. This allows application code to use the same protocol libraries (coreMQTT, coreHTTP) over either WiFi or cellular.

## 9. Transport Layer (TLS)

The `network_transport` module bridges application protocols to the network:

```
┌─────────────────────────────────────┐
│ coreMQTT / coreHTTP / coreSNTP      │
│ (uses TransportInterface_t)         │
│   .recv()  .send()  .writev()       │
└───────────────┬─────────────────────┘
                ▼
┌───────────────────────────────────────┐
│ network_transport                     │
│                                       │
│ ┌─────────────────────────────────┐   │
│ │ TLS layer (mbedTLS or wolfSSL)  │   │
│ │ - TLS handshake                 │   │
│ │ - Encrypt/decrypt               │   │
│ │ - Certificate verification      │   │
│ └──────────────┬──────────────────┘   │
│                ▼                      │
│ ┌─────────────────────────────────┐   │
│ │ TCP Sockets Wrapper             │   │
│ │ - tcp_sockets_connect()         │   │
│ │ - tcp_sockets_recv()            │   │
│ │ - tcp_sockets_send()            │   │
│ └──────────────┬──────────────────┘   │
└────────────────┼──────────────────────┘
                 ▼
        FreeRTOS+TCP sockets
        FreeRTOS_connect / FreeRTOS_recv / FreeRTOS_send
```

**Transport options:**

| Transport | TLS | Auth | Use case |
|-----------|-----|------|----------|
| `transport_mbedtls` | mbedTLS | Cert on heap | Standard TLS |
| `transport_mbedtls_pkcs11` | mbedTLS + PKCS#11 | Secure element | Hardware key storage |
| `transport_wolfSSL` | wolfSSL | Cert on heap | Alternative TLS |
| `transport_plaintext` | None | None | Testing / trusted network |

## 10. POSIX Simulation Networking

For development on Linux/macOS, FreeRTOS-Plus-TCP can run in a simulated environment:

- **Linux driver** (`portable/NetworkInterface/linux/`): Uses libpcap to send/receive real packets on the host's network interface. Creates dedicated send/recv threads.
- **libslirp** (used by some demos): Provides a user-mode network stack (NAT + DHCP + DNS forwarding) so FreeRTOS+TCP runs entirely in userspace without root privileges or TUN/TAP.

The POSIX demo at `FreeRTOS-Plus/Demo/FreeRTOS_Plus_TCP_Echo_Posix/` builds a complete echo server running on Linux.
