# A Simple React

[Demo](https://oneonetwo.github.io/handwrite-a-simple-react/)
遵循React的Fiber架构，一步步实现自己的mini版本React，以此深入的Fiber架构深层的原理。  

我们需要做的事情: 
- Step 1: createElement Function
- Step 2: render Function
- Step 3: 任务调度
- Step 4: 构建Fibers
- Step 5: Render阶段 Commit阶段
- Step 6: Reconciliation
- Step 7: 函数组件Function Components
- Step 8: Hooks

---

## 回顾
了解React，JSX和DOM元素的工作原理
1. React能做的 
    - 1.定义一个React元素。
    - 2.从DOM获取一个节点。
    - 3.将React元素渲染到容器中。
    ```javascript
	const element = <h1 title="foo">Hello</h1>
	const container = document.getElementById("root")
	ReactDOM.render(element, container)
    ```
2. JSX
    - Babel把JSX转义，并且调用用`createElement`替换每个Tag
    ```javascript 12
	//Babel转义
	const element = React.createElement(
		'h1',
		{ title: 'foo'},
		'hello'
	)
    ```
    - `React.createElement`根据参数返回一个js对象，这就是元素，具有两个属性（type和props，它具有更多的属性，我们目前只使用这两个）
        1. type 是指定节点类型，也可以是一个函数 
        2. props是个对象，包含了当前节点的所有的属性的键值，还有一个很重要的属性： children;
        3. children如果是个数组，可以包含多个子节点，如果是个字符串，表示个是文本节点
    ```javascript
    //调用createElement返回
	const element = {
	   type: 'h1',
	   props: {
		   title: 'foo',
		   children: 'hello'
	   }
	}
    ```
3. ReactDOM.render
    - render是React渲染更新dom的地方；
    ```javascript
	const container = document.getElementById('root');
	const node = document.createElement(element.type);
	node["title"] = element.props.title;
	const text = document.createTextNode('');
	text['nodeValue'] = element.props.children;
	node.appendChild(text);
	container.appendChild(node);
    ```
4. 到此，我们已经把一组react元素渲染到React;
---

## createElement Function

1. 作用：把JSX返回的js对象，创建成包含props、children的js对象
    ```javascript
	const element = (
		<div id="foo">
			<a>bar</a>
			<b />
		</div>
	)
	//jsx转义
	const element = React.createElement(
		'div',
		{ id: 'foo'},
		React.createElement('a', null, 'bar'),
		React.createElement('b')
	)
	//createElement 调用之后
	const element = {
		type: 'div',
		props: {
			id: 'foo',
			children: [{
				type: 'a',
				props: {
					children: [{
						type: 'TEXT_ELEMENT',
						nodeValue: 'bar'
					}]
				}
			},{
				type: 'b'
			}]                
		}
	}
	//由此可见
	function createElement(type, props, ...children){
		return {
			type,
			props: {
				...props,
				children
			}
		}
	}        
    ```
2. 实现一个 `createElement`,为文本节点创建一个特殊的类型：`TEXT_ELEMENT`
    ```javascript
	function createElement(type, props, ...children){
		return {
			type,
			props: {
				...props,
				children: children.map(child =>{
					return typeof child === 'object'
					?child
					:createTextElement(child);
				})
			}
		}
	} 
	function createTextElement(text){
		return {
			type: "TEXT_ELEMENT",
			props: {
				nodeValue: text,
				children: []
			}
		}
	}    
    ```
3. 测试createElement
    ```javascript
	const element = createElement(
		'div',
		{ id: 'foo'},
		createElement('a', null, 'bar'),
		createElement('b')
	)
    ```    
---

## render Function

1. 先考虑往DOM添加内容，稍后处理更新和删除
    ```javascript
	function render(element, container){
		const dom = document.createElement(element.type);
		container.appendChild(dom);
	}
    ```
2. 递归添加每个children
    ```javascript
	function render(element, container){
		const dom = document.createElement(element.type);
		element.props.children,forEach((child)=>{
			render(child, dom);
		})
		container.appendChild(dom);
	}
    ```
3. 处理文本节点，元素类型是TEXT_ELEMENT的文本节点
    ```javascript
	function render(element, container){
		const dom = element.type === 'TEXT_ELEMENT'
					?document.createTextNode('')
					:document.createElement(element.type);
		//添加节点的属性
		const isProperty = key => key!=='children';
		Object.keys(element.props)
			.filter(isProperty)
			.forEach(name =>{
				dom[name] = element.props[name];
			})

		element.props.children.forEach((child)=>{
			render(child, dom);
		})
		container.appendChild(dom);
	}
    ```
4. 测试render
    ```javascript
	const container = document.getElementById('root');
	const element = createElement(
		'div',
		{ id: 'foo'},
		createElement('a', null, 'bar'),
		createElement('b')
	)
	render(element, container);
    ```
---

## [任务调度](https://github.com/acdlite/react-fiber-architecture#scheduling)
1. 确定react何时进行更新渲染工作；
    - 上面render方法中，递归渲染每个元素，一旦渲染开始，就会到整个DOM树完成才会结束，如果元素结构很复杂，那将会渲染很长的时间阻塞主线程，而且浏览器如果需要处理更高优先的操作（例如用户输入或保持动画的流畅），则需要等到渲染完成。
    - 因此需要把工作分成多个单元来操作，在完成一个单元后，如果需要执行其他的高优先级的操作，那么让浏览器中断渲染。
    - 当前，React仍然没有充分利用调度的优势，更新会导致渲染整个子树。彻底革新React的核心算法以利用调度是Fiber背后的驱动思想。
2. 我们代码中用`requestIdleCallback`来做一个循环，但是React不使用requestIdleCallback,现在使用的是 [`scheduler`](https://react.jokcy.me/book/flow/scheduler-pkg.html) ,概念上是一样的；
    - 因为requestIdleCallback这个 API 目前还处于草案阶段，所以浏览器实现率还不高，所以在这里 React 直接使用了polyfill的方案。这个方案简单来说是通过requestAnimationFrame在浏览器渲染一帧之前做一些处理，然后通过postMessage在macro task（类似 setTimeout）中加入一个回调，在因为接下去会进入浏览器渲染阶段，所以主线程是被 block 住的，等到渲染完了然后回来清空macro task。总体上跟requestIdleCallback差不多，等到主线程有空的时候回来调用
3. 实现 wookloop方法
    ```javascript
	//全局变量，当前的工作单元Fiber
	let nextUnitOfwork = null;

	//workLoop逻辑很简单的，只是判断是否需要继续调用performUnitOfWork；
	function workloop(deadline) {
		let shouldYield = false;
		while(nextUnitOfwork&&!shouldYield){
			nextUnitOfwork = performUnitOfwork(
				nextUnitOfwork
			)
			shouldYield = deadline.timeRemaining()<1;
		}
		requestIdleCallback(workloop);
	}

	requestIdleCallback(workloop);

	function performUnitOfwork(nextUnitOfwork){
		//TODO
	}

    ```
---

## [构建Fibers](https://github.com/acdlite/react-fiber-architecture#what-is-a-fiber)
1. 概念：
	- Fiber	
    	1. 主要目的是进行**增量式渲染**，将任务切片。并且分布到多个帧；
    	2. 使React充分利用scheduling的优势，
    	3. A fiber represents a unit of work. 一个fiber代表一个工作单元
	- Fiber是一个**树结构**，child跟sibling是一个child为首的**链表**；  
	![Fiber Tree](http://static.irmvp.com/pro/fiber4.png)
2. `performUnitOfWork`方法，做三件事
    - 添加元素到dom
    - 为元素的children创建fiber
    - 选择下一个next unit of work，并return  
		1. 当我们完成 performing 中的fiber时，如果有child则作为下一个fiber(工作单元),如果没有孩子则找sibling作为下一个工作单元，如果没有child也没sibling，则找parent的sibling也就是叔叔，
		2. 找fiber的过程，使用**深度优先**
3. 代码实现
	- 改变`render`,将创建DOM节点保留在自身的dom中，在最后commit阶段在使用,在render函数中，设置`nextUnitOfWork`为fiber的root
	```javascript
	//将创建DOM节点保留在自身的dom
	function createDom(fiber) {
		const dom =
			fiber.type == "TEXT_ELEMENT"
			? document.createTextNode("")
			: document.createElement(fiber.type)
		const isProperty = key => key !== "children"
		Object.keys(fiber.props)
			.filter(isProperty)
			.forEach(name => {
				dom[name] = fiber.props[name]
			})
		return dom
	}
	//设置`nextUnitOfWork`为fiber的root
	function render(element, container) {
		nextUnitOfWork = {
			dom: container,
			props: {
				children: [element],
			}
		}
	}

	let nextUnitOfWork = null;
		
	```
	- 当浏览器空闲时，启动workloop，`performUnitOfWork`运行root fiber
	```javascript
	function performUnitOfWork(fiber){
		//主要做三步1.add dom node  2.create new fiber  3.return next unit of work
		//1.创建当前fiber的DOM添加到dom属性上。
		if(!fiber.dom){
			fiber.dom = create(fiber);
		}			
		if (fiber.parent) {
			fiber.parent.dom.appendChild(fiber.dom)
		}
		//2.为每个child创建fiber
		const elements = fiber.props.children;
		let index = 0;
		let prevSibling = null;
		while(index<elements.length){
			let element = elements[index];
			let newFiber = {
				type: element.type,
				props: element.props,
				parent: fiber,
				dom: null
			}

			//创建一个兄弟链表,首个孩子为头部
			if(index === 0){
				fiber.child = newFiber;            
			}else{
				prevSibling.sibling = newFiber;
			}
			prevSibling = newFiber;
			index++;
		}

		//3.找下一个fiber(工作单元)，深度优先遍历，先子后兄，再parent的兄弟
		if(fiber.child){
			return fiber.child;
		}
		let nextFiber = fiber;
		while(nextFiber){
			if(nextFiber.sibling){
				return nextFiber.sibling
			}
			nextFiber = newFiber.parent;
		}
	}
	```
---

## Render阶段 Commit阶段
1. 这时还有个问题，当浏览器中断工作时，我们将看不到完成的UI，所以我们要修改dom挂载的部分
	```javascript
	function render(element, container) {
		//设置wipRoot为根
		wipRoot = {
			dom: container,
			props: {
				children: [element],
			},
		}
		nextUnitOfWork = wipRoot
	}
	let nextUnitOfWork = null
	let wipRoot = null
	```
2. 添加commitRoot到Dom
	```javascript
	function workLoop(deadline) {
		//在workloop 添加以下代码
		//一旦没有下一个fiber 那么表示工作完成，调用commitRoot把整个fiber添加到Dom
		if (!nextUnitOfWork && wipRoot) {
			commitRoot()
		}

	}
	//在commitRoot函数中将所有节点递归到dom。
	function commitRoot() {
		// TODO add nodes to dom
		commitWork(wipRoot.child)
		wipRoot = null
	}
	function commitWork(fiber){
		if(!fiber){
			return;
		}
		let parentDom = fiber.parent.dom;
		parentDom.appendChild(fiber.dom);
		commitWork(fiber.child);
		commitWork(fiber.sibling);
	}
	```
---

## [reconciliation](https://github.com/acdlite/react-fiber-architecture#what-is-reconciliation)

1. 概念：
	- 作用
		1. 比较两棵树之间的不同，确定需要更新的地方；
		2. 协调时'VDOM'背后的算法；
	- 目的：能够以出色的性能重新渲染整个应用程序；
	- 基于两个特征：
		1. **React对不同类型的组件直接进行替换**
		2. **对待列表元素则是根据Key进行区分，所以Key应该是可预见的，稳定的，唯一的。**
2. 实现reconciliation方法
	- 我们在完成工作提交之前需要保存currentRoot(旧树),并为每个fiber添加alternate属性，记录相对应的old fiber
	```javascript
	function commitRoot(){
		//+ 
		currentRoot = wipRoot;
	}
	function render(element, container){
		//+
		wipRoot = {
			dom: container,
			props: {
				children: [element],
			}
			alternate: currentRoot
		}
		nextUnitOfWork = wipRoot;
	}
	//+ 全局变量
	let currentRoot = null;
	```
	- 在`performUnitOfWork`提取创建fiber的代码，放到`reconcileChildren`中
	```javascript
	function reconcileChildren(wipFiber, elements){
		let index = 0;    
		//添加old fiber的子级，赋值给oldFiber;
		let oldFiber = wipFiber.alternate?.child;
		let prevSibling = null;
		while(index<element.length || oldFiber!= null){
			let element = elements[index];
			let newFiber = null;

			//比较oldFiber跟element,查看我们上是否需要进行更改
			let sameType = element&&oldFiber&&element.type === oldFiber.type;
			if(sameType){
				//同类型的元素，进行 update
			}
			if(element && !sameType){
				//不同类型的元素，进行 add
			}
			if(oldFiber && !sameType){
				//element不存在，进行删除oldFiber
			}

			//获取链表的下一个元素，进行比较
			if (oldFiber) {
				oldFiber = oldFiber.sibling
			}
		}
	}
	```
	- 继续完善，我们在Fiber上添加 `effectTag`属性，做标记；将会在commitRoot阶段使用这个属性
	```javascript
	function reconcileChildren(wipfiber, elements){
		let index = 0;    
		//添加old fiber的子级，赋值给oldFiber;
		let oldFiber = wipfiber.alternate?.child;
		let prevSibling = null;
		while(index<element.length || oldFiber!= null){
			let element = elements[index];
			let newFiber = null;

			//比较oldFiber跟element,查看我们上是否需要进行更改
			let sameType = element&&oldFiber&&element.type === oldFiber.type;
			if(sameType){
				//同类型的元素，进行 update
				//UPDATE表示更新
				newFiber = {
					type: element.type,
					props: element.props,
					dom: oldFiber.dom,
					parent: wipFiber,
					alternate: oldFiber,
					effectTag: "UPDATE"
				}

			}
			if(element && !sameType){
				//不同类型的元素，进行 add
				//PLACEMENT表示新增
				newFiber = {
					type: element.type,
					props: element.props,
					dom: null,
					parent: wipFiber,
					alternate: null,
					effectTag: "PLACEMENT",
				}
			}
			if(oldFiber && !sameType){
				//element不存在，进行删除oldFiber,
				//新加全局的待删除的数组 let deletions = null
				//在render 中初始化数组 deletions = []
				oldFiber.effectTag = "DELETION";
				deletions.push(oldFiber);
			}

			//获取链表的下一个元素，进行比较
			if (oldFiber) {
				oldFiber = oldFiber.sibling
			}
		}
	}
	```
	- 更改`commitWork`处理 effectTags
	```javascript
	function commitWork(fiber) {
		if (!fiber) {
			return
		}
		const domParent = fiber.parent.dom;
		if(fiber.effectTag === "PLACEMENT" && fiber.dom!==null){
			//fiber标记是PLACEMENT，则与之前相同，将DOM节点追加到父节点;
			domParent.appendChild(fiber.dom)
		}else if(fiber.effectTag === "UPDATE" && fiber.dom != null){
			//fiber标记是Update，使用fiber更新现有的节点;
			updateDOM(
				fiber.dom,
				fiber.alternate.props,
				fiber.props
			)
		}else if(fiber.effectTag === "DELETION"){
			//fiber标记是DELETION，则从父节点中移除当前的节点;
			domParent.removeChild(fiber.dom)
		}    

		commitWork(fiber.child)
		commitWork(fiber.sibling)
	}

	function updateDom(dom, prevProps, nextProps) {
	  // TODO
	}
	```
	- 继续 updateDom, 新旧props做对比，删除旧的prop,添加新的prop;
	```javascript
	const isProperty = key => key !== "children";
	const isNew = (prev,next) => key => prev[key] !== next[key];
	const isGone = (prev, next) => key => !(key in next);

	function updateDom(dom, prevProps, nextProps) {
	  	// TODO
	  	//删除旧的props
		Object.keys(prevProps)
		.filter(isProperty)
		.filter(isGone(prevProps,nextProps))
		.forEach((name)=>{
			dom[name] = '';
		})
		//添加新的Prop 或者 改变的prop
		Object.keys(nextProps)
		.filter(isProperty)
		.filter(isNew(prevProps, nextProps))
		.forEach((name)=>{
			dom[name] = nextProps[name];
		})
	}
	```
	- 完善updateDom(), 对特殊props 事件的处理；
	```javascript
	function updateDom(dom, prevProps, nextProps) {
		//移除旧的或者改变的事件
		Object.keys(prevProps)
		.filter(isEvent)
		.filter(key=>
			!(key in nextProps) || isNew(prevProps,nextProps)[key] )
		.forEach(name=>{
			let eventType = name.toLowerCase().substring(2);
			dom.removeListener(eventType, prevProps[name]);
		})
		//删除旧的props
		Object.keys(prevProps)
		.filter(isProperty)
		.filter(isGone(prevProps,nextProps))
		.forEach((name)=>{
			dom[name] = '';
		})
		//添加新的Prop 或者 改变的prop
		Object.keys(nextProps)
		.filter(isProperty)
		.filter(isNew(prevProps, nextProps))
		.forEach((name)=>{
			dom[name] = nextProps[name];
		})
		//添加新的事件
		Object.keys(nextProps)
		.filter(isEvent)
		.filter(isNew(prevProps, nextProps))
		.forEach(name => {
			let eventType = name.toLowerCase().substring(2);
			dom.removeListener(eventType, prevProps[name]);
		})
	}
	```
---

## 函数组件 Function Components
1. 添加对函数组件的支持
	- 函数组件有两点不同
		1. 函数组件中没有DOM节点；
		2. children来自函数调用。
	```javascript
	function App(props) {
		return <h1>Hi {props.name}</h1>
	}
	const element = <App name="foo" />
	const container = document.getElementById("root")
	Didact.render(element, container)

	//jsx转化成js,
	function App(props){
		return CreateElement(
			"h1",
			null,
			"Hi",
			props.name
		)
	}
	const element = CreateElement(App, {
		name: 'foo',
	})	
	```
2. 增加处理函数组件的方法
	- 根据fiber type进行不同的处理, `updateHostComponent`方法跟之前一样，`updateFunctionComponent`来处理函数类型的组件；
	
	```javascript
	function performUnitOfWork(fiber){
		//主要做三步1.add dom node  2.create new fiber  3.return next unit of work
		//1.创建当前fiber的DOM添加到dom属性上。
		const isFunctionComponent = 
			fiber.type instanceof Function;
		if(isFunctionComponent){
			updateFunctionComponent(fiber);
		}else{
			updateHostComponent(fiber);
		}

		//3.找下一个fiber(工作单元)，深度优先遍历，先子后兄，再parent的兄弟
		if(fiber.child){
			return fiber.child;
		}
		let nextFiber = fiber;
		while(nextFiber){
			if(nextFiber.sibling){
				return nextFiber.sibling
			}
			nextFiber = newFiber.parent;
		}
	}

	function updateFunctionComponent(fiber) {
		// TODO
	}

	function updateHostComponent(fiber) {
		if (!fiber.dom) {
			fiber.dom = createDom(fiber)
		}
		reconcileChildren(fiber, fiber.props.children)
	}
	```
	- 继续实现`updateFunctionComponent`, 获取了children，`reconcileChildren`就按部就班的进行，不需要更改。
	```javascript
	function updateFunctionComponent(fiber) {
		//调用函数获取children
		const children = [fiber.type(fiber.props)];
		reconcileChildren(fiber, children);
	}
	```
	- 需要修改 `commitWork`函数，因为函数组件没有DOM节点，所以children需要挂到有DOM的patent上；
	```javascript
	function commitWork(fiber) {
		if (!fiber) {
			return
		}
		// s+
		let domParentFiber = fiber.parent;
		while(!domParentFiber.dom){
			domParentFiber = domParentFiber.parent;
		}
		// e+
		const domParent = fiber.parent.dom;
		if(fiber.effectTag === "PLACEMENT" && fiber.dom!==null){
			//fiber标记是PLACEMENT，则与之前相同，将DOM节点追加到父节点;
			domParent.appendChild(fiber.dom)
		}else if(fiber.effectTag === "UPDATE" && fiber.dom != null){
			//fiber标记是Update，使用fiber更新现有的节点;
			updateDOM(
				fiber.dom,
				fiber.alternate.props,
				fiber.props
			)
		}else if(fiber.effectTag === "DELETION"){
			//fiber标记是DELETION，则从父节点中移除当前的节点;
			// s-
			//domParent.removeChild(fiber.dom)
			//s+
			commitDeletion(fiber, domParent);
			//e+
		}    

		commitWork(fiber.child)
		commitWork(fiber.sibling)
	}
	//删除节点时，直到找到有DOM节点的子级
	function commitDeletion(fiber, domParent){
		if(fiber.dom){
			domParent.removeChild(fiber.dom);
		}else{
			commitDeletion(fiber.child, domParent);
		}
	}
	```
---

## HOOKS
1. 我们给函数组件添加状态，示例为计数器组件
	```javascript
	const MReact = {
		createElement,
		render,
		useState
	}
	function Counter(){
		const [state, setstate] = MReact.useState(1);
		return (
			<h1 onClick={()=>setState(c=>c+1)}>
				Count: { state }
			<h1>
		)
	}
	
	const element = <Counter />
	```
2. 在调用函数组件之前初始化一些变量，方便再 `useState`函数内部使用他们;
	```javascript
	let wipFiber = null;
	let hookIndex = null;
	function updateFunctionComponent(fiber){
		wipFiber = fiber;
		//向fiber添加一个数组，支持再函数组件中调用多次，记录当前的hooks的索引;
		hookIndex = 0;
		wipFiber.hooks = [];
		const children = [fiber.type(fiber.props)];
		reconcileChildren(fiber, children);
	}
	```
3. 每次函数组件被调用的时候，则useState则会被调用;
	- 在useState内部,首先要获取state的值,先去查看alternate是否有hook,如果有则使用这个值,如果没有使用初始值，定义新的hook,push到hooks,挂到fiber;钩子索引加1,返回状态; 
	```javascript
	function useState(initial){
		const oldHook = wipFiber.alternate?.hooks?.[hookIndex];
		const hook = {
			state: oldHook?oldHook.state:initial,
		}
		wipFiber.hooks.push(hook);
		return [hook.state];
	}
	```
	- `useState方法`应该还要返回一个更新状态的函数，一次定义一个`setState方法`，来接受更新动作，该动作将添加到钩子的queue中；
	```javascript
	function useState(initial){

		const oldHook = wipFiber.alternate?.hooks?.[hookIndex];
		const hook = {
			state: oldHook?oldHook.state:initial,
			queue: []
		}
		const actions = oldHook?oldHook.queue:[];
		//执行queue上的动作，获取最终的状态
		actions.forEach(action=>{
			hook.state = action(hook.state);
		})

		//setState
		const setState = (action)=>{
			hook.queue.push(action);
			//然后，我们执行与render函数中类似的操作，将当前的currentRoot设置为下一个nextUnitOfWork，以便工作循环可以开始新的渲染阶段。
			wipRoot = {
			  	dom: currentRoot.dom,
			  	props: currentRoot.props,
			  	alternate: currentRoot,
			}
			nextUnitOfWork = wipRoot;
			deletions = []
		}
		wipFiber.hooks.push(hook);
		hookIndex++;
		return [hook.state, setState];
	}
	```
	
> 参考链接
>> 1. https://react.docschina.org/docs/design-principles.html
>> 2. https://github.com/reactjs/zh-hans.reactjs.org/blob/master/content/docs/faq-internals.md
>> 3. https://github.com/acdlite/react-fiber-architecture
>> 4. https://pomb.us/
>> 5. https://react.jokcy.me/
	
	










