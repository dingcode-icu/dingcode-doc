## cocos creator v2+

SDK is the runtime to start explaining the execution of **dingcode** file.

### Install

> use dpm(The built-in package manager)

* Make sure **dpm**  in the current $install_path or global executable path

```shell
cd $install_path
dpm install dingcode-runtime@+
```



> use npm 

```shell
cd $install_path 
npm install dingcode-runtime
```



> static url 

* [click me to download]() 



## import to cocos creator

> step1  add dingcode-runtime files into project

![](https://dwb-public.oss-cn-beijing.aliyuncs.com/uPic20221011164514.png)



> step2 make sure global init  

We can use **Import As Plugin** to resolve it. 

Of course u can use other way to init the global plugin.



<img src="https://dwb-public.oss-cn-beijing.aliyuncs.com/uPic20221011165122.png"/>

> step3 import a *.ding.json file to project 

Just import the file into **any** path in your project

<img src="https://dwb-public.oss-cn-beijing.aliyuncs.com/uPic20221011181546.png"/>

> step4 execute the *.ding.json file 

Behavior tree should be a decision tree first and foremost
It is common practice to

* Find the right logic time to execute 
* Start from root node and iter execute to the end "action node"
* Find next right logic time to execute 

So game engine just necessary need to find the corresponding timing of execution and consistently let the behavior tree make decisions.

**But**

We use the behaviour tree in client here.

**So** here we can do like this:

* Find the right logic time to execute 
* Start from root node and iter execute 
* If there is node need holding?hold and wait...
* Continu execute the tree nodes until to the end "action node"
* Find next right logic time to execute 

<mark>Every node in behaviour tree can block the execution process.</mark>



Below is usage code for **cocos creator game engine** to <mark>Find the right logic time to execute </mark>

##### step 1 

create a cocos's cc.Component 

<img src="https://dwb-public.oss-cn-beijing.aliyuncs.com/uPic20221011184218.png"/>

##### step2 

create a "DingCode.Manager" to ctrl all dingcode 

```typescript
get mgr():DingCode.Manager<Node>{
    if(!this._mgr){
        this._mgr = new DingCode.Manager<Node>()
    }
    return this._mgr
}

```

##### step3 

add own node-logic code into the   "DingCode.Manager"

```typescript
const mgr = this.mgr
const dec_map = new Map<string, {new(data: DingCode.BTData):DingCode.BTNode}>();
//action
dec_map.set("set_game_running", Action_SetGameRunning)
dec_map.set("set_game_ending", Action_SetGameEnding)

// conditional
dec_map.set("is_out_screen", Decorator_IsOutScreen)

//root
dec_map.set("entry node", RootNode)
dec_map.set("self", SelfNode)
dec_map.set("entry", RootNode)
mgr.set_logic_decoder(dec_map);
```

##### step4 

use the  "DingCode.Manager" to execute the *.ding.json file to execute the tree's behaviour

```
this.mgr.load(this.ding_file.json as DingCode.IDingData,this.node);
this.mgr.run()
```

**BtNodeComp.ts** is a full example cocos's compoment script in [traffic]() demo.

BtNodeComp.ts

```typescript
import EventManager from "../../../scripts/common/manager/EventManager";
import Action_SetGameEnding from "./ai/action/set_game_ending";
import Action_SetGameRunning from "./ai/action/set_game_running";
import Decorator_IsOutScreen from "./ai/condition/is_out_screen";
import {RootNode } from "./ai/root/root";
import { SelfNode } from "./ai/root/self";
import { EVENT_LOGIC } from "./event";
// @ts-ignore
require("../libs/dingcode-runtime")

const {ccclass, property} = cc._decorator;

enum ERUN_POINT{
    onTap, 
    onStart,
    onEvent
}

export enum GAME_STATE{
    STARTING,
    RUNNING,
    ENDING,
}

@ccclass()
export class BtNodeComp extends cc.Component {

    /**
     * Property for *dingcode file*
     */
    @property(cc.JsonAsset)
    public ding_file:cc.JsonAsset = null 

    private _mgr:DingCode.Manager<Node> = null

    /**
     * 在action内用来可方便实例化prefab hash map
     */
    @property({
        type: [cc.String], 
        displayName: "可以被new的prefabe的hash表key"
    })
    public prefab_k:string[] = []

    @property({
        type:[cc.Prefab],
        displayName: "可以被new的prefabe的hash表value"
    })
    public prefab_v:cc.Prefab[] = []

    @property({
        type:[cc.Node],
        displayName: "可以被碰撞的节点"
    })
    public node_can_collision:cc.Node[] = []

    @property({
        type:[sp.Skeleton],
        displayName: "spine节点列表"
    })
    public list_spine:sp.Skeleton[] = []

    @property({
        type:[cc.Node],
        displayName: "可以操作的节点列表"
    })
    public node_list:cc.Node[] = []

    @property({
        type:[cc.Prefab],
        displayName: "可以操作的prefab列表"
    })
    public prefab_list:cc.Prefab[] = []

    
    /**
     * Execution timing 
     */
    @property({
        type: cc.Enum(ERUN_POINT), 
        displayName:"树逻辑执行时机"
    })
    run_point: ERUN_POINT = ERUN_POINT.onStart

    @property({
        type: cc.Enum(EVENT_LOGIC), 
        displayName:"触发事件名称", 
        visible() {
            return this.run_point == ERUN_POINT.onEvent;
        },
    }
    )
    run_point_event:EVENT_LOGIC=EVENT_LOGIC.EVENT_GAME_START

    start():void {   

        if(!this.ding_file){
            return 
        }
        const mgr = this.mgr
        const dec_map = new Map<string, {new(data: DingCode.BTData):DingCode.BTNode}>();
        //action
        dec_map.set("set_game_running", Action_SetGameRunning)
        dec_map.set("set_game_ending", Action_SetGameEnding)

        // conditional
        dec_map.set("is_out_screen", Decorator_IsOutScreen)

        //root
        dec_map.set("entry node", RootNode)
        dec_map.set("self", SelfNode)
        dec_map.set("entry", RootNode)
        mgr.set_logic_decoder(dec_map);
        mgr.load(this.ding_file.json as DingCode.IDingData,this.node);
        //-------run when start-------
        if (this.run_point == ERUN_POINT.onStart){
            mgr.run()
        }
        //-------run when ontap-------
        if (this.run_point == ERUN_POINT.onTap){
            let timeLastStart = 0
            this.node.on(cc.Node.EventType.TOUCH_START, (e:cc.Event.EventTouch)=>{
                let curTime = Date.now()
                if(curTime - timeLastStart < 1500){
                    return 
                }
                timeLastStart = curTime

                mgr.gBoard.set("EVENT_LAST_TOUCH", new cc.Vec2(e.getLocationX(), e.getLocationY()))
                mgr.run()
            })
        }
        // run when onevent
        if (this.run_point == ERUN_POINT.onEvent){
            EventManager.Instance.on("BTNODE_EVENT", this.onEventRunPoint, this)
        }
    
        //-------run when event-------
        
        // 添加黑板数据
        if(this.node_can_collision && this.node_can_collision.length > 0){
            this.setBlackData("node_can_collision_set", this.node_can_collision)
        }
        if(this.list_spine && this.list_spine.length > 0){
            this.setBlackData("list_spine_set", this.list_spine)
        }
        
    }   
    onEventRunPoint(data){
        if(data == this.run_point_event){
            this.mgr && this.mgr.run()
        }
    }
    

    /**
     * 给节点封装的通用方法
     */
    new_prefab(pk: string): cc.Node{
        const i = this.prefab_k.indexOf(pk);
        if (i<0)return;
        return cc.instantiate(this.prefab_v[i]);
    }

    public setBlackData(key, value){
        this.mgr.gBoard.set(key, value)
    }

    get mgr():DingCode.Manager<Node>{
        if(!this._mgr){
            this._mgr = new DingCode.Manager<Node>()
        }
        return this._mgr
    }

    protected onDestroy(): void {
        EventManager.Instance.off(this)
        delete this._mgr
        this._mgr = null
    }

    public set gMgr(data){
        (window as any)._mgr = data
    }
    get gMgr():DingCode.Manager<Node>{
        if(!(window as any)._mgr){
            (window as any)._mgr = {"GAME_STATE": GAME_STATE.STARTING}
        }
        return (window as any)._mgr
    }
    public setGBlackData(key, value){
        this.gMgr[key] = value
    }
    public getGBlackData(key){
        return this.gMgr[key]
    }
}
```