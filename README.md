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

