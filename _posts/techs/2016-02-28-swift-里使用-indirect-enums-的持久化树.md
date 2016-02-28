---
layout: post
title: Swift 里使用 indirect enums 的持久化树
category: 开发
tags: Swift enums
keywords: Swift, Enums, Black-red tree
description:
---

## 前言
UPDATE：升级到了 Swift 2.1b2 <br /> <br />
设想你需要一个 “有序 Set". Set 很棒，但它是无序的。尽管如此，你仍然可以使用 Set，当有必要时再去排序。或者使用 Array 并保持有序，然后将新元素插入到正确的位置。或者，你可以用树。                   
在 Swift 里建树的方法之一是使用 class 并能指向自身。像二叉搜索树：
	
	class Tree<Element: Comparable> {
	    let value: Element
	    // entries < value go on the left
	    let left: Tree<Element>?
	    // entries > value go on the right
	    let right: Tree<Element>?
	}

从此树的逻辑看，Element 只需要 Comparable，无须 Hashable， 这也是比 Set 更好的一个潜在优势。<br />
左右子树都是 Optional 的，当没有比当前节点大或小的值时，左右子树就是 nil 了。但是，这种方案有一问题：怎么表示一颗空树呢？树的值不是 optional 的。当然可以将 Tree 的 value 设置成 optional，通过设置 value 为 nil 来表示一颗空树。但是这看着不爽。<br /> <br />
讲真，最好通过 “带 associated type 的 enum ” 来构建树。像这样：

	enum Tree<Element: Comparable> {
	    case Empty
	    case Node(Tree<Element>,Element,Tree<Element>)
	}
	
这样，树可以是空的，或者有左右子树和值。子树不必再设置成 optional，因为如果它们为空，将其设置成 .Empty 就行了。

*** 

## No more boxes

不过，上面通过 enum 来定义的 tree 是不行的。 Swift 的 enum 是值类型的，因此它们不能像上面那样 contain 自身。需要使用像 class 这样的引用类型的实例。 <br /> <br />
直到 2.0b4，你不得不使用 Box trick - 在 enum 之间插入一个 class 

	final class Box<Element> {
	    let unbox: Element
	    init(_ x: Element) { self.unbox = x }
	}
	 
	enum Tree <Element: Comparable> {
	    case Empty
	    case Node(Box<Tree<Element>>,Element,Box<Tree<Element>>)
	}

这样的代码有点臃肿，何况我们还没有实现任何逻辑呢。 Box 将会使代码变得乱成一团。 <br />
但是最新版本的 Swift 为 enums 引入了 indirect 关键词。 使用 indirect 关键词就像创建一个上面的 Box class 的透明版本，提供一个封装的 value 的 “指引”。 Xcode 在编程中也会有提示。 <br />
现在可以这样写了：

	indirect enum Tree<Element: Comparable> {
	    case Empty
	    case Node(Tree<Element>,Element,Tree<Element>)
	}

但是，我们需要的是平衡树。二叉搜索树也可以使用随机数据，但是当向其中插入已经排序的数据时，它将退化成链表(因为插入只能发生在树的一侧)。<br />
使用红黑树吧。红黑树是二叉搜索树的一种，它的节点是红色或者黑色，并且使用一些不变量来保证在插入时保持平衡。我们将结合 Swift 特性来完成一个相对简单的红黑树。<br />
这样，我们添加 colour enum，并加上一些 initializers 来简化一下：

	enum Color { case R, B }
 
	indirect enum Tree<Element: Comparable> {
	    case Empty
	    case Node(Color,Tree<Element>,Element,Tree<Element>)
 
	    init() { self = .Empty }
 
	    init(_ x: Element, color: Color = .B,
	        left: Tree<Element> = .Empty, right: Tree<Element> = .Empty)
	    {
	        self = .Node(color, left, x, right)
	    }
	}
	
