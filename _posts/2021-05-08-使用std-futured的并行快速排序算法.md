---
title: '使用std::futured的并行快速排序算法'
date: 2021-05-08 01:15:45 +0800
tags: [C++]
categories: [C++]
math: true
memarid: true
---

快速排序算法我们或多或少都曾了解过。但我们使用的一般都是单线程的版本，对于小规模的数据，尚且可以应付。而对于大规模的数据，单线程的版本就有些力不从心，就为了研究并行的快速排序算法，我们不妨先回顾一下传统的快速排序算法。下面首先用Haskell简单阐述一下基本的快排原理。

代码如下：

```haskell
quicksort [] = []
quicksort (p:xs) = (quicksort lesser) ++ [p] ++ (quicksort greater)
  ``where
    ``lesser = ``filter` `(< p) xs
    ``greater = ``filter` `(>= p) xs
```

为了实现快速排序算法，我们需要选择一个基准元素p，将当前数组分为两个子数组，其中lesser数组里面存放小于基准元素p的所有元素，greater存放剩下的元素。递归执行。

仿照上述Haskell代码，我们写出以下C++代码：

```c++
template <typename T>
void quick_sort(vector<T>& v) {
  if (v.size() <= 1) return;
  auto start_it = v.begin();
  auto end_it = v.end();

  const T pivot = *start_it;

//partition the list
  vector<T> lesser;
  copy_if(start_it, end_it, std::back_inserter(lesser),
          [&](const T& el) { return el < pivot; });

  vector<T> greater;
  copy_if(start_it + 1, end_it, std::back_inserter(greater),
          [&](const T& el) { return el >= pivot; });

//solve subproblems
  quick_sort(lesser);
  quick_sort(greater);

//merge
  std::copy(lesser.begin(), lesser.end(), v.begin());
  v[lesser.size()] = pivot;
  std::copy(greater.begin(), greater.end(),
            v.begin() + lesser.size() + 1);
}

```

在C++中，函数式编程常用的filter函数对应于<algorithm>中的copy_if函数。定义如下：

```
template <class InputIterator, class OutputIterator, class UnaryPredicate>
  OutputIterator copy_if (InputIterator first, InputIterator last,
                          OutputIterator result, UnaryPredicate pred);
```

前两个参数分别是容器的头尾迭代器，第三个参数是输出结果的迭代器，最后一个是判断条件。在上述quick_sort函数中，我们用lambda实现了对数组各元组与基准元素的判断。再用递归对子数组调用算法。最后把各个子数组进行合并。

那么如何用并行的方式去计算快排呢？

我们很容易分析出，在每一个层递归的函数中，计算lesser与greater是完全独立的两个过程。根据这个原理，我们就可以设计我们的并行快排。

我们尝试用C++的<future>去实现这种思想

```c++
template <typename T>
void filter(const vector<T>& v, vector<T>& lesser, const int pivot) {
  for (const auto el : v) {
    if (el < pivot) lesser.push_back(el);
  }
}

template <typename T>
void quick_sort(vector<T>& v) {
  if (v.size() <= 1) return;
  auto start_it = v.begin();
  auto end_it = v.end();

  const T pivot = *start_it;
  vector<T> lesser;
  auto fut1 = std::async([&]() {
    filter<T>(std::ref(v), std::ref(lesser), pivot);
    quick_sort<T>(std::ref(lesser));
  });


  vector<T> greater;
  copy_if(start_it + 1, end_it, std::back_inserter(greater),
          [&](const T& el) { return el >= pivot; });

  quick_sort(greater);

  fut1.wait();

  std::copy(lesser.begin(), lesser.end(), v.begin());
  v[lesser.size()] = pivot;
  std::copy(greater.begin(), greater.end(),
            v.begin() + lesser.size() + 1);
}
```

上述并行算法在每一次将小于基准元素的数组放到lesser的操作时，创建了一个新的线程，并在当前的线程执行greater的操作。为了更明白地展示算法，线程与子数组的关系参考下图

![quick_sort](http://www.davidespataro.it/wp-content/uploads/2019/03/graph_quicksort_future-1.png)

其中虚线代表子数组由分配的新线程计算，而实线代表子数组由当前线程。

最后在执行完greater后等待子线程完成计算并合并。

从理论上来说，我们已经通过引入多个线程并行计算达到节约计算时间的效果，但实际上对于较小的数组来说，上述并行算法反而会耗时更多，原因在于开辟新的线程并维护上下文的代价是很大的。上述算法即使在子问题很小的时候也会开辟新的线程，这是很耗时也没有必要的。事实上一定是有这样一个阈值，当子问题规模小于该阈值时，开辟新线程异步的代价要比在单一线程计算要耗时。

一个简单但实用的解决方法时，我们不妨设置一个阈值，小于该阈值就用单线程计算就好了。具体代码如下：
```c++
template <typename T>
void filter(const vector<T>& v, vector<T>& lesser, const int pivot) {
	for (const auto el : v) {
		if (el < pivot) lesser.push_back(el);
	}
}

template <typename T>
void quick_sort_async_lim(vector<T>& v) {
	if (v.size() <= 1) return;
	auto start_it = v.begin();
	auto end_it = v.end();


template <typename T>
void filter(const vector<T>& v, vector<T>& lesser, const int pivot) {
	for (const auto el : v) {
		if (el < pivot) lesser.push_back(el);
	}
}

template <typename T>
void quick_sort_async_lim(vector<T>& v) {
	if (v.size() <= 1) return;
	auto start_it = v.begin();
	auto end_it = v.end();

	const T pivot = *start_it;
	vector<T> lesser;

	vector<T> greater;
	copy_if(start_it + 1, end_it, std::back_inserter(greater),
		[&](const T& el) { return el >= pivot; });

	if (v.size() >= THRESHOLD) {
		auto fut1 = std::async([&]() {
			filter<T>(std::ref(v), std::ref(lesser), pivot);
			quick_sort_async_lim<T>(std::ref(lesser));
		});

		quick_sort_async_lim(greater);
		fut1.wait();

	}
	else {
		//子问题很小，没必要开辟新的线程
		copy_if(start_it, end_it, std::back_inserter(lesser),
			[&](const T& el) { return el < pivot; });
		quick_sort_async_lim(lesser);
		quick_sort_async_lim(greater);
	}

	std::copy(lesser.begin(), lesser.end(), v.begin());
	v[lesser.size()] = pivot;
	std::copy(greater.begin(), greater.end(),
		v.begin() + lesser.size() + 1);
}

```
 现在，我们的并行快排算法在处理问题的时候就可以充分应用到多线程的优势了。当然，我们目前的代码依然还有很多可以改进的地方，比如我们可以限制我们的最大线程数小于物理上CPU支持的最大线程数（对于支持超线程技术的CPU来说，一般为核心数X2），还可以对于子问题进行进一步的划分，比如对于非常小的子问题不采用快排而采用插入排序等。但无论如何，我们的核心思想至少已经得到了体现。

