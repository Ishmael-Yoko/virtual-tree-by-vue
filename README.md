# virtual-tree

## virtual-tree是一个基于vue实现的tree组件

## 原理就是将树形结构数据扁平化为一个列表，在当前可视区域内封装一个列表，在父组件绑定滑动事件，页面滑动的时候通过js动态切换当前可视区域的列表。

## 功能

1. 支持同步数据

  同步只需要传入`tree`(树形的数据)参数即可

```js
  // 数据结构
  cosnt tree = [{ id: 1, label: 'sss', children: []}]
```

2. 支持异步加载数据

  第一次需要指定tree的第一级数据，下级通过封装的 `load`方法加载下级数据

```js
// load方法必须返回一个promise 下级数据通过promise返回数据
```

3. 结合输入框实现过滤

```js
// 传入`filterMethod`方法，第一个参数为当前值，第二个对象为当前节点对象, 返回true和false
// true表示满足条件 false不满足条件
```

4. 支持整个树的展开闭合

```js
// 整个树的展开expandAll，闭合collapseAll分两种情况
// 1. 同步树是可以直接使用
// 2. 异步树，已经展开的已经有缓存了，可以使用，但是没有展开的就不能使用
```

5. 支持选中状态

```js
// $refs.tree.setCurrent方法，参数为需要选中的对象或者id
```

## 参数 Options

### tree: Array required:true
树形数据,默认[]

### option: Object:{itemHeight: 25} required:false
 树每个节点高度

### lazy: Boolean required:false
是否支持懒加载,默认false

### load: Function required:false
加载下一级方法，参数为当前节点对象

### filterMethod: required:false
过滤条件，第一个参数为当前值，第二个对象为当前节点对象, 返回true和false

## 事件 Events

### node-click 
点击当前节点的触发的事件
参数为当前节点对象，父级对象

### expand

### collapse


## 方法 Methods

### setCurrent
参数为需要选中的对象或者id