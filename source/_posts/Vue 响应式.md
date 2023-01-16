---
title: Vue 响应式
---

## 1 - 整体目标

- [x] 了解 <font face="Times New Roman" color=#FFA500>Object.defineProperty</font> 实现原理

- [x] 了解 <font color=#FFA500>指令编译</font> 基础原理

## 2 - 响应式

### 2.1 什么是响应式

响应式指组件 data 的数据一旦变化，立即触发视图的更新。它是实现数据驱动视图的第一步。

### 2.2 如何实现响应式

在 JavaScript 里实现数据响应式一般有两种方案：<font face="Times New Roman" color=#FFA500>Object.defineProperty</font> 和 <font face="Times New Roman" color=#FFA500>Proxy</font>

1. <font face="Times New Roman" color=#FFA500>Vue2.x</font> 的响应式核心是 <font face="Times New Roman" color=#FFA500>Object.defineProperty</font>
2. <font face="Times New Roman" color=#FFA500>Vue3.x</font> 的响应式核心是 <font face="Times New Roman" color=#FFA500>Proxy</font>

### 2.3 实现对象属性拦截

> <font face="Times New Roman" color=#FFA500>Object.defineProperty()</font> 方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性，并返回此对象。

``` JavaScript
	const data = {}
	let _name = '章三'	

	Object.defineProperty(data, 'name', {
		// name 属性返回值
		get() {
			return _name
		},
		// newVal 是设置的新值	
		set(newVal) {
			name = val
		}
	})

	console.log(data.name) // get()
	data.name = '里斯' // set()
```

利用 <font face="Times New Roman" color=#FFA500>Object.defineProperty</font> 重写 <font face="Times New Roman" color=#FFA500>get</font> 和 <font face="Times New Roman" color=#FFA500>set</font>，将对象属性的赋值和获取变成函数，可以实现一个简单的双向绑定。

### 2.4 如何监听 data 的变化

共定义三个函数：

- updateView: 模拟 Vue 更新视图的入口函数。
- defineReactive: 对数据进行监听的具体实现。
- observer: 调用函数后，可对目标对象进行监听，将目标对象变成响应式。

``` JavaScript
	// 触发更新视图
	function updateView() {
		console.log('视图更新')
	}

	// 监听属性
	function defineReactive(target, key, value) {
		Object.defineProperty(target, key, {
			configurable: true,
			enumerable: true,
			get() {
				return value
			},
			set(newVal) {
				if (value !== newVal) {
					value = newVal
					updateView()
				}
			}
		})
	}

	// 监听对象属性
	function observer(target) {
		// 监听的不是对象或数组，直接返回
		if (typeof target !== 'object' || target === null) {
			return target
		}

		for (let key in target) {
			defineReactive(target, key, target[key])
		}
	}
```

> 每次执行 <font face="Times New Roman" color=#FFA500>defineReactive</font> 函数时，都会形成一个独立的函数作用域，传入的 <font face="Times New Roman" color=#FFA500>value</font> 因为闭包的关系会常驻内存；这样，每个 <font face="Times New Roman" color=#FFA500>defineReactive</font> 函数的 <font face="Times New Roman" color=#FFA500>value</font> 作为各自 <font face="Times New Roman" color=#FFA500>get</font> 和 <font face="Times New Roman" color=#FFA500>set</font> 操作的局部变量。

``` JavaScript
	const data = {
		name: '章三',
		age: 18,
	}

	// 监听数据
	observer(data)

	// 测试
	data.name = '里斯'
	data.age = 21
```


### 2.5 深度监听 data 的变化

有嵌套属性的对象

``` JavaScript
	const data = {
		name: '章三',
		age: 18,
		info: {
			address: '岳家嘴'
		}
	}
```


