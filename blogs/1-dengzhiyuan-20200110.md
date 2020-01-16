
#   react + mobx:管理多个store的最佳实践


对于store的概念相信大家都不陌生，在前端领域store的运用必不可少，mobx的多个store运用在大型项目中尤其重要，关键在于如何去组织多个store，在大型项目可能store的运用耦合性太高，导致越到后期维护越难以维护，代码编写极其困难，用文字表达不太直观，直接上代码：

## 实践一：
---
## 现在有一个Home主界面，里面包含里一个列表List组件和一个Task任务组件，以下是组件代码
```tsx
let homeStore = new HomeStore()

class App extends Component {
    render() {
        return (
            <div>
                <Home homeStore={homeStore}/>
            </div>
        );
    }
}

```

```tsx
interface HomeProps {
    homeStore: HomeStore
}

@observer
class Home extends Component<HomeProps> {
    // 假设List组件需要调用homeStore某些方法，或者引用homeStore的某些属性，此时要把homeStore作为参数传进listStore，给listStore去调用homeStore的方法，去触发或者改变Home组件某些交互

    // 同时还有一个Task组件，糟糕的是产品设计Task组件不仅仅需要可以对Home组件做出交互同时也对List组件也有交互，此时taskStore需要传入homeStore的同时还要传入listStore      
    constructor(props: HomeProps) {
        super(props)
        this.listStore = new ListStore(this.props.homeStore)
        this.taskStore = new TaskStore(this.props.homeStore, this.listStore)
        // 这里还有一种情况就是有可能觉得listStore里面已经有homeStore了，taskStore就不需要传homeStore，
        // this.taskStore = new TaskStore(this.listStore)
    }
    listStore: ListStore
    taskStore: TaskStore

    render() {
        return (
            <Fragment>
                <List listStore={this.listStore} />
                <Task taskStore={this.taskStore} />
            </Fragment>
        );
    }
}

export default Home;

```
```tsx
interface TaskProps {
    taskStore: TaskStore
}

@observer
class Task extends Component<TaskProps> {
    constructor(props: TaskProps) {
        super(props)
    }    

    handleCount() {
        // 假设此处我需要调用homeStore的某些方法，对home组件做出一些交互，又或者需要取homeStore某些属性值的时候，可能会出现以下情况：
        this.props.taskStore.homeStore.xxx()
        // 若taskStore里面只有listStore，可能会出现以下情况：
        this.props.taskStore.listStore.homeStore.xxx()
    }

    render() {
        return (
            <div onClick={this.handleCount}></div>
        );
    }
}

export default Task;

```

## 对应的HomeStore，ListStore和TaskStore的代码如下：
```ts
export class HomeStore {    
    xxx() {
        //do something here
    }
}
```
```ts
export class ListStore {    
    constructor(homeStore: HomeStore) {
        this.homeStore = homeStore
    }

    homeStore: HomeStore 

    zzz() {

    }   
}
```
```ts
//TaskStore根据上面代码有以下两种情况
export class TaskStore {    
    constructor(homeStore: HomeStore, listStore: ListStore) {
        this.homeStore = homeStore
        this.listStore = listStore
    }

    homeStore: HomeStore    
    listStore: ListStore

    yyy() {
        // 若此时需要调用homeStore的方法，会以下这样调用：
        this.homeStore.xxx()
    }
}
```
```ts
export class TaskStore {    
    constructor(listStore: ListStore) {
        this.listStore = listStore
    }  

    listStore: ListStore

    yyy() {
        // 若此时需要调用homeStore的方法，会以下这样调用：
        this.listStore.homeStore.xxx()
    }    
}
```

以上代码是实战一方案，到此时，也许你会可能注意到一个很严重的问题Task组件调用homeStore的方法的时候需要在taskStore一层一层往上走，才能找到homeStore，这里只是两层Dom结构，如果Task组件里面还有子组件，子组件也需要一个单独管理的store的话，层级会越来越多，导致代码非常难维护。

---

针对越来越多的store,管理越来越复杂的情况，结合了mobx官方文档及建议，通过创建RootStore来管理即将要用到的stores，使用一个RootStore，保持对每个子 Store的引用。如有必要，可以在子Store中传入RootStore，让子Store也可以访问到RootStore以及RootStore引用的其他store，这个RootStore可以通过Provider的方式插入到组件数当中，下面我们上实践二的代码：

