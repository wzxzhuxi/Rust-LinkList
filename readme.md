

# 手写rust链表，使用unsafe获得和C相同的性能






### 参考书籍：rust圣经，https://github.com/sunface/rust-course



**UP只是学生，非专业教学，非Rust高手，大佬轻喷，有任何不对的地方也期待您的批评指正**

**不适合纯小白，推荐观看杨旭老师的基础教学视频之后观看，【Rust编程语言入门教程】BV1hp4y1k7SV**




### 引入库函数

```rust
// 引用比较和排序功能
use std::cmp::Ordering;
// 引用格式化功能，用于调试和打印
use std::fmt::{self, Debug};
// 引用哈希和散列功能
use std::hash::{Hash, Hasher};
// 引用从迭代器创建集合的功能
use std::iter::FromIterator;
// 引用标记功能，用于类型系统的高级特性
use std::marker::PhantomData;
// 引用指针功能，用于创建非空指针
use std::ptr::NonNull;
```



### 结构体定义代码

```rust
// 定义一个泛型链表结构体
pub struct LinkedList<T> {
    // 链表头指针
    front: Link<T>,
    // 链表尾指针
    back: Link<T>,
    // 链表长度
    len: usize,
    // 用于避免未使用类型参数T的警告
    _boo: PhantomData<T>,
}
```
```rust
// 定义一个链接节点类型为Option<NonNull<Node<T>>>，即节点的指针
type Link<T> = Option<NonNull<Node<T>>>;
```
### 对数据结构的说明：

使用NonNull创建非空的指针，使用PhantomData标记存在一个数据，达到指针非空的效果

```rust
// 定义链表节点结构体
struct Node<T> {
    // 前一个节点的指针
    front: Link<T>,
    // 后一个节点的指针
    back: Link<T>,
    // 节点元素
    elem: T,
}
```

```rust
// 定义不可变迭代器结构体
pub struct Iter<'a, T> {
    // 当前前节点的指针
    front: Link<T>,
    // 当前后节点的指针
    back: Link<T>,
    // 剩余的元素个数
    len: usize,
    // 用于避免未使用生命周期和类型参数的警告
    _boo: PhantomData<&'a T>,
}

// 定义可变迭代器结构体
pub struct IterMut<'a, T> {
    // 当前前节点的指针
    front: Link<T>,
    // 当前后节点的指针
    back: Link<T>,
    // 剩余的元素个数
    len: usize,
    // 用于避免未使用生命周期和类型参数的警告
    _boo: PhantomData<&'a mut T>,
}

// 定义所有权迭代器结构体
pub struct IntoIter<T> {
    // 链表本身
    list: LinkedList<T>,
}
```


```rust
// 定义可变游标结构体
pub struct CursorMut<'a, T> {
    // 链表引用
    list: &'a mut LinkedList<T>,
    // 当前节点指针
    cur: Link<T>,
    // 当前索引位置
    index: Option<usize>,
}
```



###  实现链表的push与pop方法

