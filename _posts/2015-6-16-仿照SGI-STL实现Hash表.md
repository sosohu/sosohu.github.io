---
layout: post
time: 2015-6-16
title:	仿照SGI-STL实现Hash表
category: C/C++
keywords: STL, Hash Table
tags: STL
description: 简单的STL Hash Table实现 
---

##Hash Table
Hash表是一个特点很突出的底层容器，虽然STL中没有专门的叫做Hash表的容器，但是像unordered_map和unordered_set这些标准容器都是以Hash表为底层容器实现的．Hash表具有平均O(1)的元素访问能力，包括存取数据和查找数据，相比于vector容器它的有点就是O(1)的时间复杂度查找数据．Hash表的缺点也就是存放的数据与存入的顺序没有关系而且无法获得存放的数据之间的大小关系.

Hash表的实现主要就是通过将键值key通过hash函数映射成存放位置的坐标然后存放数据，这里可能会出现不同的key映射到同一个坐标的情况，对于这种情况的处理方式一般有线性检测，二次检测以及开链法等手段，我们这里SGI中的冲突避免策略就是采用的开链法，即hash出来的相同key按照链表的组织方式挂载到对应位置的槽上.

##Hash Table设计思路
###Hash Table类的原型
{% highlight C++ %}
template<typename Key, typename Value, typename HashFunc = hash_function<Key>,
		typename ExtractKey = extract_key_single<Key>,
		typename EqualKey = equal_key<Key>, typename Alloc = alloc>
class hash_table{
	...
}
{% endhighlight %}
这里需要解释一下,Key指的就是键的类型,Value指的是存储的数据类型，对于`unordered_set<int>`这种，对应过来的Value就是int,而对于`unordered_map<int, double>`这种对应过来的Value就是`pair<int,double>`. HashFunc即hash仿函数类.EqualKey也是一个仿函数类，它的作用就是从Value类型中提取出Key类型对应的值,比如Value如果是int,那么提取出int,如果Value是`pair<int,double>`那么就是提取出int,这在后面的unordered_map类可以再仔细看一下.EqualKey也是一个仿函数类，用来判断两个key是否相等．Alloc类即是负责空间分配的类
###数据组织形式
由于我们采用开链法，所以我们需要先预申请k个(一般是个素数)顺序存储的槽(bucket),然后每个槽存放一个链表指针用来索引hash到该槽的所有数据.然后每次添加数据时候都是先hash到对应的槽然后加入到该槽链表的尾端；如果是查找某个key也是同样hash到对应的槽然后遍历该槽的链表查看是否存在该key的数据.

这里贴一下链表和槽的数据结构代码实现:
{% highlight C++ %}
//链表节点
template<typename Value>
struct hash_node{
	hash_node *next;
	Value val;
	hash_node(hash_node *next, Value val) : next(next), val(val){}
	hash_node() = default;
};

template<typename Key, typename Value, typename HashFunc = hash_function<Key>,
		typename ExtractKey = extract_key_single<Key>,
		typename EqualKey = equal_key<Key>, typename Alloc = alloc>
class hash_table{
...
typedef hash_table<Key, Value, HashFunc, ExtractKey, EqualKey, Alloc>	hash_table_type;
...
//槽
vector<hash_node_type*, Alloc> buckets;
...
}
{% endhighlight %}

另外，为了避免冲突发生率太高,当hash表的**负载因子**(表中存放的数据总数/槽的个数)超过一定阀值(通常是1)就需要扩大槽的数目(一般是原槽数的2倍左右的一个素数).具体扩大的方法就是，申请新的槽，然后把原来槽中存放的数据全部重新hash到新槽对应的链表中，然后释放原来的槽及数据.

###迭代器设计
根据上面的数据组织形式，我们可以让迭代器关联到槽中链表上的链表节点，然后根据需要后移所在链表中的位置或者转移到下一个槽中的链表头部.所以迭代器类需要两个信息，一个是Hash Table对象指针以便更换槽;另一个是当前所在的链表节点.

这里贴上迭代器的设计代码:
{% highlight C++ %}
template<typename Key, typename Value, typename HashFunc,
			typename ExtractKey, typename EqualKey, typename Alloc = alloc>
