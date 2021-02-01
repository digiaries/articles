## 浅谈React16框架 - Fiber

---

#### **前言**

React实现可以粗划为两部分：**reconciliation**（diff阶段）和 **commit**(操作DOM阶段)。在 v16 之前，reconciliation 简单说就是一个自顶向下递归算法，产出需要对当前DOM进行更新或替换的操作列表，一旦开始，会持续占用主线程，中断操作却不容易实现。当JS长时间执行（如大量计算等），会阻塞样式计算、绘制等工作，出现页面脱帧现象。所以，v16 进行了一次重写，迎来了代号为Fiber的异步渲染架构。

---

### **Fiber**

> **Fiber核心是实现了一个基于优先级和requestIdleCallback的循环任务调度算法**。它包含以下特性：([参考：fiber-reconciler][1])
- reconciliation阶段可以把任务拆分成多个小任务
- reconciliation阶段可随时中止或恢复任务
- 可以根据优先级不同来选择优先执行任务

从其特性可看出，Fiber核心是更换了reconciliation阶段的运作。那么，问题来了：

- ##### **为什么对reconciliation阶段进行拆分，commit阶段呢？**

    reconciliation阶段包含的主要工作是对current tree 和 new tree 做diff计算，找出变化部分。进行遍历、对比等是可以中断，歇一会儿接着再来。
    commit阶段是对上一阶段获取到的变化部分应用到真实的DOM树中，是一系列的DOM操作。不仅要维护更复杂的DOM状态，而且中断后再继续，会对用户体验造成影响。在普遍的应用场景下，此阶段的耗时比diff计算等耗时相对短。
    所以，Fiber选择在reconciliation阶段拆分


- ##### **如何拆分呢?**

 首先，我们可以通过 [A Cartoon Intro to Fiber][2]中的一张图来看：

    ![image](images/1-eim.png)  


    > 1. 用户调用ReactDOM.render传入组件，React创建Element树;
    > 2. 在第一次渲染时，创建vdom树，用来维护组件状态和dom节点的信息（如List/Button/Item等）。当后续操作如render或setState时需要更新，通过diff算出变化的部分；
    > 3. 根据变化的部分更新vdom树、调用组件生命周期函数等，同步应用到真实的DOM节点中。


  在第二阶段，Fiber是把render/update分片，拆解成多个小任务来执行，每次只检查树上部分节点，做完此部分后，若当前一帧（16ms）内还有足够的时间就继续做下一个小任务，时间不够就停止操作，等主线程空闲时再恢复。
  这种停止/恢复操作，需要记录上下文信息。而当前只记录单一dom节点的vDom tree 是无法完成的，
  **Fiber引入了fiber tree，是用来记录上下文的vDom tree**，可以理解为升级版的钢铁侠。


 fiber tree上一个节点的结构大致有：

  ```
  export type Fiber = {
      tag: TypeOfWork, // 类型
      type: 'div',
      return: Fiber|null, // 父节点
      child: Fiber|null, // 子节点
      sibling: Fiber|null, // 兄弟节点
      alternate: Fiber|null, //diff出的变化记录在这个fiber上
      .....
  };
  ```
完整树结构可参见 [ReactFiber][4]

 所以，Fiber是根据一个fiber节点（VDOM节点）来拆分，以fiber node为一个任务单元，一 个组件实例都是一个任务单元。任务循环中，每处理完一个fiber node，可以中断/挂起/恢复。