```rust
	// 创建一个新的空链表
    pub fn new() -> Self {
        Self {
            front: None, // 初始化头指针为空
            back: None,  // 初始化尾指针为空
            len: 0,      // 初始化长度为0
            _boo: PhantomData, // 避免未使用类型参数的警告
        }
    }

    // 在链表头部插入元素
    pub fn push_front(&mut self, elem: T) {
        // 安全保证：这是一个链表操作
        unsafe {
            // 创建一个新的节点并将其包装成NonNull指针
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None, // 新节点前指针为空
                back: None,  // 新节点后指针为空
                elem,        // 节点元素
            })));
            if let Some(old) = self.front {
                // 如果链表头存在，更新指针
                (*old.as_ptr()).front = Some(new); // 旧头节点前指针指向新节点
                (*new.as_ptr()).back = Some(old);  // 新节点后指针指向旧头节点
            } else {
                // 如果链表为空，更新尾指针
                self.back = Some(new);
            }
            // 更新链表头指针
            self.front = Some(new);
            // 链表长度加1
            self.len += 1;
        }
    }

    // 在链表尾部插入元素
    pub fn push_back(&mut self, elem: T) {
        // 安全保证：这是一个链表操作
        unsafe {
            // 创建一个新的节点并将其包装成NonNull指针
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                back: None, // 新节点后指针为空
                front: None, // 新节点前指针为空
                elem,       // 节点元素
            })));
            if let Some(old) = self.back {
                // 如果链表尾存在，更新指针
                (*old.as_ptr()).back = Some(new); // 旧尾节点后指针指向新节点
                (*new.as_ptr()).front = Some(old); // 新节点前指针指向旧尾节点
            } else {
                // 如果链表为空，更新头指针
                self.front = Some(new);
            }
            // 更新链表尾指针
            self.back = Some(new);
            // 链表长度加1
            self.len += 1;
        }
    }

    // 弹出链表头部的元素
    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // 只有在链表头存在时才进行操作
            self.front.map(|node| {
                // 将节点从Box中取出
                let boxed_node = Box::from_raw(node.as_ptr());
                // 获取节点元素
                let result = boxed_node.elem;

                // 更新链表头指针为下一个节点
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // 清理新头节点的前指针
                    (*new.as_ptr()).front = None;
                } else {
                    // 如果链表头为空，更新尾指针为空
                    self.back = None;
                }

                // 链表长度减1
                self.len -= 1;
                // 返回弹出的元素
                result
                // Box在这里被隐式释放
            })
        }
    }

    // 弹出链表尾部的元素
    pub fn pop_back(&mut self) -> Option<T> {
        unsafe {
            // 只有在链表尾存在时才进行操作
            self.back.map(|node| {
                // 将节点从Box中取出
                let boxed_node = Box::from_raw(node.as_ptr());
                // 获取节点元素
                let result = boxed_node.elem;

                // 更新链表尾指针为上一个节点
                self.back = boxed_node.front;
                if let Some(new) = self.back {
                    // 清理新尾节点的后指针
                    (*new.as_ptr()).back = None;
                } else {
                    // 如果链表尾为空，更新头指针为空
                    self.front = None;
                }

                // 链表长度减1
                self.len -= 1;
                // 返回弹出的元素
                result
                // Box在这里被隐式释放
            })
        }
    }

	// 获取链表的长度
    pub fn len(&self) -> usize {
        self.len
    }

    // 检查链表是否为空
    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    // 清空链表
    pub fn clear(&mut self) {
        // 反复弹出头部节点，直到链表为空
        while self.pop_front().is_some() {}
    }
```



### 实现链表的各种引用

``` rust
	// 获取链表头部元素的引用
    pub fn front(&self) -> Option<&T> {
        unsafe { self.front.map(|node| &(*node.as_ptr()).elem) }
    }

    // 获取链表头部元素的可变引用
    pub fn front_mut(&mut self) -> Option<&mut T> {
        unsafe { self.front.map(|node| &mut (*node.as_ptr()).elem) }
    }

    // 获取链表尾部元素的引用
    pub fn back(&self) -> Option<&T> {
        unsafe { self.back.map(|node| &(*node.as_ptr()).elem) }
    }

    // 获取链表尾部元素的可变引用
    pub fn back_mut(&mut self) -> Option<&mut T> {
        unsafe { self.back.map(|node| &mut (*node.as_ptr()).elem) }
    }
```



### 获取各类迭代器

``` rust
	// 获取链表的不可变迭代器
    pub fn iter(&self) -> Iter<T> {
        Iter {
            front: self.front, // 迭代器从链表头开始
            back: self.back,   // 迭代器到链表尾结束
            len: self.len,     // 迭代器的初始长度
            _boo: PhantomData, // 避免未使用类型参数的警告
        }
    }

    // 获取链表的可变迭代器
    pub fn iter_mut(&mut self) -> IterMut<T> {
        IterMut {
            front: self.front, // 迭代器从链表头开始
            back: self.back,   // 迭代器到链表尾结束
            len: self.len,     // 迭代器的初始长度
            _boo: PhantomData, // 避免未使用类型参数的警告
        }
    }

    // 获取链表的可变游标
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut {
            list: self,    // 游标指向链表
            cur: None,     // 初始游标位置为空
            index: None,   // 初始索引位置为空
        }
    }
```



### 为链表实现各项trait