- 在刚进入 <font face="Times New Roman" color=#FFA500>defineReactive</font> 函数时，先调用 <font face="Times New Roman" color=#FFA500>observer</font>，基本类型属性在 <font face="Times New Roman" color=#FFA500>observer</font> 中被直接返回；由于 <font face="Times New Roman" color=#FFA500>info</font> 是对象，所以会对 <font face="Times New Roman" color=#FFA500>info</font> 遍历后执行  <font face="Times New Roman" color=#FFA500>defineReactive</font>。
- 新值有可能是个对象，在设置新值时也需要对新值进行深度监听。

``` JavaScript
	// 监听属性
	function defineReactive(target, key, value) {
		// 深度监听
		observer(value)

		Object.defineProperty(target, key, {
			configurable: true,
			enumerable: true,
			get() {
				return value
			},
			set(newVal) {
				if (newVal !== value) {
					// 深度监听
					observer(newVal将)
					value = newVal
					updateView()
				}
			}
		})
	}
```


### 2.6 如何监听数组的变化

对于数组，通过重写数组方法来实现

- 先备份将原生数组原型（防止后续的操作污染原生数组原型），基于该备份创建一个新的数组，并扩展它的方法。
- observer 方法中增加对数组的处理。

``` JavaScript
	// 触发更新视图
	function updateView() {
		console.log('视图更新')
	}

	// 重新定义数组原型
	const arrayPrototype = Array.prototype
	// 创建对象
	const arrObj = Object.create(arrayPrototype)
	['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse'].forEach(method => {
		arrObj[method] = function() {
			updateView()
			arrayPrototype[method].call(this, ...arguments)
		}
	})

	// 监听属性
	function defineReactive(target, key, value) {
		Object.defineProperty(target, key, {
			configurable: true,
			enumerable: true,
			get() {
				return value
			},
			set(newVal) {
				if (newVal !== value) {
					value = newVal
					updateView()
				}
			}
		})
	}

	// 监听对象属性
	function observer(target) {
		// 监听的不是对象或数据，直接返回
		if (typeof target !== 'object' || target === null) {
			return target
		}

		if (Array.isArray(target)) {
			target.__proto__ = arrObj
		}

		for (let key in target) {
			defineReactive(target, key, target[key])
		}
	}
```

测试

``` JavaScript
	const data = {
		name: '章三',
		age: 18,
		hobbies: ['basketball', 'one-piece'] // 需要监听的数组
	}

	// 监听数据
	observer(data)

	// 测试
	data.hobbies.push('football')
```


## 3 - 数据的变化触发视图的更新

> 将 data 中 name 的值作为文本渲染到标记了 v-text 的标签内部

``` HTML
	<div>
		<span v-text="name"></span>
	</div>

	// 触发更新视图
	function updateView() {
		const app = document.getElementById('app')
		const nodes = app.childNodes
		nodes.forEach(node => {
			// nodeType 为 1 表示元素节点
			if (node.nodeType === 1) {
				const attrs = node.attributes
				Array.from(attrs).forEach(attr => {
					const name = attr.name
					const prop = attr.value
					if (name === 'v-text') {
						node.innerText = data[prop]
					}
				})
			}
		})
	}

	// 首次渲染
	updateView()
```


## 4 - 视图的变化同步到数据

> 将 data 中 name 的值渲染到 input，如果 input 的值发生修改，反向同步到 message。

``` HTML
	<div id="app">
		<input v-model="name" />
	</div>

	// 触发更新视图
	function updateView() {
		const app = document.getElementById('app')
		const nodes = app.childNodes
		nodes.forEach(node => {
			// nodeType 为 1 表示元素节点
			if (node.nodeType === 1) {
				const attrs = node.attributes
				Array.from(attrs).forEach(attr => {
					const name = attr.name
					const prop = attr.value
					if (name === 'v-model') {
						node.value = data[prop]
						// 事件监听反向修改
						node.addEventListener('input', () => {
							data[prop] = e.target.value
						})
					}
				})
			}
		})
	}

	// 首次渲染
	updateView()
```