使用 R 和 B 来代替 Red 和 Black 是原作者的爱好，这样可以简化一下。理由是：a，红和黑只是随机选择出的颜色，b，这样使 pattern matching 简短。<br />
顺便说一下，上述实现是 Chris Okasaki’s [Purely Functional Data Structures](http://www.amazon.com/gp/product/0521663504/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0521663504&linkCode=as2&tag=airspveloc0d-20&linkId=WYQPW4P6S37XTMSN) 所讲的 ML version 的 Swift 的翻译。上述这本书值得读。 Swift 2.0 里的新特性将使 Swift 在表达力和简化性上很接近 ML 版本。

***

## 判断 Tree 是否包含某个元素
从 **contains** 入手。我们将遍历树，检查每个 value 是否为空，如果不为空，是比我们查找的元素的 value 大还是小。<br />
写一个带 **guard** 的函数：

	extension Tree {
	    func contains(x: Element) -> Bool {
	        guard case let .Node(_,left,y,right) = self
	          else { return false }
	 
	        if x < y { return left.contains(x) }
	        if y < x { return right.contains(x) }
	        return true
	    }
	} 
	
**guard** 真的很有帮助。它帮我们 unpack 函数的主体要用的重要的东西（是 node 吗 ？如果是，获得其左右子树和值，抛弃颜色，因为颜色不重要）。这也帮我们处理了失败的情况，如果得到了空节点，这样就知道树不包括这个 element.除此，如果元素小于当前元素，寻找左子树，否则寻找右子树。<br />
我们可以依赖 Comparable elements 必须是 [strict total order](https://en.wikipedia.org/wiki/Total_order#Strict_total_order) 这一事实。这就意味着，x 只能是小于，大于或等于所寻找的元素的 value 中的一种情况，因此，我们可以找到这个 element 并且 contains is true.

***

## 插入元素

**insert** 有点麻烦，红黑树的插入和无序 Set.insert 的 in-place 插入不一样。我们将定义一个 insert 方法，返回一颗新元素被插入的新树。 <br />
为了完成上述目的，我们将创建一种“持久化”的数据结构 -- 在 update 的时候不改变旧结构。看上去像是在 update 的时候做了非常多的无用的 copy，但是事实不是这样的。因为树的节点都是只读的，两棵树可以共享节点。只有那些实际需要的变化的节点才被拷贝。<br />
首先，看看 insert 方法，它做了一点：保持红黑树的平衡，即 root node 必须是黑色的。大部分工作都由辅助函数完成

	private func ins<T>(into: Tree<T>, _ x: T) -> Tree<T> {
	    guard case let .Node(c, l, y, r) = into
	        else { return Tree(x, color: .R) }
	 
	    if x < y { return balance(Tree(y, color: c, left: ins(l,x), right: r)) }
	    if y < x { return balance(Tree(y, color: c, left: l, right: ins(r, x))) }
	    return into
	}
 
 
	extension Tree {
	    func insert(x: Element) -> Tree {
	        guard case let .Node(_,l,y,r) = ins(self, x)
	            else { fatalError("ins should never return an empty tree") }
	 
	        return .Node(.B,l,y,r)
	    }
	}
	
如上，如果我们找到空节点，就把新的 value 放在这里，新节点总是红色的。<br />
要不就是把 value 插入左子树或者右子树。如果新的 element 和已有的相同，那就不要 insert 了，因为我们在这里实现的是类似 set 的行为。<br />
还需要 re-balance 红黑树：

	case let .Node(.B, .Node(.R, a, x, .Node(.R, b, y, c)), z, d):
    	return .Node(.R, .Node(.B, a, x, b), y, .Node(.B, c, z, d))

完整的 balancing function 如下：

	private func balance<T>(tree: Tree<T>) -> Tree<T> {
	    switch tree {
	    case let .Node(.B, .Node(.R, .Node(.R, a, x, b), y, c), z, d):
	        return .Node(.R, .Node(.B,a,x,b),y,.Node(.B,c,z,d))
	    case let .Node(.B, .Node(.R, a, x, .Node(.R, b, y, c)), z, d):
	        return .Node(.R, .Node(.B,a,x,b),y,.Node(.B,c,z,d))
	    case let .Node(.B, a, x, .Node(.R, .Node(.R, b, y, c), z, d)):
	        return .Node(.R, .Node(.B,a,x,b),y,.Node(.B,c,z,d))
	    case let .Node(.B, a, x, .Node(.R, b, y, .Node(.R, c, z, d))):
	        return .Node(.R, .Node(.B,a,x,b),y,.Node(.B,c,z,d))
	    default:
	        return tree
	    }
	}

注意到上述 pattern match 在 enum 里有 3 层深度的递归。这也是摆脱 **Box** 的关键，在使用 indirect 之前，它是这样的：

	if case let .Node(.B,l,z,d) = tree, 
	case let .Node(.R,ll,y,c) = l.unbox, 
	case let .Node(.R,a,x,b) = ll.unbox {
	    return .Node(.R, Box(.Node(.B,a,x,b)),y,Box(.Node(.B,c,z,d)))
	}
	
这里 return 和上面全部 4 种 rebalance patterns 一样。基于是上面 4 种情况中的哪一种来改变 pattern match. 

可能下面的版本更好：

	switch tree {
		case let .Node(.B, .Node(.R, .Node(.R, a, x, b), y, c), z, d),
		         .Node(.B, .Node(.R, a, x, .Node(.R, b, y, c)), z, d),
		         .Node(.B, a, x, .Node(.R, .Node(.R, b, y, c), z, d)),
		         .Node(.B, a, x, .Node(.R, b, y, .Node(.R, c, z, d))):
		    return .Node(.R, .Node(.B,a,x,b),y,.Node(.B,c,z,d))
		default:
		    return tree
	}

当 pattern 有变量的时候， Swift 在一个 case 里有 multiple patterns. 所以，我们不得不使用像上面的多返回值。


# 未完待续 