``` rust
// 为链表实现析构函数，确保内存安全释放
impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        // 反复弹出头部节点，直到链表为空
        while self.pop_front().is_some() {}
    }
}

// 为链表实现默认构造函数
impl<T> Default for LinkedList<T> {
    fn default() -> Self {
        Self::new()
    }
}

// 为链表实现克隆功能
impl<T: Clone> Clone for LinkedList<T> {
    fn clone(&self) -> Self {
        let mut new_list = Self::new(); // 创建一个新的链表
        for item in self {
            new_list.push_back(item.clone()); // 逐个克隆元素并添加到新链表
        }
        new_list // 返回新的链表
    }
}

// 为链表实现扩展功能
impl<T> Extend<T> for LinkedList<T> {
    fn extend<I: IntoIterator<Item = T>>(&mut self, iter: I) {
        for item in iter {
            self.push_back(item); // 逐个将元素添加到链表尾部
        }
    }
}

// 为链表实现从迭代器创建链表的功能
impl<T> FromIterator<T> for LinkedList<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self {
        let mut list = Self::new(); // 创建一个新的链表
        list.extend(iter); // 扩展链表
        list // 返回新的链表
    }
}

// 为链表实现调试格式化功能
impl<T: Debug> Debug for LinkedList<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_list().entries(self).finish() // 以调试格式打印链表
    }
}

// 为链表实现部分相等比较功能
impl<T: PartialEq> PartialEq for LinkedList<T> {
    fn eq(&self, other: &Self) -> bool {
        self.len() == other.len() && self.iter().eq(other.iter()) // 比较链表的长度和元素是否相等
    }
}

// 为链表实现完全相等比较功能
impl<T: Eq> Eq for LinkedList<T> {}

// 为链表实现部分顺序比较功能
impl<T: PartialOrd> PartialOrd for LinkedList<T> {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        self.iter().partial_cmp(other.iter()) // 比较链表元素的顺序
    }
}

// 为链表实现完全顺序比较功能
impl<T: Ord> Ord for LinkedList<T> {
    fn cmp(&self, other: &Self) -> Ordering {
        self.iter().cmp(other.iter()) // 比较链表元素的顺序
    }
}

// 为链表实现哈希功能
impl<T: Hash> Hash for LinkedList<T> {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.len().hash(state); // 先哈希链表的长度
        for item in self {
            item.hash(state); // 逐个哈希链表的元素
        }
    }
}
```



### 为链表实现迭代器相关trait

```rust
// 为链表实现扩展功能
impl<T> Extend<T> for LinkedList<T> {
    fn extend<I: IntoIterator<Item = T>>(&mut self, iter: I) {
        for item in iter {
            self.push_back(item); // 逐个将元素添加到链表尾部
        }
    }
}

// 为链表实现从迭代器创建链表的功能
impl<T> FromIterator<T> for LinkedList<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self {
        let mut list = Self::new(); // 创建一个新的链表
        list.extend(iter); // 扩展链表
        list // 返回新的链表
    }
}
```



### 不可变迭代器的方法实现

**因为返回生命周期超出当前代码段的引用，所以需要显式标注生命周期**

1. into_iter：创建不可变迭代器
2. next：获取迭代器下一元素的数据
3. size_hint：迭代器中剩余元素的下界和上界
4. next_back：获取迭代器上一个元素的数据
5. len：获取迭代器长度

``` rust
// 为链表实现不可变迭代器接口
impl<'a, T> IntoIterator for &'a LinkedList<T> {
    type IntoIter = Iter<'a, T>;
    type Item = &'a T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter() // 返回不可变迭代器
    }
}

// 为不可变迭代器实现迭代器接口
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            // 如果还有元素，返回下一个元素
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back; // 更新前指针
                &(*node.as_ptr()).elem // 返回当前元素
            })
        } else {
            None // 没有元素了
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len)) // 返回迭代器的长度
    }
}

// 为不可变迭代器实现双向迭代器接口
impl<'a, T> DoubleEndedIterator for Iter<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            // 如果还有元素，返回前一个元素
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front; // 更新后指针
                &(*node.as_ptr()).elem // 返回当前元素
            })
        } else {
            None // 没有元素了
        }
    }
}

// 为不可变迭代器实现精确大小迭代器接口
impl<'a, T> ExactSizeIterator for Iter<'a, T> {
    fn len(&self) -> usize {
        self.len // 返回迭代器的长度
    }
}
```



### 可变迭代器的方法实现

**实现方法与不可变类型相同**