struct hash_table_iterator{
public:
	typedef Key key_type;
	typedef Value value_type;
	typedef Value* pointer;
	typedef Value &reference;
	typedef hash_table_iterator<Key, Value, HashFunc, ExtractKey, EqualKey, Alloc> iterator;
	typedef const iterator const_iterator;
	typedef hash_table<Key, Value, HashFunc, ExtractKey, EqualKey, Alloc> hash_table_type;
	typedef hash_node<Value> hash_node_type;
	typedef forward_iterator_tag iterator_category;

public:
	hash_table_iterator() = default;
	hash_table_iterator(hash_node_type *cur, hash_table_type* ht) : cur(cur), ht(ht){}

	reference operator*(){
		return cur->val;
	}

	pointer operator->(){
		return &(cur->val);
	}

	iterator operator++(){
		hash_node_type *old = cur;
		cur = cur->next;
		if(cur == NULL){
			size_t id = ht->getBucketId(old->val);
			id++;
			while(id < ht->getBucketSize()){
				cur = ht->buckets[id++];
				if(cur)	break;
			}
		}
		return *this;
	}
	iterator operator++(int){
		iterator tmp = *this;
		++*this;
		return tmp;
	}
	bool operator==(const_iterator &rhs) const{
		return	cur == rhs.cur;
	}
	bool operator!=(const_iterator &rhs) const{
		return cur != rhs.cur;
	}

public:
	hash_table_type *ht; //Hash Table对象指针
	hash_node_type	*cur; //当前链表节点
};
{% endhighlight %}

