---
layout: post
title:  "经纬系联合招聘算法题"
date:   2016-09-07 21:00:00
tags:
    - 算法
    - 动态规划
---

### 1. 最小操作数
正整数 N 从 1 开始， 每次操作可以选择对 N 加 1 或者对 N 乘 2，若想获得正整数 M，最少需要多少个操作。

`#输入#`

输入一个正整数 M，表示期望获得的正整数：1 < M < 65536

`#输出#`

为一个整数，表示得到正整数 M 需要执行的操作数：如果输入数据错误输出 －1

这题采用动态规划的解法。

	import java.util.Scanner;

	/**
	 * Created by ivanchou on 9/7/16.
	 */
	public class JWOne {
	    public static void main(String[] args) {
	        Scanner in = new Scanner(System.in);
	        while (in.hasNext()) {
	            int m = in.nextInt();
	            System.out.println(new JWOne().solve(m));
	        }
	    }

	    public int solve(int m) {
	        if (m <= 1 || m >= 65536) {
	            return -1;
	        }
	        int[] arr = new int[m + 1];
	        arr[0] = arr[1] = 0;
	        for (int i = 2; i <= m; i++) {
	            if (i % 2 == 0) {
	                arr[i] = arr[i / 2] + 1;
	            } else {
	                arr[i] = arr[i - 1] + 1;
	            }
	        }
	        return arr[m];
	    }
	}


### 2. 矩阵输出
给定一个二维数组，请将其中的元素按 Z 字形打印出来。

`*输入*`

输入数据为 1 组，第一行是两个整数 m 和 n（0<m<100, 0<n<100），表示矩阵的行数和列数。下面是 m 行，对应的是 m＊n 的矩阵，矩阵元素是整数。

`*输出*`

从矩阵第一个元素开始，按 Z 字形打印输出该矩阵的元素，元素之间用空格分割。

该题就是状态控制，注意边缘检测。

	import java.util.Scanner;

	/**
	 * Created by ivanchou on 9/7/16.
	 */
	public class JWTwo {
	    public static void main(String[] args) {
	        Scanner in = new Scanner(System.in);
	        int m = in.nextInt();
	        int n = in.nextInt();
	        int[][] arr = new int[m][n];
	        while (in.hasNext()) {
	            for (int i = 0; i < m; i++) {
	                for (int j = 0; j < n; j++) {
	                    arr[i][j] = in.nextInt();
	                }
	            }
	            new JWTwo().solve(arr);
	        }
	    }

	    public void solve(int[][] nums) {
	        int dir;
	        int down = -1;
	        int leftdown = -2;
	        int rightup = 2;
	        int right = 1;
	        int m = nums.length;
	        int n = nums[0].length;
	        int[] arr = new int[m * n];
	        dir = right;
	        int a = 0, b = 0;
	        for (int i = 0; i < m * n; i++) {
	            arr[i] = nums[a][b];
	            if (dir == right) {
	                if (a == 0 && b + 1 != n - 1) {
	                    b++;
	                    dir = leftdown;
	                } else if (a == m - 1 && b + 1 != n - 1) {
	                    b++;
	                    dir = rightup;
	                }
	            } else if (dir == leftdown) {
	                if (a + 1 < m - 1 && b - 1 > 0) {
	                    a++;
	                    b--;
	                    dir = leftdown;
	                } else if (a + 1 == m - 1 && b - 1 >= 0) {
	                    b--;
	                    a++;
	                    dir = right;
	                } else if (a + 1 < m - 1 && b - 1 == 0) {
	                    a++;
	                    b--;
	                    dir = down;
	                }
	            } else if (dir == down) {
	                if (b == 0) {
	                    a++;
	                    dir = rightup;
	                } else if (b == n - 1) {
	                    a++;
	                    dir = leftdown;
	                }
	            } else if (dir == rightup) {
	                if (a - 1 > 0 && b + 1 < n - 1) {
	                    a--;
	                    b++;
	                    dir = rightup;
	                } else if (a - 1 == 0 && b + 1 < n - 1) {
	                    a--;
	                    b++;
	                    dir = right;
	                } else if (a - 1 >= 0 && b + 1 == n - 1) {
	                    a--;
	                    b++;
	                    dir = down;
	                }
	            }
	        }
	        for (int i = 0; i < m * n; i++) {
	            System.out.print(arr[i] + " ");
	        }
	    }
	}



可惜的是第二题时间不够，最后没有调试出来。仅作记录之用，如有侵权，请联系我。