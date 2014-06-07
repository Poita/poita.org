---
layout: post
title: Cube Vertex Numbering
tags:
- gamedev
- graphics
- programming
- vertex
- numbering
---
If you do any sort of graphics, physics, or 3D programming in general then
at some point you are going to be working with cubes, and at some point you
are going to need to assign a number to each vertex in those cubes.
Usually this is because you need to store them in an array, or maybe you are
labelling the children of an octree, but often you just need some consistent
way of identifying vertices.

For whatever reason, a surprisingly common way of numbering vertices is
like this, with the bottom vertices numbered in counter-clockwise order
followed by the top vertices in counter-clockwise order
(warning, incoming ASCII art):

<pre>
        7-------6
       /|      /|
      4-+-----5 | 
      | |     | |   y
      | 3-----+-2   | z
      |/      |/    |/
      0-------1     +--x
</pre>

Depending on what you are doing, there's generally only a few applications of
the numbering system:

* Converting a number to a coordinate.
* Converting a coordinate to a number.
* Enumerating adjacent vertex numbers.

Unfortunately, this common numbering scheme makes none of these tasks easy, and
you have no choice but to produce tables of coordinates and adjacency lists for
each vertex number. This is tedious, error-prone, and completely unnecessary.

Here is a better scheme:

<pre>
        3-------7
       /|      /|
      2-+-----6 | 
      | |     | |   y
      | 1-----+-5   | z
      |/      |/    |/
      0-------4     +--x
</pre>

This numbering uses the [Lexicographic Ordering][lex] of the vertices, with 0
being the lexicographically first vertex, and 7 being the last.

<table>
  <tr><th>Number<br/>(decimal)</th><th>Number<br/>(binary)</th><th>Coordinate</th></tr>
  <tr><td>0</td><td>000</td><td>0, 0, 0</td></tr>
  <tr><td>1</td><td>001</td><td>0, 0, 1</td></tr>
  <tr><td>2</td><td>010</td><td>0, 1, 0</td></tr>
  <tr><td>3</td><td>011</td><td>0, 1, 1</td></tr>
  <tr><td>4</td><td>100</td><td>1, 0, 0</td></tr>
  <tr><td>5</td><td>101</td><td>1, 0, 1</td></tr>
  <tr><td>6</td><td>110</td><td>1, 1, 0</td></tr>
  <tr><td>7</td><td>111</td><td>1, 1, 1</td></tr>
</table>

I've included the binary representation of the vertex number in table above,
because it illuminates the key property of this scheme: the coordinate of a
vertex is equal to the binary representation its vertex number.

`\[
\begin{aligned}
coordinate(n) &= ((n \gg 2) \mathrel{\&} 1, (n \gg 1) \mathrel{\&} 1, (n \gg 0) \mathrel{\&} 1) \\
number(x, y, z) &= (x \ll 2) \mathrel{|} (y \ll 1) \mathrel{|} (z \ll 0) \end{aligned}
\]`

(Of course, the zero shifts are unnecessary, but I've added them to highlight
the symmetry.)

This numbering scheme also makes adjacent vertex enumeration easy.
As an adjacent vertex is just a vertex that differs along one axis, we
just need to flip each bit using XOR to get the adjacent vertices:

`\[
\begin{aligned}
adj_x(n) &= n \oplus 100_2 \\
adj_y(n) &= n \oplus 010_2 \\
adj_z(n) &= n \oplus 001_2 \end{aligned}
\]`

It should be clear that this numbering scheme trivially extends into higher
dimension hypercubes by simply using more bits in the representation, and
extending the formulae in obvious ways.

So there you have it. Whenever you are looking to number things and aren't
sure what order to use, start with lexicographic. It usually has nice
encoding and enumeration properties, and if not then it is probably no worse
than any other ordering.

[lex]: http://en.wikipedia.org/wiki/Lexicographical_order