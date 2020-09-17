---
title: "Golang基础库sort包-API及简单原理"
date: 2020-09-15T10:15:48+08:00
draft: false
---

> 最近的项目中用到了数组排序以及查找功能，Golang基础库sort包提供了这些能力，无需自己来实现。这一篇先介绍一下sort包提供的API以及基本的实现原理，详细的排序算法实现后续文章再介绍。Go版本go1.14

## 0x0 总结先行
- sort包提供的排序算法分为非稳定版本Sort和稳定版本Stable，Sort用到了快速排序、堆排序以及插入排序，Stable则是用到的分段插入排序和merge排序。大多数情况下，使用Sort即可
- 排序可以直接调用Sort，自行封装Interface接口，也可以用闭包的形式调用Slice函数。大部分情况下两者看自己喜好，底层实现是一致的
- sort包对int，float64，string基本类型的切片进行了封装，其他类型可以参考其封装方式自行封装
- sort包的查找(Search方法)要避免踩坑，升序数组中查找值为num的索引，应该写成data[i]>=num，而不是data[i]==num

## 0x1 从sort.Sort开始

```Go
// Sort 排序方法，根据data.Less(i,j) 提供的比较方法排序
func Sort(data Interface) {
	n := data.Len()
	quickSort(data, 0, n, maxDepth(n))
}

// Interface 实现者通常是某个结构的切片
type Interface interface {
    // 返回元素的数量
	Len() int
	// 自定义大小返回规则，判定索引为i的元素是否小于索引为j的元素
	Less(i, j int) bool
	// 自定义元素交换规则，交换索引为i的元素于索引为j的原色
	Swap(i, j int)
}
```
个人理解由于Go目前不支持泛型，因此想要一套代码支持所有基本类型数据或者自定义结构数据的排序，需要借助接口来实现。纵观所有排序算法，对于被排序的数组，可以抽象出三个操作：
- 需要有方法获取数组的长度
- 需要有方法来比较元素间的大小
- 需要有方法来在元素顺序不满足需求时，交换元素位置

这也就是官方要定义Interface这个接口的原因了（吐槽一下Interface这个名字，生怕别人不知道这是个接口）

另外sort包提供了IsSorted方法，快速判断切片是否是排过序的：
```Go
func IsSorted(data Interface) bool {
	n := data.Len()
	for i := n - 1; i > 0; i-- {
		if data.Less(i, i-1) {
			return false
		}
	}
	return true
}
```

Sort和IsSorted的基本用法为：
```Go
type Person struct {
    Age  int
    Name string
}
type SortPerson []Person

func (s SortPerson) Len() int {
    return len(s)
}
func (s SortPerson) Less(i, j int) bool {
    return s[i].Age < s[j].Age
}
func (s SortPerson) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}
func TestSort(t *testing.T) {
    persons := []Person{
            Person{20, "tom"},
            Person{10, "bob"},
            Person{30, "jack"},
            Person{15, "mary"},
    }
    t.Log("before Sort:",persons)
    t.Log("IsSorted", sort.IsSorted(SortPerson(persons)))
    sort.Sort(SortPerson(persons))
    t.Log("after Sort:"persons)
    t.Log("IsSorted", sort.IsSorted(SortPerson(persons)))
}

/*
output:
before Sort: [{20 tom} {10 bob} {30 jack} {15 mary}]
IsSorted false
after Sort: [{10 bob} {15 mary} {20 tom} {30 jack}]
IsSorted true
*/
```

## 0x2 降序排序

Interface接口中的Less方法，默认规则是需要索引为i的值小于索引为j的值时，返回true，Sort在底层调用的时候也是保证i<j,这就保证了按照约定俗成的规则实现接口，得到的是升序的结果。

如果想要降序排列，一个投机的做法是在实现Interface的Less方法时，索引为i的值小于索引为j的值时，返回false，以0x01的示例为例，改为`return s[i].Age > s[j].Age`即可，但是这样比较low，不太符合接口的约定。

sort包提供了Reverse包，可以提供降序排列的能力。
```Go
type reverse struct {
    // 利用了匿名字段，复用了Interface的Len和Swap方法
	Interface
}

// 重载了Interface的Less接口，本质上与上述的方法类似
func (r reverse) Less(i, j int) bool {
	return r.Interface.Less(j, i)
}

// Reverse 包装了Interface，重载了Less方法，得到新的Interface，使得可以逆序排序
func Reverse(data Interface) Interface {
	return &reverse{data}
}
```

