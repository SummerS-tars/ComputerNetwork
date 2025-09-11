# Class 1: Introduction

## OSI Model

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

### Understanding

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

### Abstractions in Networking

Data Plane(平面) Abstractions  
OSI  
TCP/IP  

Control Plane Abstractions  
SDN(Software Defined Networking)  
openflow protocol  

### Summary

the most original and crucial questions of network design questions:  

1. the relation between data and control  
2. layers(how to do abstraction)  
