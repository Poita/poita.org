---
layout: post
title: Derivation of Smoothstep
tags:
- smoothstep
- math
- calculus
- derivative
- polynomials
---
Here's the formula for [smoothstep][1]:
`\[ f(t) = 3t^2 - 2t^3 \]`
but where does that come from?

Oddly, when you search for "derivation of smoothstep", none of the results are
able to answer that question. So here's my contribution to the Internet: a
simple derivation of smoothstep. (To be honest, I just want an excuse to test
out [MathJax][5] on my blog).

Here's what smoothstep looks like:

<br/>
<center><img src='/img/smoothstep.png' alt='smoothstep' width='252' height='249' /></center>
<br/>

So, to start, what are we actually looking for? What are the requirements of
smoothstep? Well, it starts at 0 when t is 0 and ends at 1 when t is 1, and at
both those points the curve is flat, i.e. the [derivative][2] is zero. Formally:

`\[
\begin{aligned}
f(0) &= 0 \\
f(1) &= 1 \\
f'(0) &= 0 \\
f'(1) &= 0 \end{aligned}
\]`

That's a good start, but there's infinite functions that satisfy those equations.
We need to restrict ourselves further. One way to do that would be to assume that
the solution is a polynomial, and since there are two points where the
derivative is zero we might as well assume a 3-degree polynomial (in general, a
*n*-degree polynomial has at most *n-1* [critical points][4] i.e points where the
derivative is zero).

`\[ f(t) = at^3 + bt^2 + ct + d \]`

and the derivative is

`\[ f'(t) = 3at^2 + 2bt + c \]`

Substituting in the values from our formal requirements gives us a nice system
of linear equations:

`\[
\begin{aligned}
f(0) = 0 &\implies d = 0 \\
f(1) = 1 &\implies a + b + c = 1 \\
f'(0) = 0 &\implies c = 0 \\
f'(1) = 0 &\implies 3a + 2b = 0 \end{aligned}
\]`

Solving that system of equations (manual [elimination of variables][3] is the simplest
way here) gives:

`\[ a = -2, b = 3, c = 0, d = 0 \]`

*Et voila*! Plugging those back into the original polynomial gives us smoothstep.

[1]: http://en.wikipedia.org/wiki/Smoothstep
[2]: http://en.wikipedia.org/wiki/Derivative
[3]: http://en.wikipedia.org/wiki/System_of_linear_equations#Elimination_of_variables
[4]: http://en.wikipedia.org/wiki/Critical_point_(mathematics)
[5]: http://www.mathjax.org/
