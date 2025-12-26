+++
title = "Revisit Sorting Algorithms in C++"
date = 2024-11-19
draft = false
[taxonomies]
  tags = ["C++", "Algorithm"]
[extra]
  toc = true
  keywords = "C++, Algorithm"
+++

All kinds of sorting algorithms, re-visited!

## Easiest

### Bubble Sort

Bubble sort is the easiest sorting algorithm; it can be considered an inferior version of insertion sort.

You should **never** use bubble sort in real applications. It has no advantages compared to the other sorting algorithms.

The basic concept: traverse the vector and compare `[j]` and `[j-1]`; if they are out of order, swap them and continue. Each pass pushes the largest remaining element to the end, so we can shrink the inner loop range as we go.

```cpp
void bubble_sort(vector<int>& vec) {
    for (int i = 1; i < vec.size(); i++) {
        for (int j = 1; j < vec.size() - i + 1; j++) {
            if (vec[j] < vec[j - 1]) {
                swap(vec[j], vec[j - 1]);
            }
        }
    }
}
```

### Bucket Sort

Bucket sort is also simple and runs in `O(N)` time when the value range is bounded and small.

It is optimal when the sorted value deviation is small (and non-negative in this example).

```cpp
void bucket_sort(vector<int>& vec) {
    // If values are in [0, 99].
    auto bucket = array<int, 100>({0});
    for (auto v : vec) {
        bucket[v]++;
    }
    vec.clear();
    for (auto i = 0; i < 100; i++) {
        for (auto j = 0; j < bucket[i]; j++) {
            vec.push_back(i);
        }
    }
}
```

## Easy But Not Efficient

### Insertion Sort

Insertion sort works well when the input is already nearly sorted and is stable.

It is very similar to bubble sort, except each traversal bubbles values from back to front.

```cpp
void insert_sort(vector<int>& vec) {
    for (auto i = 1; i < vec.size(); i++) {
        for (auto j = i; j > 0 && vec[j - 1] > vec[j]; j--) {
            swap(vec[j - 1], vec[j]);
        }
    }
}
```

It works better than bubble sort although they have the same worst time complexity `O(N^2)`. The heap sort in the next category can be considered as the superior version of it.

### Shell Sort

Shell sort is a gap-based variant of insertion sort. The gap sequence matters; this example uses a short, fixed sequence for simplicity.

There are things like binary insertion sort, which is conceptually similar but still quadratic in the worst case.

```cpp
void shell_sort(vector<int>& vec) {
    auto gaps = vector<int>({4, 2, 1});  // Ciura-like short sequence for demo
    for (auto gap : gaps) {
        for (auto i = 1; i < vec.size(); i++) {
            for (auto j = i; j >= gap && vec[j - gap] > vec[j]; j--) {
                swap(vec[j - gap], vec[j]);
            }
        }
    }
}
```

## Efficient

### Merge Sort

Divide and conquer sort; think recursively:

* Break a vector into 2 vectors (same length).
* Sort the 2 vectors independently so they are in order.
  * If one of the 2 vector is null, return the other vector directly.
* Merge back the 2 pieces with a 2-pointer merge.

A common optimization is to pre-allocate a `buffer` vector to hold merged results and avoid repeated allocations.

```cpp
// buffer is reused to avoid allocations while merging [start, end)
void merge_sort(vector<int>& vec, vector<int>& buffer, int start, int end) {
    if (end - start <= 1) {
        return;
    }
    int mid = start + (end - start) / 2;
    merge_sort(vec, buffer, start, mid);
    merge_sort(vec, buffer, mid, end);

    buffer.clear();
    auto i = start;
    auto j = mid;
    while (i < mid && j < end) {
        if (vec[i] <= vec[j]) {
            buffer.push_back(vec[i++]);
        } else {
            buffer.push_back(vec[j++]);
        }
    }
    copy(vec.begin() + i, vec.begin() + mid, back_inserter(buffer));
    copy(vec.begin() + j, vec.begin() + end, back_inserter(buffer));
    copy(buffer.begin(), buffer.end(), vec.begin() + start);
}
```

Call it like this to ensure the buffer is pre-sized once:

```cpp
vector<int> buffer;
buffer.reserve(vec.size());
merge_sort(vec, buffer, 0, static_cast<int>(vec.size()));
```

You can avoid clearing and copying by alternately swapping the roles of the source and destination buffers during recursion.

```cpp
// src is the current read buffer; dst is the write buffer
void merge_sort(vector<int>& src, vector<int>& dst, int start, int end) {
    if (end - start <= 1) {
        return;
    }
    int mid = start + (end - start) / 2;
    // Recurse with buffers swapped: src -> dst -> src ...
    merge_sort(dst, src, start, mid);
    merge_sort(dst, src, mid, end);

    int i = start;
    int j = mid;
    int out = start;
    while (i < mid && j < end) {
        dst[out++] = (src[i] <= src[j]) ? src[i++] : src[j++];
    }
    copy(src.begin() + i, src.begin() + mid, dst.begin() + out);
    copy(src.begin() + j, src.begin() + end, dst.begin() + out);
}
```

