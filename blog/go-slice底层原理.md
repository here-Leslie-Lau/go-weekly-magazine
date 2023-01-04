# 切片特性

> 在`golang`中，存储数据的集合有两种类型，分别是数组(array)和切片(slice)。
>
> 其中切片是可变的长度，当切片长度不够时，`golang`会对其进行扩容
> 切片遍历方式和数组一样，可以用len()求长度。表示可用元素数量，读写操作不能超过该限制。
> 切片是数组的一个引用，因此切片是引用类型。但自身是结构体，值拷贝传递。

---

# 切片使用

仅记录一些常用操作

- **创建切片:**

```go
	// 创建切片
	slice := make([]int, 1, 3) // 使用make可以创建切片的引用，第二个参数代表切片长度，第三个代表切片的容量，使用该种方法生成具有空值
	// 创建切片
	slice := make([]int, 1, 3) // 直接对切片进行初始化操作
```

- **切片遍历:**

```go
	// 使用foreach遍历
	slice := []int{1, 2, 3}
	for _, val := range slice {
		fmt.Println(val)
	}
	// 直接使用索引进行遍历
	slice := []int{1, 2, 3}
	for i := 0; i < len(slice); i++ {
		fmt.Println(slice[i])
	}
```

- **使用append追加切片元素:**

```go
	slice := make([]int, 3)
	for i := 0; i < 3; i++ {
		slice = append(slice, i)	// 该方式会在切片的末尾追加元素
	}
```

- **切片拷贝:**

```go
	slice1 := []int{1, 2, 3}
	slice2 := make([]int, 3)
	copy(slice2, slice1)
```

- **切片索引取值:**

```go
	slice := []int{1, 2, 3}
	fmt.Println(slice[0:2])	// 输出内容:[1 2], slice[0:2]代表去slice第[0]个元素，一直取到第[2]元素(不包括[2])
	fmt.Println(slice[0:1:2]) // 输出内容:[1], slice[0:1:2]第三个数字表示新切片容量为(2-0=2)
```

---

# 切片的数据结构


刚刚在**切片特性**内介绍到了golang中切片其实是值拷贝。
这里其实有一个疑问，如果切片元素内过多，那么每次传值不会特别特别慢吗？毕竟每次都要拷贝一个数组。别急，接下来介绍一下切片的数据结构。


## 数据结构

切片本身并不是动态数组或者数组指针。它内部实现的数据结构通过指针引用底层数组，设定相关属性将数据读写操作限定在指定的区域内。切片本身是一个只读对象，其工作机制类似数组指针的一种封装。

Slice 的数据结构定义如下:

