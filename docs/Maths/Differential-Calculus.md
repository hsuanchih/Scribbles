---
layout: default
title: Differential Calculus
parent: Maths
---

{% include_relative includes/head.html %}

# Differential Calculus
---


## First Principles of Calculus
---
\\(gradient_{m_{secant}} = \cfrac{f(x+h) - f(x)}{h}\\)

\\(gradient_{m_{tangent}} = \lim\limits_{h \to 0} \cfrac{f(x+h) - f(x)}{h} = f\prime(x)\\)

## Derivatives of Compound Functions
---
### Derivative of Sums
---
let \\(f(x) = u(x) + v(x)\\),

then \\(f\prime(x) = u\prime(x) + v\prime(x)\\)

__Proof:__

Using first principles:

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{f(x+h) - f(x)}{h}\\)

by substitution,

\\(= \lim\limits_{h \to 0} \cfrac{(u(x+h) + v(x+h)) - (u(x) + v(x))}{h}\\)

\\(= \lim\limits_{h \to 0} \cfrac{u(x+h) + v(x+h) - u(x) - v(x)}{h}\\)

from re-arrangement,

\\(= \lim\limits_{h \to 0} \cfrac{(u(x+h) - u(x)) + (v(x+h) - v(x))}{h}\\)

\\(= \lim\limits_{h \to 0} \cfrac{u(x+h) - u(x)}{h} + \lim\limits_{h \to 0} \cfrac{v(x+h) - v(x)}{h}\\)

accroding to first principles,

\\(= u\prime(x) + v\prime(x)\\)