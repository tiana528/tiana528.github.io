---
title: how oracle java license change affect users
date: 2019-09-23 17:58:21
tags:
---


# oracle java

## platforms
oracle java has 3 platforms, Java SE for standard usage, Java EE for enterprise usage, and Java ME for micro usage.
They are all specifications and are not implementations.

Some specification examples:
- Java SE
    - specification
        - https://docs.oracle.com/javase/specs/
        - Java Language Specification and Java Virtual Machine Specification
            - Grammars, Lexical Structure, Conversions, Classes, Interfaces, Arrays, Threads...
        - Java Virtual Machine Specification
            - Compiler, loading, linking, instruction set...
    - api 
        - https://docs.oracle.com/javase/8/docs/api/
        - https://docs.oracle.com/javase/8/docs/api/
        - java.applet, java.io, java.lang, java.sql, java.util...
- Java EE
    - https://javaee.github.io/javaee-spec/javadocs/
    - javax.annotation, javax.xml.rpc.handler...
- Java ME
    - https://www.oracle.com/java/technologies/javameoverview.html
    - IOT...
## relationship between jdk, jre and Java SE
- Refer to https://docs.oracle.com/javase/8/docs/, Java SE defines the specification. JDK and JRE are implementations of the Java SE.
- JDK is a superset of JRE. Meaning when you download JDK, the JRE is always included.
- JDK is for development programs, JRE is for running programs.

## oracle jdk user types
According to https://www.oracle.com/technetwork/java/javase/overview/oracle-jdk-faqs.html, there are at least three kinds of users.
- personal users
    - Personal use is using Java on a desktop or laptop computer to do things such as to play games or run other personal applications. If you are using Java on a desktop or laptop computer as part of any business operations, that is not personal use.
- commercial users
- Oracle Product users


## oracle jdk license
Two distinct Java releases since JAVA8 after April 16, 2019(https://www.oracle.com/technetwork/java/javase/overview/oracle-jdk-faqs.html)
- Oracle OpenJDK releases
    - license : under the open source GNU General Public License v2, with the Classpath Exception (GPLv2+CPE)
- Oracle Java SE product releases
    - license : under the OTN License Agreement for Java SE
    - includes Oracle JDK8 and later, and Oracle JRE for Java8 and so on.
    - This license permits personal use, development, testing, prototyping, demonstrating and some other uses at no cost.

Java releases before April 16, 2019
- Oracle JAVA SE product releases
    - license : BCL license


(According to the release note, https://www.java.com/en/download/faq/release_dates.xml, Oracle JDK8 until jdk1.8-201 belong to BCL license, Oracle JDK8 later than it belong to OTN license.)
(Commercial users can use BCL license at no cost for most of the features, but commercial users cannot use OTN license at no cost.)

## do you need to pay to use JAVA?

- no cost
    - All types of users can use `Oracle OpenJDK` at no cost.
    - Personal user and Oracle Product users can use `Oracle Java SE product`(Oracle JDK version before April 16, 2019) at no cost.
- need to pay
    - Commercial users who use `Oracle Java SE product`(Oracle JDK version after April 16, 2019) will require a Java SE Subscription, meaning need to pay money to Oracle.

It means that if you use Oracle JDK8 in the commercial environments at no cost, the latest Oracle JDK8 version you can use is jdk1.8-201, if you want to use newer versions(you would need to do it, for security patches), you need to either use Oracle OpenJDK or pay for the Oracle JDK.
If you want to pay for Oracle JDK, oracle says the price is `as low as $2.50/desktop user/month`, but I think this price is no low...

## suggestion for commercial users
Use Oracle OpenJDK for most of the cases.
Use and pay for Oracle JDK if you really need strong support from Oracle(like financial companies?), which I think is not the case for most of the companies.


# Other JAVAs
Zulu, AdoptOpenJDK, Corretto(from Amazon) and so on.