- ##### **基于requestIdleCallback和优先级任务调度**
    1. requestIdleCallback
      ```
      window.requestIdleCallback(callback[, options])

      ```
     浏览器提供的requestIdleCallback API中的Cooperative Scheduling可以让浏览器在空闲时间执行回调（开发者传入的方法），在回调参数中可以获取到当前帧（16ms）剩余的时间。利用这个信息可以合理的安排当前帧需要做的事情，如果时间足够，那继续做下一个任务，如果时间不够就歇一歇，调用requestIdleCallback来获知主线程不忙的时候，再继续做任务。
    2. **不同的任务分配不同的优先级**，Fiber根据任务的优先级来动态调整任务调度，先做高优先级的任务
     ```
       {  
          NoWork: 0, // No work is pending.
          SynchronousPriority: 1, // 文本输入框
          TaskPriority: 2, // 当前调度正执行的任务
          AnimationPriority: 3, // 动画过渡
          HighPriority: 4, // 用户交互反馈
          LowPriority: 5, // 数据的更新
          OffscreenPriority: 6, // 预估未来需要显示的任务
      }
     ```

    3. 任务调度的过程是：

        > 1. 在任务队列中选出高优先级的fiber node执行，调用requestIdleCallback获取所剩时间，若执行时间超过了deathLine，或者突然插入更高优先级的任务，则执行中断，保存当前结果，修改tag标记一下，设置为pending状态，迅速收尾并再调用一个requestIdleCallback，等主线程释放出来再继续
        > 2. 恢复任务执行时，检查tag是被中断的任务，会接着继续做任务或者重做
        
        一个任务单元执行结束或挂起，会调用基于requestIdleCallback的调度器，返回一个新的任务队列继续进行上述过程。

- ##### **如何任务循环**

    上面我们有介绍fiber node的属性存放上下文信息的指向。

    ![image](images/2-fiber-attr.png)

    
    - child：指向当前节点的子节点。当此节点任务结束后，如果child存在，就开始子节点的任务
    - sibling: 当子树已遍历到底部，回到父节点时，会拿到父节点的兄弟节点进行任务
    - return: 当当前节点没有子节点时，会向上回到父节点
    - **alternate**
    在调用render或setState后，会克隆出一个镜像fiber，diff产生出的变化会标记在镜像fiber上。而alternate就是链接当前fiber tree和镜像fiber tree, 用于断点恢复。
    - **workInProgress tree**
    React中对其描述如下：

     > **work-in-progress**
    A fiber that has not yet completed; conceptually, a stack frame which has not yet returned.**The alternate of the current fiber is the work-in-progress, and the alternate of the work-in-progress is the current fiber.**
    
       当前fiber节点的alternate属性指向workInProgress节点，对应workInProgress节点的alternate属性指向当前fiber节点。
    上面alternate中说到镜像fiber tree就是workInProgress tree。
    workInProgress tree上每个节点都有一个effect list，用来存放需要更新的内容。此节点更新完毕会向子节点或邻近节点合并 effect list。


    **任务循环的过程如下：**


    >1. 找到高优先级的待处理的节点
    >2. 检查当前节点是否需要更新，不需要的话，直接clone子节点，直接到5
    >3. 打个tag标记，更新自己（组件更新props，context等，DOM节点记下DOM change）
    >4. 通过render获取子节点，生成子节点的workInProgress节点
    >5. 若没有产生子节点，归并diff出的不同部分effect list（包含DOM change）到父节点
    >6. 把子节点或兄弟节点作为待处理任务，准备进入下一个任务循环。若已经回到了workInProgress tree的根节点，则任务循环结束

    通过每个节点更新结束时向上归并effect list来收集任务结果，reconciliation结束后，根节点的effect list里记录了包括DOM change在内的所有side effect

    通过一段代码，我们来看看如何进行任务循环：

    ```javascript
    export class Home extend React.component<HomeProps, any> {
        componentWillReceiveProps(nextProps: HomeProps) {}
        componentDidMount() {}
        componentDidUpdate() {}
        componentWillUnmount() {}
        .....
        render() {
            return (
                <div className="top">
                    <span>ZZ</span>
                    <button>click</button>
                </div>
            )
        }
    }
    ReactDom.render(<Home />, document.querySelector(selectors: '#hostRoot'))
    ```

    以它为例创建fiber tree：
    
    ![image](images/3-fiber-tree.png)
    
    >1. 从根节点 #hostRoot出发，走到它的child节点 div.top
    >2. div.top有两个子节点span和button ，先从左子树span开始
    >3. span只有一个子节点 ZZ，到达树的底部后原路返回
    >4. 返回到span时发现其有一个兄弟节点 button, 走去button
    >5. button有一个子节点 click, 同样到达树的底部后原路返回到button
    >6. button的右边没有兄弟了（若有，则继续重复4-5），返回到父节点 div.top
    >7. 最后回到根节点 #hostRoot


 再次setState或render时，构建workInProgress tree:
    
      ![image](images/4-workInProcess-tree.png)
    
    
    >1. 先把根节点 #hostRoot克隆出来，并用child属性指向它在fiber tree中的子节点div.top
    >2. 把div.top从fiber tree中克隆出来，需要更新的话，加一个tag标志
    >3. 更新div.top节点的状态，属性的指向等，并通过render获取到它的新子节点
    >4. 尽量复用旧子节点span或button来创建新子节点的workInProgress结构
    >5. 把新节点button做为下一个任务，调用requestIdleCallback获取当前帧所剩余的时间，如果还足够，就继续button的任务，重复2-4，否则，就等主线程空闲后再开始循环
    >6. 当子节点submit中没有child属性的指向，表示已到达底部。会把diff时产生的effect list会merge回return属性指向的父节点button上
    >7. 当父节点button没有兄弟节点时，会一直向上return回根节点，并把每个节点上产生的effect-list合并到根节点上。任务循环结束


