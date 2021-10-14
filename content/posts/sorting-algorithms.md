---
title: "排序算法"
date: 2020-02-09T21:20:50+08:00
draft: false
tags: ["algorithm"]
---

记得很早之前的一个同事新人，跟我开始了这样一次对话：

“你知道Python里面的排序用的是什么算法吗？”

“肯定是快排。“，我说。

”不对，是叫Timsort，是一个叫Tim的人开发的“

“就算现在不是快排，最开始也肯定是快排”。

我加强语气来掩饰自己对这个问题其实并不了解，直到很久以后我才知道，Timsort其实是的归并排序和插入排序的混合。

每个（只）学过一点算法的人，大概都会有快排是最好的排序算法的印象。最近重读《编程珠玑》，在第一章里，作者提到了最开始打算用归并排序去解决一个大文件的排序问题，深入分析问题后，发现可以利用bitmap，在O(n)的时间里解决那个具体的排序问题的故事。我的第一反应是为什么他首先就想到归并排序而不是其他算法呢，借此机会重新复习了一下各种排序算法的特性，以及他们的优劣之处。

考虑一个简单的场景，我们需要把一副扑克牌，从右边移动到左边，同时按照从小到大的顺序排好。最直观的做法就是，按照顺序一张一张移动，每次移动到左边时，我们都会找到合适的位置插入进去，这个方法也就是插入排序的思想。 从它的描述，我们就知道他实现起来非常简单，而且当元素的数量较小或者大部分元素有序的时候，这其实是一个非常有效的算法，不过它在最坏的情况下也是O(n^2)的时间。

和插入排序类似的一个算法就是选择排序，过程不再是直接选择下一个、插入已经排序的元素里面，而是每次都直接从未排序的里面直接选出合适的来移动，所以选择排序的执行时间完全和数据的输入无关，它永远是N^2的，唯一的优势是数据交换的次数最少，不过这个优势好像也没什么用。

插入排序和选择排序都很好理解，也很容易实现，但是大部分时间他们的效率都不是很高，所以还需要一些稍微高级一点的排序算法，快排（Quicksort）就是其中一种。叫快排是因为在同类基于比较的算法中，它确实很快。它的思想是，每次选择一个切分元素（Pivot），把元素分成小于和大于的两份，然后再递归的按同样的方式应用到切分后的两部分。快速排序平均时间是O(NlogN)，虽然在最坏的情况下也需要O(n^2)的时间，不过这种情况很少出现，而且可以通过先打乱数组来避免。快速排序的限制在于内存访问，需要数据可以完全放在内存中，并且可以随机访问。如果数据量太大，或者是像链表这样无法随机访问的数据结构就不适合用快排。

归并排序（Mergesort）和快排一样，是一个分而治之（divide-and-conquer ）的算法。它也是递归的处理数组的两部分，不过排序的过程不在切分，而是在把小的数组归并回大的数组的时候，虽然平均起来它比快排慢一点，但是它可以保证在最坏的情况下也是O(NlogN)的时间，而且不需要随机访问数据，非常适合用来排序存储在外部的数据，比如磁盘或者网络，当然链表更不在话下。不过要实现它需要O（n）的额外空间，这在一些内存比较紧张的嵌入式系统或者移动设备中会比较不适合。

还有一种比较常见的排序就是堆排序（Heapsort），这个堆和内存的那个堆没什么关系，这里是一种数据结构，通常都是二叉堆，也就是一个二叉树，之所以叫堆是因为元素在插入和删除的过程中会保证每个节点元素都不大于父节点（MaxHeap），或者都不小于父节点（MinHeap），所以根节点永远是最大或者最小的那个。因为维护一个堆的数据结构的开销很小，只需要LogN次的操作，所以可以利用堆的这个属性来实现排序。堆排序实现起来也比较简单，能够保证最坏的时间是O（NlogN）而且不需要額外的存储空间，只不过它的比较很少在相邻的元素之间，所以无法有效的利用缓存。

再说回最开始提到的Timsort，它其实是混合了归并排序和插入排序的算法，先把数组分成不同的部分，Timsort里面叫做Run，用插入排序，然后再用归并排序里面的merge方法，把这些Run合并成更大的有序数组。虽然只是基于归并和插入排序，Timsort在实现的过程中做了很多优化，比如用基于二分查找的插入排序，避免归并大小相差太大的子数组，效果也相当不错，根据Wikipedia来看，Timsort现在不仅用在Python中，Java ，Android和Switft也都在用。