Before calling the optimized version, ensure `dst` starts as a copy of `src` and call `merge_sort(dst, src, 0, src.size())`.

### Heap Sort

Heap sort is very efficient for repeatedly getting min or max. It underpins the `priority_queue` data structure.

Compared to merge sort and quick sort, it has several advantages:

* Requires no extra space, which makes it suitable for large data structures.
* Good for inserting new values at runtime, as well as popping the min/max value.
* An upgraded version of the **insertion sort**, works well with nearly sorted data.

However it is slower than merge sort in most cases for pure sorting. If all we want is sort an array, merge sort often wins.

After the vector is **made into a heap** in a certain order (which takes `O(N)`), reading the min/max value takes `O(1)`.

If the min/max element is popped or overwritten, the maintenance only takes `O(log N)` time.

Heap sort can be described as first making a vector into a heap, then popping the first element one by one. The pop is done by swapping head and tail elements, reducing the vector size by 1, then sinking the new top element.

Heap itself can be seen as a **complete binary tree** data structure. The time complexity for heap sort is very stable.

#### `make_heap()`

We can define a helper function called `sink()`. It sinks a vector node to the correct location on the heap.

Then we sink every value from the vector end to the beginning, creating the **minimum heap** structure.

```cpp
void sink(vector<int>& vec, int n, int i) {
    while (true) {
        int left = 2 * i + 1;
        int right = 2 * i + 2;
        int min_i = i;
        if (left < n && vec[left] < vec[min_i]) {
            min_i = left;
        }
        if (right < n && vec[right] < vec[min_i]) {
            min_i = right;
        }
        if (min_i == i) {
            return;
        }
        swap(vec[i], vec[min_i]);
        i = min_i;
    }
}

void make_heap(vector<int>& vec) {
    if (vec.empty()) {
        return;
    }
    for (int i = static_cast<int>(vec.size()) / 2 - 1; i >= 0; --i) {
        sink(vec, static_cast<int>(vec.size()), i);
    }
}
```

#### `pop_heap()`

`pop_heap()` will move the first element to the end of the vector, then do a heap update so that the heap is still valid.

This process can be taken as:

* Swap the begin and end element, and reduce the size of the `vec` by 1.
  * You can either directly remove the popped element or only move the popped item to the end.
  * The default C++ `pop_heap()` will only move the popped item to the end.
* Sink the new top element so the heap is still valid.
  * If you only moved at the first step, you need to remember the last element location.

Because we need to swap at first, we need an `empty()` check.

```cpp
void pop_heap(vector<int>& vec) {
    if (vec.empty()) {
        return;
    }
    swap(vec[0], vec[vec.size() - 1]);
    sink(vec, vec.size() - 1, 0);
}
```

#### `insert_heap()`

`insert_heap` is like reverse of `pop_heap()`, we append the element to the end of the vector, then bubble up the end element.

Because the bubble-up process doesn't break the heap validity, we don't need to sink afterwards.

For child index at `i` we can use `(i-1)/2` to get the parent index.

```cpp
void insert_heap(vector<int>& vec, int val) {
    vec.push_back(val);
    size_t i = vec.size() - 1;
    while (i > 0) {
        size_t parent = (i - 1) / 2;
        if (vec[parent] > vec[i]) {
            swap(vec[parent], vec[i]);
            i = parent;
        } else {
            break;
        }
    }
}
```

### Quick Sort

Quick sort can be considered as upgraded version of merge sort, although its worst time complexity is `O(N^2)`. It is also based on the **divide and conquer** concept.

However on average it performs better than the merge sort. And the space complexity is `O(N)` for the worst case.

The following code uses **Hoare partition algorithm**, which can be considered as 2 pointer method, and we ping-pong the pivot element to one of the pointer and increment the other pointer, until these 2 pointers across.

Notice it only works when we pick the first element as the pivot element. If random pivot is given, we need to swap the random pivot element and the first element before using this algorithm.

```cpp
int partition(vector<int>& vec, int low, int high) {
    auto pivot = vec[low];
    auto i = low - 1;
    auto j = high + 1;
    while (true) {
        do {
            i++;
        } while (vec[i] < pivot);
        do {
            j--;
        } while (vec[j] > pivot);
        if (i >= j) {
            return j;
        }
        swap(vec[i], vec[j]);
    }
}

void quick_sort(vector<int>& vec, int low, int high) {
    if (low >= high) {
        return;
    }
    int mid = partition(vec, low, high);  // Hoare partition returns a valid split point
    quick_sort(vec, low, mid);
    quick_sort(vec, mid + 1, high);
}
```