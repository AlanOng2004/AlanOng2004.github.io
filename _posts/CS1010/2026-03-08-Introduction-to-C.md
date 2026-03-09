---
title: Introduction to C
description: Writeup on how to start thinking in programs and C syntax
date: 2026-03-08 10:20:00 +0800
categories: [CS1010]
img_path: /assets/posts/CS1010/2026-03-07-Introduction-to-C
tags: [C]
---

Hey everyone, if you are on this page, you are either:
- [ ] taking CS1010
- [ ] learning to code in C
- [ ] my friends trolling my comments section

Either way, your purpose is clear and hopefully I don’t have to convince you why we are learning C!

In this series, I hope to discuss C standard in both a high and low level; teaching you how C code works in your head and also in the computer. I will also be sharing some exam tips and how to better visualise and understand code in general. 

If you do not want to read the long in-depth explanations, you are encouraged to refer to my summaries!

Hopefully by the end of this article series, you will be able to understand C (to some extent) and score well for your exams! 

#### Content Overview:
* [Introduction to C](#introduction-to-c)
* [Variables and Naming](#variables-and-naming)
* [Variable Type](#variable-type)
* [Bits and Bytes](#bits-and-bytes) 
* [Arithmetic Operators and Assignment Operators](#arithmetic-operators-and-assignment-operators)
* [Tips](#tips)
* [Exam Questions](#exam-questions)
* [Summary](#summary)

## Introduction to C
Let’s take a look at what C programs usually look like:

```
1. #include <stdio.h>
2. 
3. int main(void){ 
4.     printf(“Hello World!\n”);
5. }
```

If you are a complete beginner, or have only coded with python before, I am sure this code may look slightly foreign to you! Don't worry, I will go through the code line by line with you!

Line 1: `#include <stdio.h>`
* We are including the C "standard input/output" library here!
* `stdio.h` is a header file which contains most of your important functions that you usually use while coding
* Some of these functions are:
    * `printf` (including format specifiers like `%f`, `%s`, etc)
    * `sscanf`
    * `getchar`
    * `fopen`

Line 3: `int main(void)`
* This is fundamentally a function declaration (we will delve deeper into functions later on)
* We are essentially declaring the main function to the compiler
* Why have a main function?
    * It is good for you to organise your own thoughts
    * More importantly, the computer (specifically the OS and the linker) needs a fixed starting address, which is the main function.
    * When you run your compiled code, the OS sets up the memory, loads your program, and then jumps directly to the first instruction inside main.


*💡 CS1010 Tip: It is considered "Best Practice" in C to use void because it makes your code more type-safe. It prevents accidental data from being passed where it isn't wanted.*

Line 4: `printf(“Hello World!\n”);`


## Variables and Naming

## Variable Type

## Bits and Bytes

## Arithmetic Operators and Assignment Operators

## Summary

## Tips
`int main(void)`

*💡 It is considered "Best Practice" in C to use void because it makes your code more type-safe. It prevents accidental data from being passed where it isn't wanted.*



## Exam Questions
