# A Simple React

[Demo](https://oneonetwo.github.io/handwrite-a-simple-react/)
遵循React的Fiber架构，一步步实现自己的React,暂时不做优化，不实现复杂的功能；  

我们需要做的事情: 
- Step 1: createElement Function
- Step 2: render Function
- Step 3: 任务调度
- Step 4: 构建Fibers
- Step 5: Render and Commit Phases
- Step 6: Reconciliation
- Step 7: Function Components
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

### createElement Function

1. 把JSX返回的js对象，创建成还有props、children的对象
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

### render Function

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

### [任务调度](https://github.com/acdlite/react-fiber-architecture#scheduling)
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

### [构建Fibers](https://github.com/acdlite/react-fiber-architecture#what-is-a-fiber)
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
	