Reverse的使用可以参考：
```Go
func TestReverse(t *testing.T) {
    persons := []Person{
            Person{20, "tom"},
            Person{10, "bob"},
            Person{30, "jack"},
            Person{15, "mary"},
    }
    t.Log("before Sort:", persons)
    t.Log("IsSorted", sort.IsSorted(sort.Reverse(SortPerson(persons))))
    sort.Sort(sort.Reverse(SortPerson(persons)))
    t.Log("after Sort:", persons)
    t.Log("IsSorted", sort.IsSorted(sort.Reverse(SortPerson(persons))))
}

/*
output:
before Sort: [{20 tom} {10 bob} {30 jack} {15 mary}]
IsSorted false
after Sort: [{30 jack} {20 tom} {15 mary} {10 bob}]
IsSorted true
*/
```
**需要注意的是：** 如果用了Reverse进行逆序排序，传递到Sort中的Interface已经不是SortPerson，而是reverse包装之后的Interface了，需要用Reverse之后的Interface进行判定。

## 0x3 常用数据结构的封装
sort包对常用的记得数据结构进行了封装，可以直接用来进行排序。
```Go
type IntSlice []int
type Float64Slice []float64
type StringSlice []string
```
IntSlice,Float64Slice,StringSlice分别对[]int,[]float64,[]string进行了封装，实现了Interface接口，并且增加了Sort方法和Search方法(见Search族方法的介绍)，以IntSlice为例:
```Go
type IntSlice []int

func (p IntSlice) Len() int           { return len(p) }
func (p IntSlice) Less(i, j int) bool { return p[i] < p[j] }
func (p IntSlice) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }

// Sort is a convenience method.
func (p IntSlice) Sort() { Sort(p) }
```
在以上封装的基础上，sort提供了基本的排序函数：
```Go
func Ints(a []int) { Sort(IntSlice(a)) }
func Float64s(a []float64) { Sort(Float64Slice(a)) }
func Strings(a []string) { Sort(StringSlice(a)) }
func IntsAreSorted(a []int) bool { return IsSorted(IntSlice(a)) }
func Float64sAreSorted(a []float64) bool { return IsSorted(Float64Slice(a)) }
func StringsAreSorted(a []string) bool { return IsSorted(StringSlice(a)) }
```
简单用法为：
```Go
func TestCommonWrapper(t *testing.T) {
    //Ints按照整数值的大小顺序升序排列 
    ints := []int{3, 2, 4, 5, 1}
    sort.Ints(ints)
    t.Log("Ints:", ints)
    t.Log(sort.IntsAreSorted(ints))

    //Floats按照浮点数的大小顺序升序排列，NaN小于任何数
    floats := []float64{2.1, 0.1, 0.11, math.NaN(), 2.0, math.NaN()}
    sort.Float64s(floats)
    t.Log("Float64s:", floats)
    t.Log(sort.Float64sAreSorted(floats))

    // Strings按照字符串的字典序排序
    strings := []string{"foo", "bar", "hello", "world"}
    sort.Strings(strings)
    t.Log("Strings:", strings)
    t.Log(sort.StringsAreSorted(strings))
}

/*
output:
sort_test.go:55: Ints: [1 2 3 4 5]
sort_test.go:56: true
sort_test.go:60: Float64s: [NaN NaN 0.1 0.11 2 2.1]
sort_test.go:61: true
sort_test.go:65: Strings: [bar foo hello world]
sort_test.go:66: true
*/
```

## 0x4 稳定排序
Sort方法用到的是不稳定的快速排序和堆排序算法，如果用户想要稳定的排序算法，可以使用稳定的Stable方法进行排序。（具体两个排序算法的比较后续文章分析）
```Go
func Stable(data Interface) {
	stable(data, data.Len())
}
```
Stable方法入参同样是Interface，使用方法同Sort一样，大多数情况下使用Sort即可。
```Go
func TestStable(t *testing.T) {
    persons := []Person{
            Person{20, "tom"},
            Person{10, "bob"},
            Person{30, "jack"},
            Person{15, "mary"},
    }
    t.Log("before Stable:", persons)
    sort.Stable(SortPerson(persons))
    t.Log("after Stable:", persons)
}
/*
output:
before Stable: [{20 tom} {10 bob} {30 jack} {15 mary}]
after Stable: [{10 bob} {15 mary} {20 tom} {30 jack}]
*/
``` 

