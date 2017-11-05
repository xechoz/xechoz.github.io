# Batch & Cache

1. 减少不必要的计算

一般在调用频繁的代码，有些代码是每次都一样的，可以移到其他地方，循环loop， 或重复调用的方法，如 View.onDraw(), View.onMeasure() 等。

onDraw:

```java

// before
public void onDraw(Canvas canvas) {
	paint.setTextColor(textColor); // 固定color
	paint.setTextSize(textSize); // 固定 size
	paint.setxxx(...); // 变动的
	...
}


// after

public void CustomView(...) {
	...
	init();
	...
}

private init() {
	...
	paint.setTextColor(textColor);
	paint.setTextSize(textSize);
	...
}

public void onDraw(Canvas canvas) {
	paint.setxxx(...); // 只留下变动的
	...
}

```


loop:

```java
// before
for (int i=0; i<size; i++) {
	int width = margin.left + margin.right; // 每次都计算

	if (width < MAX_WIDTH) {
		...
	} else {
		...
	}
}


// after, width 只算一次
for (int i=0, width = margin.left + margin.right; i<size; i++) {

	if (width < MAX_WIDTH) {
		...
	} else {
		...
	}
}

```

2. cache 计算结果

```

```

3. 减少方法调用次数

写递归的时候要注意

```java
// 会出现 stack over flow： 性能最差，但易懂
public long computeFibonacci(int positionInFibSequence) {
    if (positionInFibSequence <= 2) {
        return 1;
    } else {
        return computeFibonacci(positionInFibSequence - 1)
                + computeFibonacci(positionInFibSequence - 2);
    }
}

// 会出现 stack over flow：性能一般，比较难懂
public long computeFibonacci(int term, long value, long prev) {
    if (term == 0) {
        return prev;
    }

    if (term == 1) {
        return value;
    }

    return computeFibonacci(term - 1, value+prev, value);
}

/**
 * 性能最好, 比较难懂
 * 
 * calculate from fib(0), fib(1) , fib(2), fib(3)...  until fib(term)
 */
public long fib(int term) {
    if (term == 0) {
        return 0;
    }

    if (term == 1) {
        return 1;
    }

    long a = 0;
    long b = 1;

    long temp = a+b;

    for (int i=2; i <= term; i++) {
        temp = a+b; // store next result
        a = b; // move b to a
        b = temp; // b now stores the new result
    }

    return temp;
}
```

# Doc

1. [Udacity: Android Performance](https://classroom.udacity.com/courses/ud825/lessons/3764448604/concepts/37310891220923)