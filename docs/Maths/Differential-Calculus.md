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

---

### Applying the First Principles of Calculus
---

***Example - \\(f(x) = x^2\\)***

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(x+h)^2 - x^2}{h}\\)

by expansion,

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(x^2 + 2xh + h^2) - x^2}{h}\\)

via simplification,

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{2xh + h^2}{h} = \lim\limits_{h \to 0} \cfrac{(2x + h) \cdot h}{h} = \lim\limits_{h \to 0} 2x + h\\)

by substitution,

\\(\boxed{(f\prime(x) = 2x}\\)

<br/>***Example - \\(f(x) = x^3\\)***

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(x+h)^2 - x^2}{h}\\)

by expansion,

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(x^3 + 3x^2h + 3xh^2 + h^3) - x^3}{h}\\)

via simplification,

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{3x^2h + 3xh^2 + h^3}{h} = \lim\limits_{h \to 0} \cfrac{(3x^2 + 3xh + h^2) \cdot h}{h} = \lim\limits_{h \to 0} 3x^2 + 3xh + h^2\\)

by substitution,

\\(\boxed{f\prime(x) = 3x^2}\\)

---
## Derivatives of Compound Functions
---
### Derivative of Sums
<br/>
let \\(f(x) = u(x) + v(x)\\),

then \\(f\prime(x) = u\prime(x) + v\prime(x)\\)

---

***Proof:***

Using first principles:

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{f(x+h) - f(x)}{h}\\)

by substitution,

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(u(x+h) + v(x+h)) - (u(x) + v(x))}{h}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{u(x+h) + v(x+h) - u(x) - v(x)}{h}\\)

from re-arrangement,

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(u(x+h) - u(x)) + (v(x+h) - v(x))}{h}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{u(x+h) - u(x)}{h} + \lim\limits_{h \to 0} \cfrac{v(x+h) - v(x)}{h}\\)

back to first principles,

\\(f\prime(x) = u\prime(x) + v\prime(x)\\)