---------------------
**我们再来看下源码中的reconcilation阶段：**

- **workLoop阶段**

 首先获取到下一个任务的执行优先级，调用一个performUnitOfWork函数进入reconcilation阶段的工作, 分为beginWork和completeWork

 1. ##### **beginWork**

  ![image](5-begin-work.png)

    beginWork中只是处理节点的不同tag属性对应不同的更新方法。如 updateClassComponent方法对应的是tag类型是React组件实例。

  ![image](6-update-comp.png)

 reconcileChildren 实现对新老节点的diff。其内部主要是调用 reconcileChildFibers 方法。

  ![image](7-reco-child-fibers.png)

  React中diff是基于同层级节点的，节点的移动/插入/删除等操作都是在fiber tree的同一层级中进行。

  ![image](8-rec-array.png)

  2. ##### **completeWork**
  
  completeWork的工作主要是通过新老节点的prop或tag等，收集节点的effect-list。然后向上一层一层循环，merge每个节点的effect-list，当到达根节点#hostRoot时，节点上包含所有的effect-list。并把effect-list传给pendingcommit，进入commit阶段。

  ![image](9-all-flow.png)




----------
- **Fiber对生命周期的影响**
```
  // reconciliation阶段
  componentWillMount
  componentWillReceiveProps
  shouldComponentUpdate
  componentWillUpdate
  --------------------------------
  // commit阶段
  componentDidMount
  componentDidUpdate
  componentWillUnmount
```

在我们之前的文章中也介绍过，在reconciliation阶段，生命周期函数会被多次调用，开发者慎重调用这个阶段的生命周期。
Reactv16.3已经开始了废弃计划：
>- 16.3版本：引入带UNSAFE_前缀的3个生命周期函数UNSAFE_componentWillMount，UNSAFE_componentWillReceiveProps和UNSAFE_componentWillUpdate，这个阶段新旧6个函数都能用。新引入生命周期函数getDerivedStateFromProps和getSnapshotBeforeUpdate，用来代替componentWillReceiveProps和componentWillUpdate
>- 16.3+版本：警告componentWillMount，componentWillReceiveProps和componentWillUpdate即将过时，这个阶段新旧6个函数也都能用，只是旧的在DEV环境会报Warning
>- 17.0版本：正式废弃componentWillMount，componentWillReceiveProps和componentWillUpdate，这个阶段只有新的带UNSAFE_前缀的3个函数能用，旧的不会再触发


本文只是分析了reconciliation阶段，不涉及commit阶段，感兴趣的同学可以研究下。




  [1]: https://reactjs.org/docs/codebase-overview.html#fiber-reconciler
  [2]: https://www.youtube.com/watch?v=ZCuYPiUIONs
  [4]: https://github.com/facebook/react/blob/v16.2.0/packages/react-reconciler/src/ReactFiber.js#L91



