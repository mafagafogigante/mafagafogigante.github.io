---
layout: post
title: Examining Compiler Output 1
---

In this post I present a short comparison between the machine code generated by
Clang 5.0.0 for two different for loops.

## Sum up to N

The first function sums from 1 to N and returns the result.

### C++

```cpp
int sumUpTo(int n) {
  int x = 0;
  for (int i = 1; i <= n; i++) {
    x += i;
  }
  return x;
}
```

### Output

```asm
sumUpTo(int):
  test  edi, edi
  jle   .LBB0_1
  lea   eax, [rdi - 1]
  lea   ecx, [rdi - 2]
  imul  rcx, rax
  shr   rcx
  lea   eax, [rcx + 2*rdi - 1]
  ret
.LBB0_1:
  xor   eax, eax
  ret
```

As you can see, Clang used a closed formula to evaluate the expression!

### What should it be

```
  (1 + n)(n) / 2
= (n^2 + n) / 2
```

### What Clang does

```
  (n - 1)(n - 2) / 2 + (2n - 1)
= (n^2 - 3n + 2) / 2 + (4n - 2) / 2
= (n^2 + n) / 2
```

It is important to note that the reason why a compiler would rather evaluate
this convoluted expression is that it will not overflow like the first
expression would.

## Sum up to N multiplying by 2

### C++

```cpp
int sumTwiceUpTo(int n) {
  int x = 0;
  for (int i = 1; i <= n; i++) {
    x += 2 * i;
  }
  return x;
}
```

### Output

```
sumTwiceUpTo(int):
  test  edi, edi
  jle   .LBB1_1
  lea   eax, [rdi - 1]
  lea   ecx, [rdi - 2]
  imul  ecx, eax
  and   ecx, -2
  lea   eax, [rcx + 4*rdi - 2]
  ret
.LBB1_1:
  xor   eax, eax
  ret
```

### What is is

```
  (1 + n)(n)
= (n^2 + n)
```

### What Clang does

```
  ((n - 1)(n - 2) & 111...0) + 4n - 2
= (n - 1)(n - 2) + 4n - 2
= n^2 + n
```

Again, a closed formula!

It is interesting to note that the and is completely useless here: it is forcing
the product (which is always even) into a even number. Also, it requires the
multiplication result to be evaluated, so it could add measurable delay.