```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

切片的结构体由3部分构成，Pointer 是指向一个数组的指针，len 代表当前切片的长度，cap 是当前切片的容量。cap 总是大于等于 len 的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/a4d6a8dfa563412cb1aef33836c92d73.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6L-ZTGVzbGllX0xhdQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
*图片来自互联网*

---

## 切片操作底层原理

- **创建切片**

  创建切片有两种形式，make 创建切片，空切片。

  ```go
  func makeslice(et *_type, len, cap int) slice {
    	// 根据切片的数据类型，获取切片的最大容量
    	maxElements := maxSliceCap(et.size)
    	// 比较切片的长度，长度值域应该在[0,maxElements]之间
    	if len < 0 || uintptr(len) > maxElements {
      		panic(errorString("makeslice: len out of range"))
    	}
    	// 比较切片的容量，容量值域应该在[len,maxElements]之间
    	if cap < len || uintptr(cap) > maxElements {
        	panic(errorString("makeslice: cap out of range"))
    	}
    	// 根据切片的容量申请内存
    	p := mallocgc(et.size*uintptr(cap), et, true)
    	// 返回申请好内存的切片的首地址
    	return slice{p, len, cap}
  }
  ```

- **nil与空切片**

  nil与空切片在项目中也是比较常用的，比如一些接口需要返回一个切片，但是程序出错时，就需要返回一个nil的切片或者带有空值的切片。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/1909ffe5d4d144f9b3a5015546456ddd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6L-ZTGVzbGllX0xhdQ==,size_20,color_FFFFFF,t_70,g_se,x_16)

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/1122d82ffc734a38bdcaff16a772d400.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6L-ZTGVzbGllX0xhdQ==,size_20,color_FFFFFF,t_70,g_se,x_16)

  空切片和 nil 切片的区别在于，空切片指向的地址不是nil，指向的是一个内存地址，但是它没有分配任何内存空间，即底层元素包含0个元素。

- **拷贝切片**

  这里直接上源码:

  ```go
  func slicecopy(to, fm slice, width uintptr) int {
      // 如果源切片或者目标切片有一个长度为0，那么就不需要拷贝，直接 return 
      if fm.len == 0 || to.len == 0 {
          return 0
      }
      // n 记录下源切片或者目标切片较短的那一个的长度
      n := fm.len
      if to.len < n {
          n = to.len
      }
      // 如果入参 width = 0，也不需要拷贝了，返回较短的切片的长度
      if width == 0 {
          return n
      }
      // 如果开启了竞争检测
      if raceenabled {
          callerpc := getcallerpc(unsafe.Pointer(&to))
          pc := funcPC(slicecopy)
          racewriterangepc(to.array, uintptr(n*int(width)), callerpc, pc)
          racereadrangepc(fm.array, uintptr(n*int(width)), callerpc, pc)
      }
      // 如果开启了 The memory sanitizer (msan)
      if msanenabled {
          msanwrite(to.array, uintptr(n*int(width)))
          msanread(fm.array, uintptr(n*int(width)))
      }
  
      size := uintptr(n) * width
      if size == 1 { 
          // TODO: is this still worth it with new memmove impl?
          // 如果只有一个元素，那么指针直接转换即可
          *(*byte)(to.array) = *(*byte)(fm.array) // known to be a byte pointer
      } else {
          // 如果不止一个元素，那么就把 size 个 bytes 从 fm.array 地址开始，拷贝到 to.array 地址之后
          memmove(to.array, fm.array, size)
      }
      return n
  }
  ```

  在这个方法中，slicecopy 方法会把源切片值(即 fm Slice )中的元素复制到目标切片(即 to Slice )中，并返回被复制的元素个数，copy 的两个类型必须一致。slicecopy 方法最终的复制结果取决于较短的那个切片，当较短的切片复制完成，整个复制过程就全部完成了。
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/9fa2c51d5ab9432997c1666832497704.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6L-ZTGVzbGllX0xhdQ==,size_20,color_FFFFFF,t_70,g_se,x_16)


- **切片扩容**

  ```go
  func growslice(et *_type, old slice, cap int) slice {
      if raceenabled {
          callerpc := getcallerpc(unsafe.Pointer(&et))
          racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
      }
      if msanenabled {
          msanread(old.array, uintptr(old.len*int(et.size)))
      }
  
      if et.size == 0 {
          // 如果新要扩容的容量比原来的容量还要小，这代表要缩容了，那么可以直接报panic了。
          if cap < old.cap {
              panic(errorString("growslice: cap out of range"))
          }
  
          // 如果当前切片的大小为0，还调用了扩容方法，那么就新生成一个新的容量的切片返回。
          return slice{unsafe.Pointer(&zerobase), old.len, cap}
      }
  
    // 这里就是扩容的策略
      newcap := old.cap
      doublecap := newcap + newcap
      if cap > doublecap {
          newcap = cap
      } else {
          if old.len < 1024 {
              newcap = doublecap
          } else {
              for newcap < cap {
                  newcap += newcap / 4
              }
          }
      }
  
      // 计算新的切片的容量，长度。
      var lenmem, newlenmem, capmem uintptr
      const ptrSize = unsafe.Sizeof((*byte)(nil))
      switch et.size {
      case 1:
          lenmem = uintptr(old.len)
          newlenmem = uintptr(cap)
          capmem = roundupsize(uintptr(newcap))
          newcap = int(capmem)
      case ptrSize:
          lenmem = uintptr(old.len) * ptrSize
          newlenmem = uintptr(cap) * ptrSize
          capmem = roundupsize(uintptr(newcap) * ptrSize)
          newcap = int(capmem / ptrSize)
      default:
          lenmem = uintptr(old.len) * et.size
          newlenmem = uintptr(cap) * et.size
          capmem = roundupsize(uintptr(newcap) * et.size)
          newcap = int(capmem / et.size)
      }
  
      // 判断非法的值，保证容量是在增加，并且容量不超过最大容量
      if cap < old.cap || uintptr(newcap) > maxSliceCap(et.size) {
          panic(errorString("growslice: cap out of range"))
      }
  
      var p unsafe.Pointer
      if et.kind&kindNoPointers != 0 {
          // 在老的切片后面继续扩充容量
          p = mallocgc(capmem, nil, false)
          // 将 lenmem 这个多个 bytes 从 old.array地址 拷贝到 p 的地址处
          memmove(p, old.array, lenmem)
          // 先将 P 地址加上新的容量得到新切片容量的地址，然后将新切片容量地址后面的 capmem-newlenmem 个 bytes 这块内存初始化。为之后继续 append() 操作腾出空间。
          memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
      } else {
          // 重新申请新的数组给新切片
          // 重新申请 capmen 这个大的内存地址，并且初始化为0值
          p = mallocgc(capmem, et, true)
          if !writeBarrier.enabled {
              // 如果还不能打开写锁，那么只能把 lenmem 大小的 bytes 字节从 old.array 拷贝到 p 的地址处
              memmove(p, old.array, lenmem)
          } else {
              // 循环拷贝老的切片的值
              for i := uintptr(0); i < lenmem; i += et.size {
                  typedmemmove(et, add(p, i), add(old.array, i))
              }
          }
      }
      // 返回最终新切片，容量更新为最新扩容之后的容量
      return slice{p, old.len, newcap}
  }
  ```

  Go 中切片扩容的策略是这样的：

  如果切片的容量小于 1024 个元素，于是扩容的时候就翻倍增加容量。上面那个例子也验证了这一情况，总容量从原来的4个翻倍到现在的8个。

  一旦元素个数超过 1024 个元素，那么增长因子就变成 1.25 ，即每次增加原来容量的四分之一。

  注意：扩容扩大的容量都是针对原来的容量而言的，而不是针对原来数组的长度而言的。

---