```rust
// 为链表实现可变迭代器接口
impl<'a, T> IntoIterator for &'a mut LinkedList<T> {
    type IntoIter = IterMut<'a, T>;
    type Item = &'a mut T;

    fn into_iter(self) -> Self::IntoIter {
        self.iter_mut() // 返回可变迭代器
    }
}

// 为可变迭代器实现迭代器接口
impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            // 如果还有元素，返回下一个元素
            self.front.map(|node| unsafe {
                self.len -= 1;
                self.front = (*node.as_ptr()).back; // 更新前指针
                &mut (*node.as_ptr()).elem // 返回当前元素
            })
        } else {
            None // 没有元素了
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.len, Some(self.len)) // 返回迭代器的长度
    }
}

// 为可变迭代器实现双向迭代器接口
impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.len > 0 {
            // 如果还有元素，返回前一个元素
            self.back.map(|node| unsafe {
                self.len -= 1;
                self.back = (*node.as_ptr()).front; // 更新后指针
                &mut (*node.as_ptr()).elem // 返回当前元素
            })
        } else {
            None // 没有元素了
        }
    }
}

// 为可变迭代器实现精确大小迭代器接口
impl<'a, T> ExactSizeIterator for IterMut<'a, T> {
    fn len(&self) -> usize {
        self.len // 返回迭代器的长度
    }
}
```



### 所有权迭代器的方法实现：

**所有权迭代器会消费链表节点，所以无需标注生命周期**

```rust
// 为链表实现所有权迭代器接口
impl<T> IntoIterator for LinkedList<T> {
    type IntoIter = IntoIter<T>;
    type Item = T;

    fn into_iter(self) -> Self::IntoIter {
        IntoIter { list: self } // 返回所有权迭代器
    }
}

// 为所有权迭代器实现迭代器接口
impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        self.list.pop_front() // 弹出链表头部元素并返回
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len)) // 返回迭代器的长度
    }
}

// 为所有权迭代器实现双向迭代器接口
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        self.list.pop_back() // 弹出链表尾部元素并返回
    }
}

// 为所有权迭代器实现精确大小迭代器接口
impl<T> ExactSizeIterator for IntoIter<T> {
    fn len(&self) -> usize {
        self.list.len // 返回迭代器的长度
    }
}
```



### 游标与相关方法实现：

1. index// 获取游标的当前索引

2. move_next// 移动游标到下一个元素

3. move_prev// 移动游标到前一个元素

4. current// 获取当前游标指向的元素的可变引用

5. peek_next// 查看下一个元素的可变引用

6. peek_prev// 查看前一个元素的可变引用

7. split_before// 分割链表，将当前游标之前的部分拆分出来

8. split_after// 分割链表，将当前游标之后的部分拆分出来

9. splice_before// 将一个链表插入到当前游标之前

10. splice_after// 将一个链表插入到当前游标之后

    

