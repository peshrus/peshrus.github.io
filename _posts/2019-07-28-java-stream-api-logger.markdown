---
layout: post
title:  "Java Stream API Logger"
date:   2019-07-28 14:45:00 +0300
categories: java stream-api
---
While I was preparing for [OCA](/java/oca/2019/05/13/oca.html) exam I realized that it was not bad to have something to debug streams actively used in 
Java projects now. Nobody cares about making them clearer in the code that's why I decided to make a Java agent to add some logging when required. 

Meet [Java Stream API Logger](https://github.com/peshrus/java-stream-api-logger)!

I would like to change the code of the project to add [PeekConsumer](https://github.com/peshrus/java-stream-api-logger/blob/master/src/main/java/com/peshchuk/java/stream/api/logger/PeekConsumer.java) 
directly to the modified bytecode, not to specify `-Xbootclasspath/a` but still has not realized how to do that properly. 
[Byte Buddy](https://bytebuddy.net/) or [ASM](https://asm.ow2.io/) may help ,e in the future...
