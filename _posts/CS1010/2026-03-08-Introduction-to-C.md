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
* [Format Specifiers](#format-specifiers)
* [Variables and Naming](#variables-and-naming)
* [Variable Type](#variable-type)
* [Bits and Bytes](#bits-and-bytes) 
* [Arithmetic Operators and Assignment Operators](#arithmetic-operators-and-assignment-operators)
* [Summary](#summary)
* [Tips](#tips)
* [Exam Questions](#exam-questions)


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
    * More importantly, the computer (specifically the OS and the linker) needs a fixed starting address, which is the main function
    * When you run your compiled code, the OS sets up the memory, loads your program, and then jumps directly to the first instruction inside main


*💡 CS1010 Tip: It is considered "Best Practice" in C to use void because it makes your code more type-safe. It prevents accidental data from being passed where it isn't wanted.*

Line 4: `printf(“Hello World!\n”);`
* We are calling the printf function which ouputs anything you write in the brackets!
    * Make sure that you wrap your text in "";
    * We will cover how to add variables in your output later
    * `\n` means new line. In C, the output does not automatically move to the next line after outputting the given text/values, so you will have to do it for them!
* This line of code will output `Hello World!`

What if you want to print something that is not determined yet? How do we print `2 * 3`? Sure we can do this simple calculation in our heads right now, but what if the numbers get very complicated? 

The solution is to use the computer to do the calculation for you! Save your brain power for more complicated tasks later on!

### Format Specifiers
In this program, we are trying to calculate the tax of a good worth $888. If the tax is 9%, then the total tax value should be `888 * 9/100`! Simple math!

How do we output the value though? Would the following example work? 

`printf("Tax (cents): 888 * 9 / 100");`

The answer is no! As said previously, printf will output everything wrapped in "". This means that it would not consider the math operation you wrote down and just directly print it! The output will therefore be:

> Tax (cents): 888 * 9 / 100 

How you may ask, should we output the calculated value? The answer is format specifiers.

```
1. #include <stdio.h>
2.
3. int main(void){
4.     printf("Tax (cents): %d\n", 888 * 9 / 100);
5. }
```
Line 4: `printf("Tax (cents): %d\n", 888 * 9 / 100);`
* We have a format specifier `%d` and we have also left the calculations outside of the "text"
    * Format specifiers are a form of placeholders that tell the compiler what type of data to expect and in what position
    * In this example, the `%d` specifier represents an integer data type, and therefore expects an integer argument
    * The argument is placed outside of the "text" in respective positions according to the positioning of the format specifiers. 
* This code will now output: `Tax (cents): 79`

Format Specifiers are a great way to output any unknown variable or calculations that you may have in your code. It is also very helpful in print debugging. In CS1010, it is expected knowledge after week 1 so definitely try to familiarise yourself with it!

Here are some of the format specifiers that we frequently use:
| Specifier | Data Type | Description | Example Output |
| :--- | :--- | :--- | :--- |
| **`%d`** or **`%i`** | `int` | Signed decimal integer | `42`, `-15` |
| **`%u`** | `unsigned int` | Unsigned decimal integer | `42` |
| **`%zu`** | `size_t` | Unsigned integer representing memory size | `40`, `8` |
| **`%f`** | `float` or `double` | Decimal floating-point | `3.141590` |
| **`%c`** | `char` | Single character | `'A'` |
| **`%s`** | `char[]` or `char*` | String of characters | `"Hello"` |
| **`%p`** | `void *` | Pointer (memory address) | `0x7ffee1b2c` |
| **`%x`** or **`%X`** | `unsigned int` | Unsigned hexadecimal | `2a`, `2A` |

## Variables and Naming

## Variable Type

## Bits and Bytes

## Arithmetic Operators and Assignment Operators

## Summary
| Specifier | Data Type | Description | Example Output |
| :--- | :--- | :--- | :--- |
| **`%d`** or **`%i`** | `int` | Signed decimal integer | `42`, `-15` |
| **`%u`** | `unsigned int` | Unsigned decimal integer | `42` |
| **`%f`** | `float` or `double` | Decimal floating-point | `3.141590` |
| **`%c`** | `char` | Single character | `'A'` |
| **`%s`** | `char[]` or `char*` | String of characters | `"Hello"` |
| **`%p`** | `void *` | Pointer (memory address) | `0x7ffee1b2c` |
| **`%x`** or **`%X`** | `unsigned int` | Unsigned hexadecimal | `2a`, `2A` |

## Tips
`int main(void)`

*💡 It is considered "Best Practice" in C to use void because it makes your code more type-safe. It prevents accidental data from being passed where it isn't wanted.*

## Exam Questions