```rust
// 为游标实现相关方法
impl<'a, T> CursorMut<'a, T> {
    // 获取游标的当前索引
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    // 移动游标到下一个元素
    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // 当前游标指向下一个元素
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    // 更新索引
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // 如果没有下一个元素，索引为空
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // 如果链表非空且游标在尾部，将游标移动到头部
            self.cur = self.list.front;
            self.index = Some(0);
        }
    }

    // 移动游标到前一个元素
    pub fn move_prev(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // 当前游标指向前一个元素
                self.cur = (*cur.as_ptr()).front;
                if self.cur.is_some() {
                    // 更新索引
                    *self.index.as_mut().unwrap() -= 1;
                } else {
                    // 如果没有前一个元素，索引为空
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // 如果链表非空且游标在头部，将游标移动到尾部
            self.cur = self.list.back;
            self.index = Some(self.list.len - 1);
        }
    }

    // 获取当前游标指向的元素的可变引用
    pub fn current(&mut self) -> Option<&mut T> {
        unsafe { self.cur.map(|node| &mut (*node.as_ptr()).elem) }
    }

    // 查看下一个元素的可变引用
    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            let next = if let Some(cur) = self.cur {
                // 获取下一个节点的指针
                (*cur.as_ptr()).back
            } else {
                // 如果游标在尾部，获取链表头部的指针
                self.list.front
            };

            // 如果下一个节点存在，返回其元素的可变引用
            next.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    // 查看前一个元素的可变引用
    pub fn peek_prev(&mut self) -> Option<&mut T> {
        unsafe {
            let prev = if let Some(cur) = self.cur {
                // 获取前一个节点的指针
                (*cur.as_ptr()).front
            } else {
                // 如果游标在头部，获取链表尾部的指针
                self.list.back
            };

            // 如果前一个节点存在，返回其元素的可变引用
            prev.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    // 分割链表，将当前游标之前的部分拆分出来
    pub fn split_before(&mut self) -> LinkedList<T> {
        if let Some(cur) = self.cur {
            unsafe {
                // 当前链表的长度
                let old_len = self.list.len;
                // 当前游标的索引
                let old_idx = self.index.unwrap();
                // 当前节点的前一个节点
                let prev = (*cur.as_ptr()).front;

                // 更新当前链表的新长度
                let new_len = old_len - old_idx;
                // 更新当前链表的新头指针
                let new_front = self.cur;
                // 更新当前链表的新尾指针
                let new_back = self.list.back;
                // 更新当前游标的新索引
                let new_idx = Some(0);

                // 更新拆分出来的链表的长度
                let output_len = old_len - new_len;
                // 更新拆分出来的链表的头指针
                let output_front = self.list.front;
                // 更新拆分出来的链表的尾指针
                let output_back = prev;

                // 断开当前节点与前一个节点的链接
                if let Some(prev) = prev {
                    (*cur.as_ptr()).front = None;
                    (*prev.as_ptr()).back = None;
                }

                // 更新当前链表的长度和头尾指针
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // 如果游标在尾部，将链表替换为空链表
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    // 分割链表，将当前游标之后的部分拆分出来
    pub fn split_after(&mut self) -> LinkedList<T> {
        if let Some(cur) = self.cur {
            unsafe {
                // 当前链表的长度
                let old_len = self.list.len;
                // 当前游标的索引
                let old_idx = self.index.unwrap();
                // 当前节点的后一个节点
                let next = (*cur.as_ptr()).back;

                // 更新当前链表的新长度
                let new_len = old_idx + 1;
                // 更新当前链表的新尾指针
                let new_back = self.cur;
                // 更新当前链表的新头指针
                let new_front = self.list.front;
                // 更新当前游标的新索引
                let new_idx = Some(old_idx);

                // 更新拆分出来的链表的长度
                let output_len = old_len - new_len;
                // 更新拆分出来的链表的头指针
                let output_front = next;
                // 更新拆分出来的链表的尾指针
                let output_back = self.list.back;

                // 断开当前节点与后一个节点的链接
                if let Some(next) = next {
                    (*cur.as_ptr()).back = None;
                    (*next.as_ptr()).front = None;
                }

                // 更新当前链表的长度和头尾指针
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            }
        } else {
            // 如果游标在尾部，将链表替换为空链表
            std::mem::replace(self.list, LinkedList::new())
        }
    }

    // 将一个链表插入到当前游标之前
    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        unsafe {
            if input.is_empty() {
                // 如果输入链表为空，不做任何操作
            } else if let Some(cur) = self.cur {
                // 如果当前链表和输入链表都非空
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(prev) = (*cur.as_ptr()).front {
                    // 常规情况，链表内部链接修复
                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                } else {
                    // 如果没有前一个节点，插入到头部
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                    self.list.front = Some(in_front);
                }
                // 更新索引
                *self.index.as_mut().unwrap() += input.len;
            } else if let Some(back) = self.list.back {
                // 如果游标在尾部，插入到尾部
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*back.as_ptr()).back = Some(in_front);
                (*in_front.as_ptr()).front = Some(back);
                self.list.back = Some(in_back);
            } else {
                // 如果链表为空，将其替换为输入链表
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            input.len = 0;
        }
    }

    // 将一个链表插入到当前游标之后
    pub fn splice_after(&mut self, mut input: LinkedList<T>) {
        unsafe {
            if input.is_empty() {
                // 如果输入链表为空，不做任何操作
            } else if let Some(cur) = self.cur {
                // 如果当前链表和输入链表都非空
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(next) = (*cur.as_ptr()).back {
                    // 常规情况，链表内部链接修复
                    (*next.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(next);
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                } else {
                    // 如果没有后一个节点，插入到尾部
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                    self.list.back = Some(in_back);
                }
            } else if let Some(front) = self.list.front {
                // 如果游标在头部，插入到头部
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*front.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(front);
                self.list.front = Some(in_front);
            } else {
                // 如果链表为空，将其替换为输入链表
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            input.len = 0;
        }
    }
}
```



### 多线程环境准备

```rust
// 为链表和迭代器实现Send和Sync特性，以便在多线程环境中使用
unsafe impl<T: Send> Send for LinkedList<T> {}
unsafe impl<T: Sync> Sync for LinkedList<T> {}

unsafe impl<'a, T: Send> Send for Iter<'a, T> {}
unsafe impl<'a, T: Sync> Sync for Iter<'a, T> {}

unsafe impl<'a, T: Send> Send for IterMut<'a, T> {}
unsafe impl<'a, T: Sync> Sync for IterMut<'a, T> {}
```



