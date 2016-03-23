---
layout: post
time: 2015-5-16
title: 仿照SGI-STL实现Vector容器
category: C/C++
keywords: STL, Vector
tags: STL
description: 简单的STL Vector实现 
---

## vector容器
vector容器是一种动态数组的实现，它将数据连续存储在内存空间中并且支持动态增长，在C++中是一个十分常用的容器

## vector容器的设计总体思路

### 内存管理
由于vector容器需要确保数据是在内存中连续存储，所以一般需要预先申请一片连续的空间来顺序的存储数据，当这块空间填满之后就需要申请一块更大的空间，然后把原来的数据重新拷贝到新的空间上以确保数据存储的连续性.

我们的内存预分配策略就是，对于空的vector起始预分配1个数据大小，然后不够的时候翻倍，即2,4,8，....之所以这样分配是为了和前面说的Allocator对应起来，因为freelist指向的都是2的指数次的空间. 另外，如果初始化vector容器时候就有一定个数的数据，比如有k个,那么预留的数据个数就是最小的那个大于等于k且是2的整数次幂的数.

从上面的内存策略可以看到我们需要维持三个指针，一个指针指向目前申请空间的起始地址，一个指向结束地址，一个指向可用的剩余空间的地址.至于空间的申请，则完全交给前面说到的Allocator来处理.

### 迭代器
vector容器也需要一个迭代器以支持很多基于迭代器的算法或者成员函数，一般来说容器的迭代器都是很容器的存储方式有关，由于vector的数据是类似数组一样的顺序存储，所以可以直接用指针来作为迭代器，不必单独定义一个迭代器类.


