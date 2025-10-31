# Class 1: Introduction

Chapter 1: introduction  

Main Points:  

- Internet? Protocol?  
- network edge  
- network core  
- performance  
- protocal layers, service models  
- security  

## 1. OSI Model

The OSI (Open Systems Interconnection) Model  
layers:  

1. application(应用层)
2. presentation(表示层)  
3. session(会话层)
4. transport(传输层)
5. network(网络层)
6. data link(数据链路层)
7. physical(物理层)

end system/ host:  
owns the complete 7 layers  

router:  
only the last 3 layers  

### 1.1. Understanding

why **layers**?  
**Abstraction**  

when we send some data through the network  
we packet(分组) the data  
and add some headers(头) before the data  
from top to bottom layers, some layers add their own headers in sequence  

what is **header**?  
the essence of header is **control**  
it is also called **signaling(信令)**

the first and most important question in network design is about  
the relation between **data** and **control**  

signaling examples:  
随路信令(In-band signaling / Channel Associated Signaling, CAS)  
共路信令(Out-of-band signaling / Common Channel Signaling, CCS)  

### 1.2. Abstractions in Networking

Data Plane(平面) Abstractions  
OSI  
TCP/IP  

Control Plane Abstractions  
SDN(Software Defined Networking)  
openflow protocol  

### 1.3. Summary

the most original and crucial questions of network design questions:  

1. the relation between data and control  
2. layers(how to do abstraction)  

## 2. Network

collection of nodes and links which connect them  

*network != Internet*  

**computer network**  

end system + link + switching node  
host + link + router(Internet)  

properties:  

- Interconnected  
- Autonomous  
- Collection  
- Scalability  

### 2.1. Different Views of Internet

"nuts and bolts":  
network of networks  

"services":  
infrastructure that provides services to apps  
and programming interface to distributed apps  

### 2.2. Protocol

computer protocol:  
defines the **format**, **order of messages sent and received** and **actions** taken on message transmission, receipt  

## Network Core

packet/circuit switching  
internet structure  

packet switching:  
hosts break app-layer messages into packets  

### Circuit Switching(电路交换)

end-end resources allocated to and reserved for each call  
call links the source and destination  

properties:  

1. dedicated resources(专用资源)  
    not sharing  
    similar to the circuit  
2. circuit segment idle if not used  
3. commonly used in traditional telephone networks

#### FDM and TDM

- FDM: frequency division multiplexing(频分复用)  
- TDM: time division multiplexing(时分复用)  

example: T1/DS1(Digital Signal Designator)  
it concentrated multiple audio calls on a single physical circuit  
based on the TDM  

|   properties   |                         description                          |             relation between circuit switching             |
| :------------: | :----------------------------------------------------------: | :--------------------------------------------------------: |
|      rate      |                          1.544 Mbps                          |             the total capacity of the T1 line              |
|   component    |                       24 DS0 Channels                        | based on TDM, 24 Channels are allocated to different users |
|    channel     |                      64 Kbps every DS0                       |              the standard rate of the digital              |
| implementation |                             TDM                              |            bandwidth divided into 24 time slots            |
|     frame      | T1 frame is composed of 24 DS0 slots and 1 extra framing bit |     the framing bit is used to synchronize the frames      |
|   frame rate   |                  8000 frames/sec * 193 bits                  |                                                            |

### Packet Switching(分组交换)

**each end-end data stream divided into packets**  

properties:  

- share network resources between users  
- each packet uses full link bandwidth  
- resources used as needed(资源按需使用)  
- aggregate resource demand can exceed amount available  
- congestion may occur: packets queue, wait for link use  
- store-and-forward(存储转发)

*unlike circuit switching, bandwidth is not divided in to "pieces", allocation is not dedicated and resource is not reserved*  

#### Mainly Property 1: Queueing

as mentioned above: the aggregate resource demand can exceed amount available  

in this situation, the packets will queue  
and if the memory(buffer) in router is full, the packets may be lost(dropped)  

#### Mainly Property 2: Store-and-Forward

entire packet must be received in the router before it can be transmitted on next link  

Store-and-Forward mechanism contains:  

1. Store: receive and store  
2. Process: check the header info  
3. Queue: if there are some other packets, it will queue  
4. Forward: transmit the packet(push the bit stream to the link)  

normal formula to calculate the latency:  

$$
\text{Latency} = \frac{\text{L}}{\text{R}}
$$

- L: packet length (bits)
- R: link bandwidth (bps)

**why?**  
this mechanism can guarantee the integrity of the packet  
which cost some time for the router to wait for the whole packet  
this is the origin of the latency which is the period from the router receiving the first bit to enabled to transmit the packet  
*here we assuming zero propagation delay*  

## Performance

next we'll talk about:  
loss, delay, throughput  

### Loss

when the arrival rate keeps exceeding the output link capacity  
the buffer may overflow  
which may cause the router dropping packets  

### Delay

for sources:  

$$
d_{nodal} = d_{proc} + d_{queue} + d_{trans} + d_{prop}
$$

- $d_{proc}$: processing delay(typically < 1 ms)
- $d_{queue}$: queuing delay(varies from 0 to seconds, depending on congestion situation)
- $d_{trans}$: transmission delay(L/R, typically a few ms or less)
- $d_{prop}$: propagation delay(d/s, typically a few ms or less)
