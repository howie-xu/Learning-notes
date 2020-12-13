# JVM笔记

### Java支持多平台原理

<img src="https://imgs-seven.vercel.app/JVM/java多平台原理.jpg" alt="Java支持多平台原理" style="zoom:70%;" />

### JDK|JRE|JVM

Java官网 ：https://docs.oracle.com/javase/8/
Reference -> Developer Guides -> 定位到:https://docs.oracle.com/javase/8/docs/index.html  

~~~
JDK 8 is a superset of JRE 8, and contains everything that is in JRE 8, plus tools such as the compilers and debuggers necessary for developing applets and applications. JRE 8 provides the libraries, the Java Virtual Machine (JVM), and other components to run applets and applications written in the Java programming language. Note that the JRE includes components not required by the Java SE specification, including both standard and non-standard Java components.
~~~

![Java概念图](https://imgs-seven.vercel.app/JVM/JavaConceptualDiagram.PNG)

###   源码到类文件  

  编译: javac -g:vars Person.java ---> Person.class  

#### Javac编译

<img src="https://imgs-seven.vercel.app/JVM/编译过程.jpg" alt="编译过程" style="zoom:80%;" />

#### 类文件（Class文件）

官网 ： https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html  

### 类文件到虚拟机（类加载机制）