### 测试用例

**直接搬运原书测试用例**

```rust
#[allow(dead_code)]
fn assert_properties() {
    fn is_send<T: Send>() {}
    fn is_sync<T: Sync>() {}

    is_send::<LinkedList<i32>>();
    is_sync::<LinkedList<i32>>();

    is_send::<IntoIter<i32>>();
    is_sync::<IntoIter<i32>>();

    is_send::<Iter<i32>>();
    is_sync::<Iter<i32>>();

    is_send::<IterMut<i32>>();
    is_sync::<IterMut<i32>>();

    fn linked_list_covariant<'a, T>(x: LinkedList<&'static T>) -> LinkedList<&'a T> {
        x
    }
    fn iter_covariant<'i, 'a, T>(x: Iter<'i, &'static T>) -> Iter<'i, &'a T> {
        x
    }
    fn into_iter_covariant<'a, T>(x: IntoIter<&'static T>) -> IntoIter<&'a T> {
        x
    }

    /// ```compile_fail,E0308
    /// use linked_list::IterMut;
    ///
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
}

#[cfg(test)]
mod test {
    use super::LinkedList;

    fn generate_test() -> LinkedList<i32> {
        list_from(&[0, 1, 2, 3, 4, 5, 6])
    }

    fn list_from<T: Clone>(v: &[T]) -> LinkedList<T> {
        v.iter().map(|x| (*x).clone()).collect()
    }

    #[test]
    fn test_basic_front() {
        let mut list = LinkedList::new();

        // Try to break an empty list
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Try to break a one item list
        list.push_front(10);
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);

