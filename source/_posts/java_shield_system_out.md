---
title: Java：屏蔽项目中System.out和System.err
tags:
  - 屏蔽System.out
---

## 屏蔽原因
对性能要求很高的项目来说，项目中最好不要使用System.out和System.err，其会带来一定的性能开销。

## 屏蔽方法
``` java
public static void redirectSystemOutAndErr() {
	System.setOut(new PrintStream(new OutputStream() {
		@Override
		public void write(int b) throws IOException {
		}
	}));
	System.setErr(new PrintStream(new OutputStream() {
		@Override
		public void write(int b) throws IOException {
		}
	}));
}
```
屏蔽的办法即重写PrintStream的write方法，使其无操作。