## 0x5 Slice族方法
Sort方法理论上可以对任意实现了Interface的结构体进行排序，但是缺点是基本结构体需要包装成Interface，对于slice类型调用起来有点麻烦，因此sort包提供了Slice族的方法，专门来对slice类型进行排序。调用需要提供Less方法指定排序规则即可，Len和Swap由系统默认提供。
```Go
var reflectValueOf = reflectlite.ValueOf  提供了Len()功能
var reflectSwapper = reflectlite.Swapper  //提供了Swap功能

func Slice(slice interface{}, less func(i, j int) bool) {
	rv := reflectValueOf(slice)
	swap := reflectSwapper(slice)
	length := rv.Len()
	quickSort_func(lessSwap{less, swap}, 0, length, maxDepth(length))
}
```

另外SLice族方法中还包括SliceStable提供稳定排序，SliceIsSort判断slice是否是排序状态的。
```Go
func SliceStable(slice interface{}, less func(i, j int) bool) {
	rv := reflectValueOf(slice)
	swap := reflectSwapper(slice)
	stable_func(lessSwap{less, swap}, rv.Len())
}


func SliceIsSorted(slice interface{}, less func(i, j int) bool) bool {
	rv := reflectValueOf(slice)
	n := rv.Len()
	for i := n - 1; i > 0; i-- {
		if less(i, i-1) {
			return false
		}
	}
	return true
}
```

使用方法为：
```Go
func TestSlice(t *testing.T) {
    persons := []Person{
            Person{20, "tom"},
            Person{10, "bob"},
            Person{30, "jack"},
            Person{15, "mary"},
    }
    t.Log("before Slice:", persons)
    sort.Slice(persons, func(i, j int) bool {
            return persons[i].Age < persons[j].Age
    })
    t.Log("after Slice:", persons)
}
/*
output:
before Slice: [{20 tom} {10 bob} {30 jack} {15 mary}]
after Slice: [{10 bob} {15 mary} {20 tom} {30 jack}]
*/
```
另外，sort.Slice第二个参数是一个闭包，可以做一些骚操作，就不展开说了。（其实是没想到什么比较骚的操作,但确实有骚的空间）

## 0x6 Search族方法

```Go
func Search(n int, f func(int) bool) int {
	// Define f(-1) == false and f(n) == true.
	// Invariant: f(i-1) == false, f(j) == true.
	i, j := 0, n
	for i < j {
		h := int(uint(i+j) >> 1) // avoid overflow when computing h
		// i ≤ h < j
		if !f(h) {
			i = h + 1 // preserves f(i-1) == false
		} else {
			j = h // preserves f(j) == true
		}
	}
	// i == j, f(i-1) == false, and f(j) (= f(i)) == true  =>  answer is i.
	return i
}
```
Search方法利用二分法返回[0,n)范围内，使f返回true的最小值，需要保证的是如果f(i)为true，那么f(i+1)及以后的值都要为true，所以不是简单的查找值。以一个例子说明一下：
```Go
func TestSearch(t *testing.T) {
    datas := []int{1, 2, 3, 4, 5, 6, 7, 9, 10}
    i1 := sort.Search(len(datas), func(i int) bool {
            return datas[i] == 6
    })
    t.Log("Index:", i1)
    for _, tobefind := range []int{6, 8, 11} {
            i2 := sort.Search(len(datas), func(i int) bool {
                    return datas[i] >= tobefind
            })
            t.Log("Index:", i2)
    }
}
/*
output:
Index:9
Index:5
Index:7
Index:9
*/
```
所以说如果要找出一个升序排列数组中等于6的值的索引，不能写`data[i]==6`，应该是写成`data[i]>=6`，保证索引右侧的值都能使得f返回true.
如果没有任何最小值满足条件，那么返回len(datas)。

如果数组是降序排列的，查找某个值时，需要写成`data[i]<=${num}`的形式。

另外，sort包提供了对int,float64,string类型的封装，用于快速在升序数组中查找:
```Go
func SearchInts(a []int, x int) int {
	return Search(len(a), func(i int) bool { return a[i] >= x })
}

func SearchFloat64s(a []float64, x float64) int {
	return Search(len(a), func(i int) bool { return a[i] >= x })
}

func SearchStrings(a []string, x string) int {
	return Search(len(a), func(i int) bool { return a[i] >= x })
}
```

`0x3`节中介绍了IntSlice、Float64Slice，StringSlice结构也实现了Search方法，也是比较简单，是对Search方法的封装
```Go
func (p IntSlice) Search(x int) int { return SearchInts(p, x) }

func (p Float64Slice) Search(x float64) int { return SearchFloat64s(p, x) }

func (p StringSlice) Search(x string) int { return SearchStrings(p, x) }
```
