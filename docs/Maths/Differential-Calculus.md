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

[comment]: Example - f(x) = x^2
\\(\underline{\bold{Example:}\space f(x) = x^2}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(x+h)^2 - x^2}{h}\\)

\\(\text{by expansion,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(x^2 + 2xh + h^2) - x^2}{h}\\)

\\(\text{via simplification,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{2xh + h^2}{h} = \lim\limits_{h \to 0} \cfrac{(2x + h) \cdot h}{h} = \lim\limits_{h \to 0} 2x + h\\)

\\(\text{from substitution,}\\)

\\(\boxed{(f\prime(x) = 2x}\\)



[comment]: Example - f(x) = x^3
<br/><br/>
\\(\underline{\bold{Example:}\space f(x) = x^3}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(x+h)^2 - x^2}{h}\\)

\\(\text{by expansion,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(x^3 + 3x^2h + 3xh^2 + h^3) - x^3}{h}\\)

\\(\text{via simplification,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{3x^2h + 3xh^2 + h^3}{h} = \lim\limits_{h \to 0} \cfrac{(3x^2 + 3xh + h^2) \cdot h}{h} = \lim\limits_{h \to 0} 3x^2 + 3xh + h^2\\)

\\(\text{from substitution,}\\)

\\(\boxed{f\prime(x) = 3x^2}\\)

---
## Derivatives of Compound Functions
---
### Derivative of Sums Rule
<br/>
\\(\text{if }\boxed{f(x) = u(x) + v(x)},\text{ then }\boxed{f\prime(x) = u\prime(x) + v\prime(x)}\\)

---

[comment]: Proof - Derivative of Sums
\\(\underline{\bold{Proof:}}\\)

\\(\text{Using first principles:}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{f(x+h) - f(x)}{h}\\)

\\(\text{by substitution,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(u(x+h) + v(x+h)) - (u(x) + v(x))}{h}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{u(x+h) + v(x+h) - u(x) - v(x)}{h}\\)

\\(\text{from re-arrangement,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(u(x+h) - u(x)) + (v(x+h) - v(x))}{h}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{u(x+h) - u(x)}{h} + \lim\limits_{h \to 0} \cfrac{v(x+h) - v(x)}{h}\\)

\\(\text{back to first principles,}\\)

\\(\boxed{f\prime(x) = u\prime(x) + v\prime(x)}\\)


[comment]: Example - f(x) = (4x^2 + 1)^2
<br/><br/>
\\(\underline{\bold{Example:}\space f(x) = (4x^2 + 1)^2}\\)

\\(f(x) = (4x^2 + 1)^2 = 16x^4 + 8x^2 + 1\\)

\\(\text{let } u(x) = 16x^4,\space v(x) = 8x^2 + 1\\)

\\(\text{we know from above that the derivative of sums is the sum of derivatives,}\\)

\\(f\prime(x) = u\prime(x) + v\prime(x) = \frac{d}{dx}16x^4 + \frac{d}{dx}(8x^2 + 1)\\)

\\(\text{diffrentiate using first principles,}\\)

\\(\boxed{f\prime(x) = 64x^3 + 16x}\\)