##Hash Table具体设计
具体代码如下面所示，其实完整的工程代码请查看我的github上的[Tiny-STL](https://github.com/sosohu/Tiny-Stl/blob/master/include/hash_table.h)项目.
{% highlight C++ %}
static const int prime_table_size = 28;
static const unsigned int prime_table[prime_table_size] = {
		53, 97, 193, 389, 769,
		1543, 3079, 6151, 12289, 24593,
		49157, 98317, 196613, 393241, 786433,
		1572869, 3145739, 6291469, 12582917, 25165843,
		50331653, 100663319, 201326611, 402653189, 805306457,
		1610612741, 3221225473ul, 4294967291ul
};

/*
 * ExtractKey tells you how to get the key from the value. For example, Value type is pair<int, int> hash_map and
 * the ExtractKey is select the first one of the pair.
 */
template<typename Key, typename Value, typename HashFunc = hash_function<Key>,
		typename ExtractKey = extract_key_single<Key>,
		typename EqualKey = equal_key<Key>, typename Alloc = alloc>
class hash_table{
	template<typename K, typename V, typename H, typename Ex, typename Eq, typename Al>
	friend  struct hash_table_iterator;
	template<typename Ku, typename Hu, typename Eu, typename Au>
	friend class unordered_set;
	template<typename Km, typename Mapped, typename Hm, typename Em, typename Am>
	friend class unordered_map;
public:
	typedef Key Key_type;
	typedef Value Value_type;
	typedef size_t size_type;
	typedef hash_table<Key, Value, HashFunc, ExtractKey, EqualKey, Alloc>	hash_table_type;
	typedef hash_table_iterator<Key, Value, HashFunc, ExtractKey, EqualKey, Alloc> iterator;
	typedef hash_node<Value> hash_node_type;

	hash_table(size_type n = prime_table[0]) : buckets(vector<hash_node_type*, Alloc>(n, NULL)),
			num_element(0), prime_pos(0), load_factor_gate(1.0), finish(NULL, this){}

	~hash_table(){
		clear();
	}

	size_type getBucketSize() const	{	return buckets.size(); }
	size_type maxBucketSize()	const	{	return UINT_MAX;	}
	size_type getBucketId(const Value_type val)	const{
		return hash_func(get_key(val)) % getBucketSize();
	}
	size_type getBucket(const size_type n)	const{
		hash_node<Value_type> *pCur = buckets[n];
		size_type num = 0;
		while(pCur){
			num++;
			pCur = pCur->next;
		}
		return num;
	}
	bool empty()	{	return num_element == 0;	}
	size_type size()	{	return num_element;	}
	size_type max_size()	{	return UINT_MAX;	}
	double load_factor()	{	return num_element * 1.0 / getBucketSize(); }
	double	max_load_factor()	{	return load_factor_gate;	}
	void max_load_factor(double lfg)	{	load_factor_gate = lfg;	}

	// iterator interface
	iterator begin(){
		iterator start = iterator(NULL, this);
		for(size_type i = 0; i < buckets.size(); i++)
			if(buckets[i]){
				start = iterator(buckets[i], this);
				break;
			}
		return start;
	}
	iterator end() const { return finish; }

	size_type count(const Key_type &vale);
	iterator find(const Key_type &vale);
	void insert_equal(const Value_type value);
	void insert_unique(const Value_type value);
	void erase(iterator iter);
	void erase(const Key_type key);
	void clear();
	void swap(hash_table_type &rhs);

private:
	void resize(const size_type sz);

private:
	vector<hash_node_type*, Alloc> buckets;
	size_t num_element; // count how many keys have been contained
	size_t prime_pos;
	double	load_factor_gate;
	ExtractKey get_key;
	HashFunc hash_func;
	EqualKey equal;
	iterator finish;
};

template<typename Key, typename Value, typename HashFunc, typename ExtractKey, typename Equal, typename Alloc>
typename hash_table<Key, Value, HashFunc, ExtractKey, Equal, Alloc>::size_type
hash_table<Key, Value, HashFunc, ExtractKey, Equal, Alloc>::count(const Key_type &key){
	hash_node_type *pCur = buckets[hash_func(key) % getBucketSize()];
	size_type num = 0;
	while(pCur){
		if(equal(get_key(pCur->val), key))	num++;
		pCur = pCur->next;
	}
	return num;
}

template<typename Key, typename Value, typename HashFunc, typename ExtractKey, typename Equal, typename Alloc>
typename hash_table<Key, Value, HashFunc, ExtractKey, Equal, Alloc>::iterator
hash_table<Key, Value, HashFunc, ExtractKey, Equal, Alloc>::find(const Key_type &val){
	hash_node_type *pCur = buckets[hash_func(val) % getBucketSize()];
	while(pCur){
		if(equal(get_key(pCur->val), val )){
			return iterator(pCur, this);
		}
		pCur = pCur->next;
	}
	return finish;
}

template<typename Key, typename Value, typename HashFunc, typename ExtractKey, typename Equal, typename Alloc>
void hash_table<Key, Value, HashFunc, ExtractKey, Equal, Alloc>::clear(){
	for(size_type i = 0; i < buckets.size(); i++){
		hash_node_type *pHead = buckets[i], *pNext = NULL;
		while(pHead){
			pNext = pHead->next;
			_destroy(pHead);
			pHead = pNext;
			--num_element;
		}
	}
}

template<typename Key, typename Value, typename HashFunc, typename ExtractKey, typename Equal, typename Alloc>
void hash_table<Key, Value, HashFunc, ExtractKey, Equal, Alloc>::resize(const size_type sz){
	vector<hash_node_type*, Alloc> old_buckets(sz, NULL);
	buckets.swap(old_buckets);
	num_element = 0;
	prime_pos = 0;
	while(prime_table[prime_pos+1] < sz)	prime_pos++;
	for(size_type i = 0; i < old_buckets.size(); i++){
		hash_node_type *pHead = old_buckets[i], *pNext = NULL;
		while(pHead){
			pNext = pHead->next;
			insert_equal(pHead->val);
			_destroy(pHead);
			pHead = pNext;
		}
	}
}

template<typename Key, typename Value, typename HashFunc, typename ExtractKey, typename Equal, typename Alloc>
void hash_table<Key, Value, HashFunc, ExtractKey, Equal, Alloc>::insert_unique(const Value_type value){
	if(count(get_key(value))){
		//remove the older value
		*find(get_key(value)) = value;
		return;
	}
	insert_equal(value);
}

template<typename Key, typename Value, typename HashFunc, typename ExtractKey, typename Equal, typename Alloc>
void hash_table<Key, Value, HashFunc, ExtractKey, Equal, Alloc>::insert_equal(const Value_type value){
	if(num_element >= load_factor_gate * getBucketSize()){
		prime_pos++;
		resize(prime_table[prime_pos]);
	}
	hash_node_type *pCur = buckets[hash_func(get_key(value)) % getBucketSize()];
	hash_node_type *pNode = new hash_node_type(NULL, value);
	if(!pCur)	buckets[hash_func(get_key(value)) % getBucketSize()] = pNode;
	else{
		while(pCur->next)	pCur = pCur->next;
		pCur->next = pNode;
	}
	num_element++;
}

// erase all the same Value
template<typename Key, typename Value, typename HashFunc, typename ExtractKey, typename Equal, typename Alloc>
void hash_table<Key, Value, HashFunc, ExtractKey, Equal, Alloc>::erase(const Key_type key){
	hash_node_type *pHead = buckets[hash_func(key) % getBucketSize()];
	hash_node_type *pCur = pHead, *pNext = NULL;
	if(!pCur)	return;
	while(pCur->next){
		if(equal(get_key(pCur->next->val), key)){
			pNext = pCur->next;
			pCur->next = pNext->next;
			_destroy(pNext);
			num_element--;
		}
	}
	if(equal(get_key(pHead->val), key)){
		buckets[hash_func(key) % getBucketSize()] = pHead->next;
		_destroy(pHead);
		num_element--;
	}
}

// only erase the one that iterator point
template<typename Key, typename Value, typename HashFunc, typename ExtractKey, typename Equal, typename Alloc>
void hash_table<Key, Value, HashFunc, ExtractKey, Equal, Alloc>::erase(iterator iter){
	hash_node<Value_type> *pCur = buckets[hash_func(get_key(iter->cur->val)) % getBucketSize()];
	if(pCur != iter)
		while(pCur->next != iter->cur)	pCur = pCur->next;
	pCur->next = iter->cur->next;
	_destroy(iter->cur);
	num_element--;
}

template<typename Key, typename Value, typename HashFunc, typename ExtractKey, typename Equal, typename Alloc>
void hash_table<Key, Value, HashFunc, ExtractKey, Equal, Alloc>::swap(hash_table_type &rhs){
	buckets.swap(rhs.buckets);
	std::swap(num_element, rhs.num_element);
	std::swap(prime_pos, rhs.prime_pos);
	std::swap(load_factor_gate, rhs.load_factor_gate);
}
{% endhighlight %}

##根据Hash Table实现unordered_set和unordered_map
有了上面的Hash Table底层类再来实现unordered_set和unordered_map简直不要太轻松.我们只需要准备好Hash Table所需要的Key, Value, ExtractKey等类参数然后包装一层API即可.

###unordered_set
其中由于unordered_set的值就是Key,所以Value的类型就是Key,ExtractKey也很就很简单(extract_key_single),代码如下:
{% highlight C++ %}
template<typename Key>
struct extract_key_single{
	Key operator()(const Key key)	const{
		return key;
	}
};

template<typename Key, typename HashFunc = hash_function<Key>, typename EqualKey = equal_key<Key>, typename Alloc = alloc>
class unordered_set{
public:
	typedef Key Key_type;
	typedef Key Value_type;
	typedef Key* pointer;
	typedef Key& reference;
	typedef unordered_set<Key, HashFunc, EqualKey, Alloc> unordered_set_type;
	typedef size_t size_type;
	//注意准备的Key, Value，ExtractKey，这和unordered_map略微不同
	typedef hash_table<Key, Key, HashFunc, extract_key_single<Key>, EqualKey, Alloc> hash_table_type;
	typedef typename hash_table_type::iterator iterator;

	unordered_set() = default;
	template<typename InputIterator>
	unordered_set(InputIterator first, InputIterator last){
		while(first != last){
			insert(*first++);
		}
	}
	unordered_set(const unordered_set_type &rhs){
		ht.swap(rhs.ht);
	}
	~unordered_set()	= default;

	bool empty()	{	return ht.empty();	}
	size_type size()	{	return ht.size();	}
	size_type max_size()	{	return ht.max_size();	}

	iterator begin()	{	return ht.begin();	}
	iterator end()	{	return ht.end();	}

	size_type count(const Key_type key)	{	return ht.count(key);	}
	iterator find(const Key_type key){	return ht.find(key);	}

	void insert(const Value_type key)	{	ht.insert_unique(key); }
	void erase(const Key_type key)	{	ht.erase(key);	}
	void erase(iterator iter)	{	ht.erase(iter);	}
	void clear()	{	ht.clear();	}
	void swap(unordered_set_type &rhs)	{	ht.swap(rhs.ht);	}

	size_type bucket_count()	{	return ht.getBucketSize();	}
	size_type max_bucket_count()	{	return ht.maxBucketSize();	}
	size_type bucket_size(size_type n)	{	return ht.getBucket(n);	}
	size_type bucket(const Key_type key)	{	return ht.getBucketId(key);	}

	double load_factor()	{	return ht.load_factor();	}
	double max_load_factor()	{	return ht.max_load_factor();	}
	void max_load_factor(double factor)	{	ht.max_load_factor(factor);	}
	 // n is the new buckets num
	void rehash(size_type n){
		ht.resize(n);
	}
	// n is the num of element
	void reserve(size_type n){
		if(n >	bucket_count() * load_factor()){
			rehash(n / max_load_factor());
		}
	}

private:
	//封装Hash Table
	hash_table_type ht;
};
{% endhighlight %}
###unordered_map
我们在使用unordered_map时候一般会确定两个类型，比如unordered_map<int,double>,这里int是Key类型而double是Mapped类型，所以传递给Hash Table的Value类型就是pair<Key, Mapped>即pair<int, double>，同样对于ExtractKey(使用extract_key_pair)也需要注意一下.

{% highlight C++ %}
template<typename Key, typename Value>
struct extract_key_pair{
	Key operator()(const Value value)	const{
		return value.first;
	}
};

template<typename Key, typename Mapped, typename HashFunc = hash_function<Key>,
		typename EqualKey = equal_key<Key>, typename Alloc = alloc>
class unordered_map{
public:
	typedef size_t size_type;
	typedef Key key_type;
	typedef Mapped mapped_type;
	typedef std::pair<Key, Mapped>	value_type;
	typedef unordered_map<Key, Mapped, HashFunc, EqualKey, Alloc>	unordered_map_type;
	//准备参数
	typedef hash_table<Key, value_type, HashFunc, extract_key_pair<Key, value_type>, EqualKey, Alloc>	hash_table_type;
	typedef	typename hash_table_type::iterator	iterator;

	unordered_map() = default;
	template<typename InputIterator>
	unordered_map(InputIterator first, InputIterator last){
		while(first != last){
			insert(*first++);
		}
	}
	unordered_map(const unordered_map_type &rhs){
		ht.swap(rhs.ht);
	}
	~unordered_map() = default;

	bool empty()	{	return ht.empty();	}
	size_type size()	{	return ht.size();	}
	size_type max_size()	{	return ht.max_size();	}

	iterator begin()	{	return ht.begin();	}
	iterator end()	{	return ht.end();	}

	mapped_type& operator[](const key_type key){
		if(ht.find(key) == end()){
			insert(std::make_pair(key, mapped_type()));
		}
		return (ht.find(key))->second;
	}

	mapped_type at(const key_type key){
		if(ht.find(key) == end())	THROW_OUT_OF_RANGE;
		return (ht.find(key))->second;
	}

	size_type count(const key_type key)	{	return ht.count(key);	}
	iterator find(const key_type key)	{	return ht.find(key);	}

	/*
	 * attention: insert the value_type and erase the key_type
	 */
	void insert(const value_type key)	{	ht.insert_unique(key); }
	void erase(const key_type key)	{	ht.erase(key);	}
	void erase(iterator iter)	{	ht.erase(iter);	}
	void clear()	{	ht.clear();	}
	void swap(unordered_map_type &rhs)	{	ht.swap(rhs.ht);	}

	size_type bucket_count()	{	return ht.getBucketSize();	}
	size_type max_bucket_count()	{	return ht.maxBucketSize();	}
	size_type bucket_size(size_type n)	{	return ht.getBucket(n);	}
	size_type bucket(const key_type key)	{	return ht.getBucketId(key);	}

	double load_factor()	{	return ht.load_factor();	}
	double max_load_factor()	{	return ht.max_load_factor();	}
	void max_load_factor(double factor)	{	ht.max_load_factor(factor);	}
	 // n is the new buckets num
	void rehash(size_type n){
		ht.resize(n);
	}
	// n is the num of element
	void reserve(size_type n){
		if(n >	bucket_count() * load_factor()){
			rehash(n / max_load_factor());
		}
	}

private:
	hash_table_type ht;
};
{% endhighlight %}

##总结
只要你把Key, Value各自的作用搞得很清楚，然后参照上面提到的数据存储方法就可以很清楚的写出相关代码，另外从这个实例也可以看出迭代器的设计与容器是相互依存的，迭代器需要知道容器的各种信息(通常包含一个容器对象指针)，容器也需要迭代器作为其许多成员函数的接口.
