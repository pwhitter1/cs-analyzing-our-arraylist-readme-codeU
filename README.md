# cs-analyzing-our-arraylist-readme


## Overview

This README kills two birds with one stone: we present solutions to the previous lab, and also use them to demonstrate analysis of algorithms.


## Objectives

1.  Read and understand solutions to the previous lab.
2.  Classify the methods in `MyArrayList`.
3.  Use amortized analysis to classify appropriate algorithms.


## Classifying MyArrayList methods

For many methods, we can identify the order of growth by examining the code.  For example, here's the implementation of `get` from `MyArrayList`:

```java
	public E get(int index) {
		if (index < 0 || index >= size) {
			throw new IndexOutOfBoundsException();
		}
		return array[index];
	}
```

Everything in `get` is constant time, so `get` is constant time.  No problem.

Now that we've classified `get`, we can classify `set`, which uses it.  Here is my implementation of `set` from the previous lab:

```java
	public E set(int index, E element) {
		E old = get(index);
		array[index] = element;
		return old;
	}
```

One slightly clever part of this solution is that it does not check the bounds of the array explicitly; it takes advantage of `get`, which raises an exception if the index is invalid.

Everything in `set`, including the invocation of `get`, is constant time, so `get` is also constant time.

Next we'll look at some linear methods.  For example, here's my implementation of `indexOf`:

```java
	public int indexOf(Object target) {
		for (int i = 0; i<size; i++) {
			if (equals(target, array[i])) {
				return i;
			}
		}
		return -1;
	}
```

Each time through the loop, `indexOf` invokes `equals`, so we have to classify `equals` first.  Here it is:

```java
	private boolean equals(Object target, Object element) {
		if (target == null) {
			return element == null;
		} else {
			return target.equals(element);
		}
	}
```

This method invokes `target.equals`; the runtime of this method might depend on the size of `target` or `element`, but it probably doesn't depend on the size of the array, so we consider it constant time for purposes of analyzing `indexOf`.

Getting back to `indexOf`, everything inside the loop is constant time, so the next question we have to consider is: how many times does the loop execute?

If we get lucky, we might find the target object right away and return after testing only one element.  If we are unlucky, we might have to test all of the elements.  On average we expect to test half of the elements, so this method is considered linear except in the (unlikely) case that we know the target element is at the beginning of the array.

The analysis of `remove` is similar.  Here's my implementation:

```java
	public E remove(int index) {
		E e = get(index);
		for (int i=index; i<size-1; i++) {
			array[i] = array[i+1];
		}
		size--;
		return e;
	}
```

It uses `get`, which is constant time, and then loops through the array, starting from `index`.  If we remove the element at the end of the list, the loop never runs and this method is constant time.  If we remove the first element, we loop through all of the remaining elements, which is linear.  So, again, this method is considered linear except in the special case where we know the element is at the end.



## Classifying `add`

In the previous lab, you wrote a version of `add` that takes an index and an element as parameters.  Here's my solution:

```java
	public void add(int index, E element) {
		if (index < 0 || index > size) {
			throw new IndexOutOfBoundsException();
		}
		// add the element to get the resizing
		add(element);
		
		// shift the other elements
		for (int i=size-1; i>index; i--) {
			array[i] = array[i-1];
		}
		// put the new one in the right place
		array[index] = element;
	}
```

This version of `add` uses the other version of `add`, which puts the new element at the end.  Then it shifts the other elements to the right, and puts the new element in the right place.

To distinguish the two versions, I'll call them `add(E)` and `add(int, E)`.
Before we can classify `add(int, E)`, we have to classify `add(E)`:

```java
	public boolean add(E e) {
		if (size >= array.length) {
			// make a bigger array and copy over the elements
			E[] bigger = (E[]) new Object[array.length * 2];
			System.arraycopy(array, 0, bigger, 0, array.length);
			array = bigger;
		} 
		array[size] = e;
		size++;
		return true;
	}
```

This version turns out to be hard to analyze.  If there is an unused spaces in the array it is constant time, but if we have to resize the array, it's linear.  So which is it?

We can classify this method by thinking about the average number of operations per add over a series of n adds.
For simplicity, assume we start with an array that has room for 2 elements.

* The first time we call add, it finds unused space in the array, so it stores 1 element.
* The second time, it finds unused space in the array, so it stores 1 element.
* The third time, we have to resize the array, copy 2 elements, and store 1 element.  Now the size of the array is 4.
* The fourth time stores 1 element.
* The fifth time resizes the array, copies 4 elements, and stores 1 element.  Now the size of the array is 8.
* The next 3 adds store 3 elements.
* The next one copies 8 and stores 1.  Now the size is 16
* The next 7 adds store 7 elements.

And so on.  Adding things up:

* After 4 adds, we've stored 4 elements and copied 2.
* After 8 adds, we've stored 8 elements and copied 6.
* After 16 adds, we've stored 16 elements and copied 14.

By now you should see the pattern: to do n adds, we have to store n elements and copy n-2.  So the total number of operations is 2n-2.  To get the average number of operations per add, we dividing the total by n; the result is 2 - 2/n, which means we can think of `add` as constant time.  

If might seem strange that an algorithm that is sometimes linear can be constant time, on average.  The key is that we double the length of the array each time it gets resized.  That limits the number of times each element gets copied.  Otherwise — if we add a fixed amount to the length of the array, rather than multiplying by a fixed amount — the analysis doesn't work.

This way of classifying an algorithm, by computing the average time in a series of invocations, is called [amortized analysis](https://en.wikipedia.org/wiki/Amortized_analysis).  The idea is that the extra cost of copying the array is spread, or "amortized", over a series of invocations.

Now, if `add(E)` is constant time, what about `add(int, E)`?  After calling `add(E)`, it loops through part of the array and shifts elements.  This loop is linear, except in the special case where we are adding at the end of the list.  So `add(int, E)` is linear.

This is an example of a tricky API, where two methods that seem similar have qualitatively different performance.


## Classifying `removeAll`

The last example we'll consider is `removeAll`; here's the implementation in `MyArrayList`:

```java
	public boolean removeAll(Collection<?> c) {
		boolean flag = true;
		for (Object o: c) {
			flag &= remove(o);
		}
		return flag;
	}
```

Each time through the loop, `removeAll` invokes `remove`, which is linear.  So it is tempting to think that `removeAll` is quadratic.  But that's not necessarily the case.

In this method, the loop runs once for each element in the Collection `c`.  If `c` contains m elements and the list contains `n` elements, this method is in O(nm).  If the size of the collection can be considered constant, `removeAll` is linear.  But if the size of the collection is proportional to n, it's quadratic.  For example, if `c` always contains 100 or fewer elements, `removeAll` is linear.  But if `c` generally contains 1% of the elements in the list, `removeAll` is quadratic.

Sometimes when we talk about "problem size" we have to be careful about which size, or sizes, we are talking about.


## Resources

[Amortized analysis](https://en.wikipedia.org/wiki/Amortized_analysis): Wikipedia page.