## 实践二：
---
```tsx
import { Provider } from "mobx-react";
import rootStore from '../xxx/xxx/xxx';

class App extends Component {
    //使用Provider组件将stores传递，注意了，Home组件的homeStore并没有通过props传入
    render() {
        return (
            <Provider {...rootStore}>
                <div>                  
                    <Home/>
                </div>
            </Provider>
        );
    }
}
```
```ts
export class RootStore {
    homeStore: HomeStore
    listStore: ListStore
    taskStore: TaskStore
    constructor() {      
        //this并非一定要传入子store的，当你有需要的时候才传入  
        this.homeStore = new HomeStore(this)
        this.listStore = new ListStore(this)
        this.taskStore = new TaskStore(this)
    }
}
export default new RootStore();
```
此时，可能会产生疑问Home页面以及Home里面怎么引用相应的store
```tsx
import { observer, inject } from "mobx-react";

interface HomeProps {
    //这里大家会以为为什么要写个问号，因为此处若不带问号，Home组件入口会报错，显示你并没有传入homeStore,所以这里要带问号
    homeStore?: HomeStore 
}

//组件通过inject装饰器注入了homeStore，组件里面就可以通过props访问homeStore了
@inject("homeStore")
@observer
class Home extends Component<HomeProps> {  
    constructor(props: HomeProps) {
        super(props)
    }

    doSomething() {
        this.props.homeStore.xxx()
    }

    render() {
        // 同理，List组件和Task组件并没有通过props传入相应的store
        return (
            <Fragment>
                <List />
                <Task />
                <div onClick={this.doSomething}></div>
            </Fragment>
        );
    }
}

export default Home;
```
通过这种方式组件确实可以访问注入进来的store了，但是这里有个小问题，或者说是一个可以优化的点，也是在各论坛比较多人问的问题，就是homeStore的类型是否一定要写到HomeProps里面并且带个问号来解决Home组件props没有传入的报错问题，个人认为并非最优办法，下面有一种办法，比较容易区分哪些对象是通过props传入哪些对象（store）是通过inject注入，以下是Home组件的更优写法：
```tsx
import { observer, inject } from "mobx-react";

interface HomeProps {

}

//此处的InjectedProps接口继承了HomeProps
interface InjectedProps extends HomeProps {
    homeStore: HomeStore;
}

@inject("homeStore")
@observer
class Home extends Component<HomeProps> {  
    constructor(props: HomeProps) {
        super(props)
    }
    
    get injected() {
        return this.props as InjectedProps;
    }    

    doSomething() {
        //此时可以通过get访问属性来读取inject注入的homeStore，这样的好处是可以明确区分props传入的对象和inject注入的对象的同时也解决了Home不传入homeStore的报错问题，若此时，想通过props来访问homeStore，显然是不行的，个人认为这种方式更优，若有更好的办法也可以推荐上来
        this.injected.homeStore.xxx()        
    }

    render() {
        return (
            <Fragment>
                <List />
                <Task />
                <div onClick={this.doSomething}></div>
            </Fragment>
        );
    }
}

export default Home;
```
至于后面的List组件和Task组件也是一样方式注入
```tsx
interface TaskProps {
    
}

//需要用到什么store，就注入什么store
interface InjectedProps extends HomeProps {
    homeStore: HomeStore;
    taskStore: TaskStore;
    listStore: ListStore;
}
@inject("homeStore", "taskStore", "listStore")
@observer
class Task extends Component<TaskProps> {
    constructor(props: TaskProps) {
        super(props)
    }    

    get injected() {
        return this.props as InjectedProps;
    }     

    handleCount() {
        //当然组件的逻辑可以在taskStore里面操作，并非一定要在组件上去操作
        this.injected.homeStore.xxx()
        this.injected.listStore.zzz()
    }

    render() {
        return (
            <div onClick={this.handleCount}></div>
        );
    }
}

export default Task;

```
通过inject方式更好地解决了props一层一层传递store的问题，避免了父级的store一定要通过子组件的props一级一级地往下传的问题，此时可能会产生一个疑惑taskStore现在要怎么读取其他store的东西呢，以下是taskStore的代码：

```ts
export class TaskStore {    
    constructor(rootStore: RootStore) {
        this.rootStore = rootStore
    }  

    rootStore: RootStore

    yyy() {
        //若此时要调用homeStore或者listStore的方法，可以通过rootStore去访问相应的方法就可以了，这样就避免了很多层级的问题
        this.rootStore.homeStore.xxx()
        this.rootStore.listStore.zzz()
    }    
}
```
---
通过实践二，可以发现通过inject的方式更加合理的管理stores，这样会减低了store与store之间的耦合性，代码更加简单易懂，但是此时读者可能会产生很多疑问，比方说，所有的store是不是一定要通过inject方式注入？Provider是不是全局只能有一个？多个Provider又怎么去配合使用

假设：我们Task组件并非一开始就加载或者写死的，是动态添加挂载的，而且每个Task要对应自己的一个store，并且Task组件下面可能还存在一些子组件，子组件里面可能会有一些**局部唯一**的store的情况

此时的taskStore就不能在rootStore里面new 一个实例然后通过inject方式注入，这样显然是不行的，通常**全局唯一**或者**局部唯一**的store才放到相对的根store里面，所以根据上述假设，Task组件的taskStore要通过props传递进入，然后我会建议在Task组件里再创建一个Provider来传递子组件的**局部唯一**store

**这里先定义一个相当于Task组件的局部根Store**
```ts
export class TaskRootStore {
    taskStore: TaskStore
    storeA: StoreA
    storeB: StoreB
    constructor() {      
        this.taskStore = new TaskStore(this)
        this.storeA = new StoreA(this)
        this.storeB = new StoreB(this)
    }
}
export default new TaskRootStore();
```

```tsx
//这里TaskProps定义了一个taskRootStore是因为每个Task都会对应单独的store，所以注册Task组件的时候，就要<Task taskRootStore={new taskRootStore()} />
interface TaskProps {
    taskRootStore: TaskRootStore;
}

interface InjectedProps extends HomeProps {
    homeStore: HomeStore;
    listStore: ListStore;
}
@inject("homeStore", "listStore")
@observer
class Task extends Component<TaskProps> {
    constructor(props: TaskProps) {
        super(props)
    }    

    get injected() {
        return this.props as InjectedProps;
    }     

    //Children组件就可以通过inject方式注入StoreA，StoreB，当然了也可以更上一级的homeStore
    render() {
        return (
            <Provider {...this.props.taskRootStore}>                
                <Children/>
            </Provider>
        );
    }
}

export default Task;

```
以上就是对于局部动态加载一个组件的实践，通过增加Provider去分发局部不变的store，如此改造，可以更加方便管理store维护store，以上就是对react + mobx管理多个store的实践见解
