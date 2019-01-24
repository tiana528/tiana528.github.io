---
title: Very basic understanding of CAP theorem
date: 2019-01-19 21:46:03
tags: database
---


# Wiki description

According to the [wiki](https://en.wikipedia.org/wiki/CAP_theorem): CAP theorem explains that it is impossible for a *distributed data store* to simultaneously provide more than two out of the following three guarantees:

- Consistency: Every read receives the most recent write or an error.
- Availability: Every request receives a (non-error) response – without the guarantee that it contains the most recent write.
- Partition tolerance: The system continues to operate despite an arbitrary number of messages being dropped (or delayed) by the network between nodes.

# Simple understanding

Here explain the theorem using a simple example. Considering there are two nodes, and data is replicated in two nodes.
- If AP, then not C. When network partition happens(P), a write request arrives at one node and success(A), then the data on the two nodes must be inconsistent, meaning C is not satisfied.
- If CP, then not A. When network partition happens(P), the read on either node always gets the latest written value. Then write operation must be forbidden on both side of the parititons, meaning A is not satisfied.
- IF AC, then not P. If write always successes(A), read always gets the recent written value(C), then network partition must not happen, meaning P is not satisfied.

# Does it explain well

The above understanding of CAP is not sufficient. CAP is always criticized for being too simplistic and often misleading. Actually, many distributed data systems are not even CP, AP or AC, considering the strict definition. It is difficult for people to understand what is CAP by the above simple description which is explained in a lot of places on the Internet. The following of this article aims at **adding a little more about the missing part of the above CAP theorem description and make it understandable**.

# Definition of terms

Before going any further, we need to make precise definition of terms. There are many articles discussing CAP, without clear definition and the readers may not have enough knowledge to understand them.

## Serialization

- serialization is a transaction concept. Multiple transactions are processed in parallel, and each transaction contains one or more instructions. The database needs to {% link schedule https://en.wikipedia.org/wiki/Schedule_(computer_science) [external] [title] %} how the instructions are executed. Note that different execution orders may result in different result.
    - For example, start with the status a = 1, b = 3
        - t1
            - i1 : t1 = a
            - i2 : b = t1 + 1
        - t2
            - i3 : t2 = b
            - i4 : a = t2 + 1
        - Execution order of i1, i3, i2, i4 results in a = 4, b = 2. Execution order of i1, i2, i3, i4 results in a = 3, b = 2.
- A transaction schedule is *serializable* if the outcome is as if transactions are executed atomically and in some sequential order.
- Strict serializability : Serializability + transactions are processed in the "real-time" order.

## How to garrantee serialization
- Using concurrency control protocol  such as [2PL](https://en.wikipedia.org/wiki/Two-phase_locking)
    - 2PL : 2 phrase locking
        - acquire locks in first phrase, release locks in second phrase
    - C2PL : Conservative two-phase locking
        - acquire all locks at the beginning of the transaction, release locks in the second phrase.
    - S2PL : Strict two-phase locking
        - acquire locks in first phrase, release write locks at the end of the  transaction.
    - SS2PL : Strong strict two-phase locking
        - acquire locks in first phrase, release both read and write locks at the end of the transaction.
        - SS2PL has been the concurrency control protocol of choice for most database systems and utilized since their early days in the 1970s.
- Using other approaches

## Linearizability
- [Linearizability](https://en.wikipedia.org/wiki/Linearizability) concept is from concurrent programming.  Linearizability is originally talking about single objects and not transactions. Nowadays, many papers talk about linearilzability transactions the same as strict serializability, and each transaction contains only one operation. 
- How Linearizability is related with CAP
    - Linearizability is what the CAP Theorem calls Consistency.

## Isolation

- {% link Phantom Read Phenomena https://en.wikipedia.org/wiki/Isolation_(database_systems) [external] [title] %}
    - Dirty reads
        - A dirty read occurs when a transaction is allowed to read data from a row that has been modified by another running transaction and not yet committed.
    - Non-repeatable reads
        - A non-repeatable read occurs, when during the course of a transaction, a row is retrieved twice and the values within the row differ between reads.
    - Phantom reads
        - A phantom read occurs when, in the course of a transaction, new rows are added or removed by another transaction to the records being read.
    - Note that only these are insufficient and there are more situations.
- Levels(stronger to lower)
    - Serializable
        - This is the highest isolation level.
        - With a lock-based concurrency control DBMS implementation, serializability requires read and write locks (acquired on selected data) to be released at the end of the transaction. Also range-locks must be acquired when a SELECT query uses a ranged WHERE clause, especially to avoid the phantom reads phenomenon.
    - Repeatable reads
        - a lock-based concurrency control DBMS implementation keeps read and write locks (acquired on selected data) until the end of the transaction. However, range-locks are not managed, so phantom reads can occur.
    - Read committed
        - a lock-based concurrency control DBMS implementation keeps write locks (acquired on selected data) until the end of the transaction, but read locks are released as soon as the SELECT operation is performed (so the non-repeatable reads phenomenon can occur in this isolation level). As in the previous level, range-locks are not managed.
    - Read uncommitted
        - This is the lowest isolation level. In this level, dirty reads are allowed, so one transaction may see not-yet-committed changes made by other transactions.
    - There are also incufficient and there are more that can be listed.
- How isolation is related in CAP
    - Since the C(consistency) in CAP refers to linearizability which is very strong, and per transaction contains one operation, all actions happen instantaneously. The C(consistency) actually ensures the strongest isolation : serializable isolution level.

## What isolation level are commercial databases providing

- READ COMMITTED is defaulted isolation level on PostgreSQL, SQL Server, and Oracle.
- REPEATABLE READ is defaulted isolation level on Mysql Innodb.

We can see that serializable isolation level is not used at lease by default. It is too heavy and required more locks(If use lock as concurrency control), less concurrency, and less throughput. It also infers that in CAP, even if we can only choose 2 of them at the same time, C(strong consistency, linearizability, highest isolation level) is mostly not chosen. Most databases choose weaker consistency, which has better performance.

# Look back at CAP theorem
- C
    - definition
        - A guarantee that every node in a distributed cluster returns the same, most recent, successful write. Consistency refers to every client having the same view of the data. There are various types of consistency models. Consistency in CAP (used to prove the theorem) refers to linearizability or sequential consistency, a very strong form of consistency.
    - meaning
        - Apparently, this is very strong consistency and there are no dirty reads, non-repeatable reads, or phantom reads. 
- A
    - definition
        -  Every non-failing node returns a response for all read and write requests in a reasonable amount of time. The key word here is every. To be available, every node on (either side of a network partition) must be able to respond in a reasonable amount of time.
    -  meaning
        -  If some operation finishes exceeding an acceptable time(e.g. 30 seconds), it just means not available.
        -  This definition of availability is also strong. Note that during network partition, even if you say the partition which contains the majority of nodes responses the client, it is not seen as achieve availability in CAP. It is called availability if the less node partition nodes also responses write/read to the client without error.
- P
    - definition
        - The system continues to function and upholds its consistency guarantees in spite of network partitions. Network partitions are a fact of life. Distributed systems guaranteeing partition tolerance can gracefully recover from partitions once the partition heals.
    - meaning
        - This is unavoided in a distributed environment. Not only network parition happens, but also network connection timeout can be seen as parititon.

# Summarize

In a network partition, even if it is only posssible to make a choice between avaialbility or consistency. Many databases are not choosing any of them, since the consistency (linearizability) is so expensive. Actually, this leaves a space for the database designers/users to choose a level of balanced consistency and availability according to the use case. Whenever looking into a database, it will be a good point to check what level of A and C does it provides when P happens, even if it is not perfect A or C according to the CAP definition.


# Reference

- [please-stop-calling-databases-cp-or-ap](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html)
- [cap-twelve-years-later-how-the-rules-have-changed](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed)
- [practical-guide-sql-isolation](https://begriffs.com/posts/2017-08-01-practical-guide-sql-isolation.html)
- [consistency](https://irenezhang.net/blog/2015/02/01/consistency.html)
- [What is the strongest consistency that could be achieved
in a large scale distributed system?](https://project.inria.fr/epfl-Inria/files/2017/02/JadHamza-talk.pdf)
- [linearizability-versus-serializability/](http://www.bailis.org/blog/linearizability-versus-serializability/)
- [linearizability-and-serializability/](https://dddpaul.github.io/blog/2016/03/17/linearizability-and-serializability/)
- [linearizability-and-serializability-in-context-of-software-transactional-memory](https://cs.stackexchange.com/questions/41698/linearizability-and-serializability-in-context-of-software-transactional-memory)
- [understanding-the-cap-theorem](https://dzone.com/articles/understanding-the-cap-theorem)
