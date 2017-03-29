---
layout: post
comments: true
tags: lua rexx z/os
title: Lua vs REXX shootout
---

<table>
<tr>
<td>
   <pre lang="csharp">
   const int x = 3;
   const string y = "foo";
   readonly Object obj = getObject();
   </pre>
</td>
<td>
  <pre lang="nemerle">
  def x : int = 3;
  def y : string = "foo";
  def obj : Object = getObject();
  </pre>
</td>
<td>
  Variables defined with <code>def</code> cannot be changed once defined. This is similar to <code>readonly</code> or <code>const</code> in C# or <code>final</code> in Java. Most variables in Nemerle aren't explicitly typed like this.
</td>
</tr>
