---
title: kubernetes资源换算
date: 2017-11-17 15:18:35
tags:
  - quota
  - 资源
  - 单位
categories:
  - kubernetes
---
## CPU单位的计算

cpu单位计算比较简单，因为cpu只有单位m或者没有单位。换算为：1CPU=1000m。

## Memory的计算

Memory的换算相对要复杂一些。Memory的单位分为两类：

1. 两个单位间相差10的三次方的单位：E，P，T，G，M，K，k，m，u，n。
2. 两个单位间相差2的十次方的单位：Ei, Pi, Ti, Gi, Mi, Ki。

## 换算函数
下面提供一个简单的内存换算函数：

```
	/**
	 * convertMemory:转换内存值字符串为无单位数值. <br/>
	 *
	 * @param memory
	 * @return double
	 */
	public static double parseMemory(String memory) {
		if (StringUtils.isEmpty(memory)) {
			return 0d;
		}
		memory = memory.replaceAll("i", "");
		double ret;
		if (memory.endsWith("n")) {
			ret = Double.parseDouble(memory.replace("n", "")) * Math.pow(10, -9);
		} else if (memory.endsWith("u")) {
			ret = Double.parseDouble(memory.replace("u", "")) * Math.pow(10, -6);
		} else if (memory.endsWith("m")) {
			ret = Double.parseDouble(memory.replace("m", "")) * Math.pow(10, -3);
		} else if (memory.endsWith("k")) {
			ret = Double.parseDouble(memory.replace("k", "")) * Math.pow(10, 3);
		} else if (memory.endsWith("K")) {
			ret = Double.parseDouble(memory.replace("K", "")) * Math.pow(10, 3);
		} else if (memory.endsWith("M")) {
			ret = Double.parseDouble(memory.replace("M", "")) * Math.pow(10, 6);
		} else if (memory.endsWith("G")) {
			ret = Double.parseDouble(memory.replace("G", "")) * Math.pow(10, 9);
		} else if (memory.endsWith("T")) {
			ret = Double.parseDouble(memory.replace("T", "")) * Math.pow(10, 12);
		} else if (memory.endsWith("P")) {
			ret = Double.parseDouble(memory.replace("P", "")) * Math.pow(10, 15);
		} else if (memory.endsWith("E")) {
			ret = Double.parseDouble(memory.replace("E", "")) * Math.pow(10, 18);
		} else if (memory.endsWith("Ki")) {
			ret = Double.parseDouble(memory.replace("Ki", "")) * Math.pow(2, 10);
		} else if (memory.endsWith("Mi")) {
			ret = Double.parseDouble(memory.replace("Mi", "")) * Math.pow(2, 20);
		} else if (memory.endsWith("Gi")) {
			ret = Double.parseDouble(memory.replace("Gi", "")) * Math.pow(2, 30);
		} else if (memory.endsWith("Ti")) {
			ret = Double.parseDouble(memory.replace("Ti", "")) * Math.pow(2, 40);
		} else if (memory.endsWith("Pi")) {
			ret = Double.parseDouble(memory.replace("Pi", "")) * Math.pow(2, 50);
		} else if (memory.endsWith("Ei")) {
			ret = Double.parseDouble(memory.replace("Ei", "")) * Math.pow(2, 60);
		} else {
			ret = Double.parseDouble(memory);
		}
		return ret;
	}
```