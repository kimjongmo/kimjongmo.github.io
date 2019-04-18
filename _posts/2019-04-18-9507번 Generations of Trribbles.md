---
layout: post
title: 9507번 - Generations of Tribbles
data: 2019-04-18 16:49:00
categories: algorithm
permalink: /algorithm/9507
tags: dp
---

```java
import java.util.Arrays;
import java.util.Scanner;

public class Main {

	static long[] cache = new long[68];
	static long[] answer;
	public static void main(String[] args) {

		Arrays.fill(cache, -1);
		Scanner sc = new Scanner(System.in);
		int t = sc.nextInt();
		answer = new long[t];
		
		int i=0;
		
		while (t-- > 0) {
			long ans = koong(sc.nextInt());
			answer[i++] = ans;
		}
		
		for(long ans : answer)
			System.out.println(ans);

		sc.close();
	}

	static long koong(int n) {
		if (cache[n] != -1)
			return cache[n];
		if (n < 2)
			return 1;
		else if (n == 2) {
			return 2;
		} else if (n == 3) {
			return 4;
		} else {
			long ans = koong(n - 1) + koong(n - 2) + koong(n - 3) + koong(n - 4);
			cache[n] = ans;
			return ans;
		}
	}
}

```

