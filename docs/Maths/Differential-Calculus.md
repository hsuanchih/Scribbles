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
### Applying the First Principles of Calculus:
<br/>
\\(f(x) = x^2 \space \text{, find } f\prime(x)\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(x+h)^2 - x^2}{h}\\)

\\(\text{by expansion,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(x^2 + 2xh + h^2) - x^2}{h}\\)

\\(\text{via simplification,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{2xh + h^2}{h} = \lim\limits_{h \to 0} \cfrac{(2x + h) \cdot h}{h} = \lim\limits_{h \to 0} 2x + h\\)

\\(\text{from substitution,}\\)

\\(\boxed{(f\prime(x) = 2x}\\)


<br/><br/>
\\(f(x) = x^3 \space \text{, find } f\prime(x)\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(x+h)^3 - x^3}{h}\\)

\\(\text{by expansion,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{(x^3 + 3x^2h + 3xh^2 + h^3) - x^3}{h}\\)

\\(\text{via simplification,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{3x^2h + 3xh^2 + h^3}{h} = \lim\limits_{h \to 0} \cfrac{(3x^2 + 3xh + h^2) \cdot h}{h} = \lim\limits_{h \to 0} 3x^2 + 3xh + h^2\\)

\\(\text{from substitution,}\\)

\\(\boxed{f\prime(x) = 3x^2}\\)

---
## Derivative of Sums Rule
---
\\(\text{if }\boxed{f(x) = u(x) + v(x)},\text{ then }\boxed{f\prime(x) = u\prime(x) + v\prime(x)}\\)

---
### Proof:
<br/>
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

---
### Example:
<br/>
\\(f(x) = (4x^2+1)^2 \space \text{, find } f\prime(x)\\)

\\(f(x) = (4x^2 + 1)^2 = 16x^4 + 8x^2 + 1\\)

\\(\text{let } u(x) = 16x^4,\space v(x) = 8x^2 + 1\\)

\\(\text{we know from above that the derivative of sums is the sum of derivatives,}\\)

\\(f\prime(x) = u\prime(x) + v\prime(x) = \frac{d}{dx}16x^4 + \frac{d}{dx}(8x^2 + 1)\\)

\\(\text{diffrentiate using first principles,}\\)

\\(\boxed{f\prime(x) = 64x^3 + 16x}\\)

---
## Chain (Function of a Function) Rule
---
\\(\text{if }\boxed{f(x) = u(g(x))},\text{ then }\boxed{f\prime(x) = u\prime(g(x)) \cdot g\prime(x) = \frac{du}{dg} \cdot \frac{dg}{dx}}\\)

---
### Proof:
<br/>
\\(\text{Using first principles:}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{f(x+h) - f(x)}{h}\\)

\\(\text{by substitution,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{u(g(x+h)) - u(g(x))}{h}\\)

\\(\text{re-writing the denominator} \space h \space \text{as} \space (x+h)-x,\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{u(g(x+h))-u(g(x))}{(x+h)-x}\\)

\\(\text{multiply by}\space \cfrac{g(x+h)-g(x)}{g(x+h)-g(x)},\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{u(g(x+h))-u(g(x))}{(x+h)-x} \cdot \cfrac{g(x+h)-g(x)}{g(x+h)-g(x)}\\)

\\(\text{re-arrangement the denominator,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{u(g(x+h))-u(g(x))}{g(x+h)-g(x)} \cdot \cfrac{g(x+h)-g(x)}{(x+h)-x}\\)

\\(\text{since the limit of a product is the product of limits,}\\)

\\(f\prime(x) = \lim\limits_{h \to 0} \cfrac{u(g(x+h))-u(g(x))}{g(x+h)-g(x)} \cdot \lim\limits_{h \to 0} \cfrac{g(x+h)-g(x)}{(x+h)-x}\\)

\\(\text{back to first principles,}\\)

\\(\boxed{f\prime(x) = u\prime(g(x)) \cdot g\prime(x) = \frac{du}{dg} \cdot \frac{dg}{dx}}\\)

---
### Example:
<br/>
\\(f(x) = (4x^2+1)^2 \space \text{, find } f\prime(x)\\)

\\(\text{let } f(x)=u(g(x))=(g(x))^2 \space \text{, and } g(x) = 4x^2+1\\)

\\(\text{diffrentiate using first principles,}\\)

\\(u\prime(g(x)) = \frac{du}{dg} = 2 \cdot g(x) = 2 \cdot (4x^2 + 1) = 8x^2 + 2\\)

\\(g\prime(x) = \frac{dg}{dx} = 8x\\)

\\(\text{apply chain rule,}\\)

\\(f\prime(x) = u\prime(g(x)) \cdot g\prime(x) = \frac{du}{dg} \cdot \frac{dg}{dx} = (8x^2 + 2) \cdot 8x\\)

\\(\boxed{f\prime(x) = 64x^3 + 16x}\\)