## vector容器具体代码实现
下面只是vector容器的代码展示，其中还有依赖一些其他函数，具体细节可以查看我的github上Tiny-STL项目的[全部代码](https://github.com/sosohu/Tiny-Stl/blob/master/include/vector.h)．

{% highlight C++ %}
template<typename T, typename Alloc = alloc>
class vector{
private:
	typedef simple_alloc<T, alloc> data_alloc;

public:
	typedef T value_type;
	typedef T* poniter;
	typedef const T* const_pointer;
	typedef T& reference;
	typedef const T& const_reference;
	typedef size_t size_type;
	typedef ptrdiff_t difference_type;

	typedef poniter iterator;

	//基本的构造析构函数
	explicit vector();
	explicit vector(size_type n);
	vector(size_type n, const_reference x);
	vector(const vector<T>& rhs);

	template<typename InputIterator>
	vector(InputIterator begin, InputIterator end);

	~vector();

	//迭代器函数
	iterator begin(){ return start;}
	iterator end(){ return finish; }

	//属性函数
	size_type size() const{ 	return finish - start; }
	size_type max_size() const{ 	return UINT_MAX / sizeof(T);	}
	size_type capacity() const{	return end_of_storage - start;	}
	bool empty() const{	return finish == start;	}
	void resize(size_type n){
		if(capacity() < n){
			reserve(n);
		}else{
			finish = start + n;
		}
	}
	void reserve(size_type n){
		size_type old_sz = capacity();
		if(old_sz < n){
			iterator new_start = data_alloc::allocate(n);
			uninitialized_copy(start, finish, new_start);
			data_alloc::deallocate(start, old_sz);
			start = new_start;
			finish = start + old_sz;
			end_of_storage = start + n;
		}
	}

	//访问函数
	const_reference operator[](size_type pos) const;
	reference operator[](size_type pos);
	value_type at(size_type pos) const;
	value_type front() const;
	value_type back() const;

	//修改函数
	void assign(size_type n, const_reference value);
	template<typename InputIterator>
	void assign(InputIterator begin, InputIterator end);

	void push_back(const const_reference value);
	void pop_back();

	template<typename InputIterator>//pos之前插入
	void insert(iterator pos, InputIterator begin, InputIterator end);

	iterator insert(iterator pos, const_reference value);
	void insert(iterator pos, size_type n, const_reference value);

	iterator erase(iterator pos);
	iterator erase(iterator begin, iterator end);

	void swap(vector<T, Alloc> &rhs);
	void clear();

private:
	template<typename InputIterator>
	void vector_aux(InputIterator begin, InputIterator end, Is_Int);

	template<typename InputIterator>
	void vector_aux(InputIterator begin, InputIterator end, Not_Int);

	template<typename InputIterator>
	void assign_aux(InputIterator begin, InputIterator end, Is_Int);

	template<typename InputIterator>
	void assign_aux(InputIterator begin, InputIterator end, Not_Int);

	template<typename InputIterator>
	void insert_aux(iterator pos, InputIterator begin, InputIterator end, Is_Int);

	template<typename InputIterator>
	void insert_aux(iterator pos, InputIterator begin, InputIterator end, Not_Int);

private:
	iterator start;
	iterator finish;
	iterator end_of_storage;
};

template<typename T, typename Alloc>
vector<T,Alloc>::vector(){
	start = data_alloc::allocate();
	finish = start;
	end_of_storage = start + 1;
}

template<typename T, typename Alloc>
vector<T,Alloc>::vector(size_type n){
	start = data_alloc::allocate(n);
	finish = start;
	end_of_storage = finish;
}

/*
 * Alloc负责为对象分配内存，uninitialized系列函数负责构造对象
 */
template<typename T, typename Alloc>
vector<T, Alloc>::vector(size_type n, const_reference x){
	start = data_alloc::allocate(n);
	cout<<n<<","<<x<<","<<start<<endl;
	hu::uninitialized_fill_n(start, n, x);
	finish = start + n;
	end_of_storage = finish;
}

template<typename T, typename Alloc>
template<typename InputIterator>
void vector<T,Alloc>::vector_aux(InputIterator begin, InputIterator end, Is_Int){
	size_type n = static_cast<size_type>(begin);
	value_type x = static_cast<value_type>(end);
	start = data_alloc::allocate(n);
	uninitialized_fill_n(start, n, x);
	finish = start + n;
	end_of_storage = finish;
}

template<typename T, typename Alloc>
template<typename InputIterator>
void vector<T,Alloc>::vector_aux(InputIterator begin, InputIterator end, Not_Int){
	difference_type n = distance(begin, end);
	start = data_alloc::allocate(n);
	uninitialized_copy(begin, end, start);
	finish = start + n;
	end_of_storage = finish;
}

template<typename T, typename Alloc>
template<typename InputIterator>
vector<T,Alloc>::vector(InputIterator begin, InputIterator end){
	typedef typename traits_int<InputIterator>::is_int Judge;
	vector_aux(begin, end, Judge());
}

template<typename T, typename Alloc>
vector<T,Alloc>::~vector(){
	data_alloc::deallocate(start, capacity());
}

template<typename T, typename Alloc>
typename vector<T,Alloc>::reference vector<T,Alloc>::operator[](size_type pos) {
	return *(start + pos);
}

template<typename T, typename Alloc>
typename vector<T,Alloc>::const_reference vector<T,Alloc>::operator[](size_type pos) const{
	return (const_cast<vector<T, Alloc> >(*this))[pos];
}

template<typename T, typename Alloc>
typename vector<T,Alloc>::value_type vector<T,Alloc>::at(size_type pos) const{
	if(pos >= size())	THROW_OUT_OF_RANGE;
	return *(start + pos);
}

template<typename T, typename Alloc>
typename vector<T,Alloc>::value_type vector<T, Alloc>::front() const{
	if(empty())	THROW_OUT_OF_RANGE;
	return *start;
}

template<typename T, typename Alloc>
typename vector<T,Alloc>::value_type vector<T, Alloc>::back() const{
	if(empty())	THROW_OUT_OF_RANGE;
	return *(finish - 1);
}

template<typename T, typename Alloc>
void vector<T, Alloc>::assign(size_type n, const_reference value){
	resize(n);
	clear();
	uninitialized_fill_n(start, n, value);
	finish = start + n;
}

template<typename T, typename Alloc>
template<typename InputIterator>
void vector<T,Alloc>::assign_aux(InputIterator begin, InputIterator end, Is_Int){
	assign(size_type(begin), end);
}

template<typename T, typename Alloc>
template<typename InputIterator>
void vector<T,Alloc>::assign_aux(InputIterator begin, InputIterator end, Not_Int){
	size_type dis = distance(begin, end);
	resize(dis);
	clear();
	uninitialized_copy(begin, end, start);
	finish = start + dis;
}

template<typename T, typename Alloc>
template<typename InputIterator>
void vector<T, Alloc>::assign(InputIterator begin, InputIterator end){
	//需要判断InputIterator是否为Integer,也采用traits技术
	typedef typename traits_int<InputIterator>::is_int Judge;
	assign_aux(begin, end, Judge());
}

template<typename T, typename Alloc>
void vector<T, Alloc>::push_back(const const_reference value){
	if(finish == end_of_storage){
		size_type old_sz = size();
		size_type new_sz = old_sz == 0? 1 : 2*old_sz;
		resize(new_sz);
	}
	_construct(&*finish, value);
	finish++;
}

template<typename T, typename Alloc>
void vector<T, Alloc>::pop_back(){
	if(empty())
		THROW_OUT_OF_RANGE;
	finish--;
}

template<typename T, typename Alloc>
template<typename InputIterator>
void vector<T, Alloc>::insert_aux(iterator pos, InputIterator begin, InputIterator end, Is_Int){
	insert(pos, static_cast<size_type>(begin), static_cast<value_type>(end));
}

template<typename T, typename Alloc>
template<typename InputIterator>
void vector<T, Alloc>::insert_aux(iterator pos, InputIterator begin, InputIterator end, Not_Int){
	size_type dif = distance(start, pos);
	size_type dis = distance(begin, end);
	reserve(dis + size());
	pos = start + dif;
	//copy from back to front
	_copy_backward(pos, finish, pos + dis);
	uninitialized_copy(begin, end, pos);
	finish = finish + dis;
}

template<typename T, typename Alloc>
template<typename InputIterator>
void vector<T, Alloc>::insert(iterator pos, InputIterator begin, InputIterator end){
	typedef typename traits_int<InputIterator>::is_int Judge;
	insert_aux(pos, begin, end, Judge());
}

template<typename T, typename Alloc>
typename vector<T, Alloc>::iterator vector<T, Alloc>::insert(iterator pos, const_reference value){
	return insert(pos, 1, value);
}

template<typename T, typename Alloc>
void vector<T, Alloc>::insert(iterator pos, size_type n, const_reference value){
	size_type dif = distance(start, pos);
	reserve(size() + n);
	pos = start + dif;
	//copy from back to front
	_copy_backward(pos, finish, pos + n);
	uninitialized_fill_n(pos, n , value);
	finish = finish + n;
}


template<typename T, typename Alloc>
typename vector<T, Alloc>::iterator vector<T, Alloc>::erase(iterator pos){
	return erase(pos, pos+1);
}

template<typename T, typename Alloc>
typename vector<T,Alloc>::iterator vector<T, Alloc>::erase(iterator begin, iterator end){
	iterator pos;
	// copy from front to back
	_copy_forward(end, finish, begin);
	finish = finish - (end - begin);
	return pos;
}

template<typename T, typename Alloc>
void vector<T, Alloc>::swap(vector<T, Alloc> &rhs){
	std::swap(start, rhs.start);
	std::swap(finish, rhs.finish);
	std::swap(end_of_storage, rhs.end_of_storage);
}

template<typename T, typename Alloc>
void vector<T, Alloc>::clear(){
	finish = start;
}
{% endhighlight %}

### 注意
这里需要重点介绍一下代码中出现的几个*_aux私有成员函数，可以看到它们的意义就是为了实现int trait(int类型萃取)

以assign成员函数为例,我们可以看到有两个重载的assign成员函数:
{% highlight C++ %}
void assign(size_type n, const_reference value);
template<typename InputIterator>
void assign(InputIterator begin, InputIterator end);
{% endhighlight %}
一个为当前vector assign n个value,一个是将[begin, end)里的数据assign给当前vector,它们两个似乎没有什么冲突，但是会有什么问题呢？

我们可以看到第一个assign函数的第一个参数是size_type即unsigned int，第二个参数是const T; 我们可以想一下，如果这个T是int,那么我们如果使用assgin(3, 3)本意调用第一个assgin函数，那么就是形参都是int,所以会将InputIterator解释为int,**然后匹配第二个assgin函数!!!**除非我们assgin((unsigned int)3, 3)这样调用才会匹配到第一个assgin函数，这当然是不方便的.所以我们需要针对这种情况判断InputIterator是否是int类型，如果是则再调用第一个assgin函数. 同理，我们可以看到vector的迭代器构造函数，以及insert函数都有这样的问题，也需要这样解决.
