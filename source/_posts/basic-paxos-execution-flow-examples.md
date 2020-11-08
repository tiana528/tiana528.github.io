---
title: basic paxos execution flow examples
date: 2020-11-08 15:37:49
tags: distributed system
---

# Background
The paper of [Paxos Made Simple](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/paxos-simple-Copy.pdf) and the video of [implementing replicated logs with paxos](https://www.youtube.com/watch?v=JEpsBg0AO6o&ab_channel=DiegoOngaro) are very good materials for learning Paxos.

It is not very easy to understand Paxos algorithm, and there are not many examples to show how it is working.

Some readers might be confusing about how Paxos algorithm can help achieve the consensus, and this articles aims at providing some execution examples to help understand it.

# Features and constraints

- It ensures that a single one among the proposed values is chosen.
- A value is chosen : A value is chosen when a single proposal with that value has been accepted by a majority of the acceptors. 
- It is fault tolerant. In general, a system which supports F failures must have 2F+1 replicas.
- Messages can take arbitrarily long to be delivered, can be duplicated, and can be lost, but they are not corrupted.

# Procedure
There two phrases according to the above paper.

- Phase 1. (prepare phrase)
    - (a) A proposer selects a proposal number n and sends a prepare request with number n to a majority of acceptors.
    - (b) If an acceptor receives a prepare request with number n greater than that of any prepare request to which it has already responded, then it responds to the request with a promise not to accept any more proposals numbered less than n and with the highest-numbered proposal (if any) that it has accepted.
- Phase 2. (accept phrase)
    - (a) If the proposer receives a response to its prepare requests (numbered n) from a majority of acceptors, then it sends an accept request to each of those acceptors for a proposal numbered n with a value v, where v is the value of the highest-numbered proposal among the responses, or is any value if the responses reported no proposals. 
    - (b) If an acceptor receives an accept request for a proposal numbered n, it accepts the proposal unless it has already responded to a prepare request having a number greater than n.

# Examples

## 1
One node.
![](paxos_1.png)

- how to read this graph
  - There is only one node(which acts both as proposer and acceptor), the node’s name is s1
  - “s1,x” means that s1 tries to propose the value of x
  - “P1” means prepare phrase with proposal number 1 finishes properly
  - “A1,x” means accept phrase with proposal number 1 and value x finishes properly

- this examples shows that
  - With only 1 node, the node itself can achieve consensus, the chosen value is x
  - With only 1 node, if this node crashes, then we will lose the information that which value has been chosen

## 2
Two nodes
![](paxos_2.png)


- how to read this graph
  - There are two nodes, s1 tries to propose value x, s2 does not propose values.
  - Prepare phrase of proposal number 1 finishes successfully on both nodes, then the accept phrase also finishes successfully on both nodes, the chosen value is x.

- this examples shows that
  - With 2 nodes, it can achieve consensus if no nodes fail.

## 3
Two nodes
![](paxos_3.png)


- how to read this graph
  - s3 crashes, and s1 cannot get majority of instances to finish the prepare phrase, so it fails to propose.

- this examples shows that
  - With 2 nodes, if 1 node fail, then it cannot achieve consensus.

## 4
Two nodes
![](paxos_4.png)


- how to read this graph
  - X means the phrase fails

- this examples shows that
  - When s1 submits accept requests of proposal number 1 to s1 and s2, it fails on both nodes. Because both nodes have finished responding the prepare request of proposal number 2, and promise that they won’t accept any proposal number less than 2(according to rule phrase1b).
  - The chosen value is y

## 5
Two nodes
![](paxos_5.png)



- this examples shows that
  - The chosen value is y, the reason is the same as above

## 6
Two nodes
![](paxos_6.png)



- this examples shows that
  - The chosen value is y, the reason is the same as above

## 7
Two nodes
![](paxos_7.png)


- this examples shows that
  - When s1 submits prepare request to s2, it will be rejected because s2 has already responded to a higher proposal number(according to rule phrase1b).

## 8
Three nodes
![](paxos_8.png)


- this examples shows that
  - When s1 submits accept requests to 3 nodes, all of them failed because all 3 nodes have promised that they won’t accept proposal number which is smaller than 2.
  - value y is chosen.

## 9
Three nodes
![](paxos_9.png)


- this examples shows that
  - When s3 gets the majority responses for the proposal 2, it knows that s1 has already accepted value x, so s3 also updates its value to x(according to rule phrase 2a).
  - value x is chosen.

## 10
Three nodes
![](paxos_10.png)


- this examples shows that
  - When 1 node crashes among 3 nodes, it can still achieve the consensus.
  - The chosen value is y. Note that it is fine that the chosen value is not x since x is not accepted by the majority.

## 11
Three nodes
![](paxos_11.png)


- this examples shows that
  - When 1 node crashes among 3 nodes, it can still achieve the consensus.
  - The chosen value is x since x is accepted by the majority.

## 12
Three nodes
![](paxos_12.png)


- this examples shows that
  - The chosen value is x
  - Note that only s1 and s3 knows that value is chosen because they are proposals, if s2 wants to know what value has been chosen, it can try to propose.

## 13
Three nodes
![](paxos_13.png)



## 14
Three nodes
![](paxos_14.png)


## 15
Three nodes
![](paxos_15.png)


## 16
Three nodes
![](paxos_16.png)


## 17
Three nodes
![](paxos_17.png)


## 18
Three nodes
![](paxos_18.png)

- this examples shows that
  - If 2 nodes among 3 crash, then we will not be able to know what has been chosen.

## 19
Three nodes
![](paxos_19.png)

- this examples shows that
  - it is possible that livelock happens, every proposal proposes a new proposal with a larger number when it is rejected