        // Mess around
        list.push_front(10);
        assert_eq!(list.len(), 1);
        list.push_front(20);
        assert_eq!(list.len(), 2);
        list.push_front(30);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(30));
        assert_eq!(list.len(), 2);
        list.push_front(40);
        assert_eq!(list.len(), 3);
        assert_eq!(list.pop_front(), Some(40));
        assert_eq!(list.len(), 2);
        assert_eq!(list.pop_front(), Some(20));
        assert_eq!(list.len(), 1);
        assert_eq!(list.pop_front(), Some(10));
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
        assert_eq!(list.pop_front(), None);
        assert_eq!(list.len(), 0);
    }

    #[test]
    fn test_basic() {
        let mut m = LinkedList::new();
        assert_eq!(m.pop_front(), None);
        assert_eq!(m.pop_back(), None);
        assert_eq!(m.pop_front(), None);
        m.push_front(1);
        assert_eq!(m.pop_front(), Some(1));
        m.push_back(2);
        m.push_back(3);
        assert_eq!(m.len(), 2);
        assert_eq!(m.pop_front(), Some(2));
        assert_eq!(m.pop_front(), Some(3));
        assert_eq!(m.len(), 0);
        assert_eq!(m.pop_front(), None);
        m.push_back(1);
        m.push_back(3);
        m.push_back(5);
        m.push_back(7);
        assert_eq!(m.pop_front(), Some(1));

        let mut n = LinkedList::new();
        n.push_front(2);
        n.push_front(3);
        {
            assert_eq!(n.front().unwrap(), &3);
            let x = n.front_mut().unwrap();
            assert_eq!(*x, 3);
            *x = 0;
        }
        {
            assert_eq!(n.back().unwrap(), &2);
            let y = n.back_mut().unwrap();
            assert_eq!(*y, 2);
            *y = 1;
        }
        assert_eq!(n.pop_front(), Some(0));
        assert_eq!(n.pop_front(), Some(1));
    }

    #[test]
    fn test_iterator() {
        let m = generate_test();
        for (i, elt) in m.iter().enumerate() {
            assert_eq!(i as i32, *elt);
        }
        let mut n = LinkedList::new();
        assert_eq!(n.iter().next(), None);
        n.push_front(4);
        let mut it = n.iter();
        assert_eq!(it.size_hint(), (1, Some(1)));
        assert_eq!(it.next().unwrap(), &4);
        assert_eq!(it.size_hint(), (0, Some(0)));
        assert_eq!(it.next(), None);
    }

    #[test]
    fn test_iterator_double_end() {
        let mut n = LinkedList::new();
        assert_eq!(n.iter().next(), None);
        n.push_front(4);
        n.push_front(5);
        n.push_front(6);
        let mut it = n.iter();
        assert_eq!(it.size_hint(), (3, Some(3)));
        assert_eq!(it.next().unwrap(), &6);
        assert_eq!(it.size_hint(), (2, Some(2)));
        assert_eq!(it.next_back().unwrap(), &4);
        assert_eq!(it.size_hint(), (1, Some(1)));
        assert_eq!(it.next_back().unwrap(), &5);
        assert_eq!(it.next_back(), None);
        assert_eq!(it.next(), None);
    }

    #[test]
    fn test_rev_iter() {
        let m = generate_test();
        for (i, elt) in m.iter().rev().enumerate() {
            assert_eq!(6 - i as i32, *elt);
        }
        let mut n = LinkedList::new();
        assert_eq!(n.iter().rev().next(), None);
        n.push_front(4);
        let mut it = n.iter().rev();
        assert_eq!(it.size_hint(), (1, Some(1)));
        assert_eq!(it.next().unwrap(), &4);
        assert_eq!(it.size_hint(), (0, Some(0)));
        assert_eq!(it.next(), None);
    }

    #[test]
    fn test_mut_iter() {
        let mut m = generate_test();
        let mut len = m.len();
        for (i, elt) in m.iter_mut().enumerate() {
            assert_eq!(i as i32, *elt);
            len -= 1;
        }
        assert_eq!(len, 0);
        let mut n = LinkedList::new();
        assert!(n.iter_mut().next().is_none());
        n.push_front(4);
        n.push_back(5);
        let mut it = n.iter_mut();
        assert_eq!(it.size_hint(), (2, Some(2)));
        assert!(it.next().is_some());
        assert!(it.next().is_some());
        assert_eq!(it.size_hint(), (0, Some(0)));
        assert!(it.next().is_none());
    }

    #[test]
    fn test_iterator_mut_double_end() {
        let mut n = LinkedList::new();
        assert!(n.iter_mut().next_back().is_none());
        n.push_front(4);
        n.push_front(5);
        n.push_front(6);
        let mut it = n.iter_mut();
        assert_eq!(it.size_hint(), (3, Some(3)));
        assert_eq!(*it.next().unwrap(), 6);
        assert_eq!(it.size_hint(), (2, Some(2)));
        assert_eq!(*it.next_back().unwrap(), 4);
        assert_eq!(it.size_hint(), (1, Some(1)));
        assert_eq!(*it.next_back().unwrap(), 5);
        assert!(it.next_back().is_none());
        assert!(it.next().is_none());
    }

    #[test]
    fn test_eq() {
        let mut n: LinkedList<u8> = list_from(&[]);
        let mut m = list_from(&[]);
        assert!(n == m);
        n.push_front(1);
        assert!(n != m);
        m.push_back(1);
        assert!(n == m);

        let n = list_from(&[2, 3, 4]);
        let m = list_from(&[1, 2, 3]);
        assert!(n != m);
    }

    #[test]
    fn test_ord() {
        let n = list_from(&[]);
        let m = list_from(&[1, 2, 3]);
        assert!(n < m);
        assert!(m > n);
        assert!(n <= n);
        assert!(n >= n);
    }

    #[test]
    fn test_ord_nan() {
        let nan = 0.0f64 / 0.0;
        let n = list_from(&[nan]);
        let m = list_from(&[nan]);
        assert!(!(n < m));
        assert!(!(n > m));
        assert!(!(n <= m));
        assert!(!(n >= m));

        let n = list_from(&[nan]);
        let one = list_from(&[1.0f64]);
        assert!(!(n < one));
        assert!(!(n > one));
        assert!(!(n <= one));
        assert!(!(n >= one));

        let u = list_from(&[1.0f64, 2.0, nan]);
        let v = list_from(&[1.0f64, 2.0, 3.0]);
        assert!(!(u < v));
        assert!(!(u > v));
        assert!(!(u <= v));
        assert!(!(u >= v));

        let s = list_from(&[1.0f64, 2.0, 4.0, 2.0]);
        let t = list_from(&[1.0f64, 2.0, 3.0, 2.0]);
        assert!(!(s < t));
        assert!(s > one);
        assert!(!(s <= one));
        assert!(s >= one);
    }

    #[test]
    fn test_debug() {
        let list: LinkedList<i32> = (0..10).collect();
        assert_eq!(format!("{:?}", list), "[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]");

        let list: LinkedList<&str> = vec!["just", "one", "test", "more"]
            .iter()
            .copied()
            .collect();
        assert_eq!(format!("{:?}", list), r#"["just", "one", "test", "more"]"#);
    }

    #[test]
    fn test_hashmap() {
        // Check that HashMap works with this as a key

        let list1: LinkedList<i32> = (0..10).collect();
        let list2: LinkedList<i32> = (1..11).collect();
        let mut map = std::collections::HashMap::new();

        assert_eq!(map.insert(list1.clone(), "list1"), None);
        assert_eq!(map.insert(list2.clone(), "list2"), None);

        assert_eq!(map.len(), 2);

        assert_eq!(map.get(&list1), Some(&"list1"));
        assert_eq!(map.get(&list2), Some(&"list2"));

        assert_eq!(map.remove(&list1), Some("list1"));
        assert_eq!(map.remove(&list2), Some("list2"));

        assert!(map.is_empty());
    }

    #[test]
    fn test_cursor_move_peek() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 1));
        assert_eq!(cursor.peek_next(), Some(&mut 2));
        assert_eq!(cursor.peek_prev(), None);
        assert_eq!(cursor.index(), Some(0));
        cursor.move_prev();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.current(), Some(&mut 2));
        assert_eq!(cursor.peek_next(), Some(&mut 3));
        assert_eq!(cursor.peek_prev(), Some(&mut 1));
        assert_eq!(cursor.index(), Some(1));

        let mut cursor = m.cursor_mut();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 6));
        assert_eq!(cursor.peek_next(), None);
        assert_eq!(cursor.peek_prev(), Some(&mut 5));
        assert_eq!(cursor.index(), Some(5));
        cursor.move_next();
        assert_eq!(cursor.current(), None);
        assert_eq!(cursor.peek_next(), Some(&mut 1));
        assert_eq!(cursor.peek_prev(), Some(&mut 6));
        assert_eq!(cursor.index(), None);
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.current(), Some(&mut 5));
        assert_eq!(cursor.peek_next(), Some(&mut 6));
        assert_eq!(cursor.peek_prev(), Some(&mut 4));
        assert_eq!(cursor.index(), Some(4));
    }

    #[test]
    fn test_cursor_mut_insert() {
        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.splice_before(Some(7).into_iter().collect());
        cursor.splice_after(Some(8).into_iter().collect());
        // check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[7, 1, 8, 2, 3, 4, 5, 6]
        );
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        cursor.splice_before(Some(9).into_iter().collect());
        cursor.splice_after(Some(10).into_iter().collect());
        check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[10, 7, 1, 8, 2, 3, 4, 5, 6, 9]
        );

        /* remove_current not impl'd
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), None);
        cursor.move_next();
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(7));
        cursor.move_prev();
        cursor.move_prev();
        cursor.move_prev();
        assert_eq!(cursor.remove_current(), Some(9));
        cursor.move_next();
        assert_eq!(cursor.remove_current(), Some(10));
        check_links(&m);
        assert_eq!(m.iter().cloned().collect::<Vec<_>>(), &[1, 8, 2, 3, 4, 5, 6]);
        */

        let mut m: LinkedList<u32> = LinkedList::new();
        m.extend([1, 8, 2, 3, 4, 5, 6]);
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        let mut p: LinkedList<u32> = LinkedList::new();
        p.extend([100, 101, 102, 103]);
        let mut q: LinkedList<u32> = LinkedList::new();
        q.extend([200, 201, 202, 203]);
        cursor.splice_after(p);
        cursor.splice_before(q);
        check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[200, 201, 202, 203, 1, 100, 101, 102, 103, 8, 2, 3, 4, 5, 6]
        );
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_prev();
        let tmp = cursor.split_before();
        assert_eq!(m.into_iter().collect::<Vec<_>>(), &[]);
        m = tmp;
        let mut cursor = m.cursor_mut();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        cursor.move_next();
        let tmp = cursor.split_after();
        assert_eq!(
            tmp.into_iter().collect::<Vec<_>>(),
            &[102, 103, 8, 2, 3, 4, 5, 6]
        );
        check_links(&m);
        assert_eq!(
            m.iter().cloned().collect::<Vec<_>>(),
            &[200, 201, 202, 203, 1, 100, 101]
        );
    }

    fn check_links<T: Eq + std::fmt::Debug>(list: &LinkedList<T>) {
        let from_front: Vec<_> = list.iter().collect();
        let from_back: Vec<_> = list.iter().rev().collect();
        let re_reved: Vec<_> = from_back.into_iter().rev().collect();

        assert_eq!(from_front, re_reved);
    }
}
```

