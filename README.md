# A Simple React
[Demo](https://oneonetwo.github.io/handwrite-a-simple-react/)
遵循React的Fiber架构，一步步实现自己的React,暂时不做优化，不实现复杂的功能；  
需要了解数据结构： 树，链表

我们需要做的事情: 
- Step 1: The createElement Function
- Step 2: The render Function
- Step 3: Concurrent Mode
- Step 4: Fibers
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

### 实现`createElement`方法

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
                    children: 'bar'
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
                    children: children.map(child)=>{
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
    
---

### render方法
1. 


