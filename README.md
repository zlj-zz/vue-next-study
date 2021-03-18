以项目为案例了解 vue3 源码运行操作。vue3 的核心代码在 `vue-next/packages/` 下， 做了分包处理。

---

`shared` 存在了共享内容，不依赖其他文件。

```
├── shared
    ├── api-extractor.json
    ├── index.js
    └── src
        ├── codeframe.ts
        ├── domAttrConfig.ts
        ├── domTagConfig.ts
        ├── escapeHtml.ts
        ├── globalsWhitelist.ts
        ├── index.ts                # 导出整个shared， 还存放一些判断转换的方法
        ├── looseEqual.ts           # 宽松比较传入变量是否相等的方法(any)  宽松找到下标的方法
        ├── makeMap.ts              # 根据字符串创建 map 的方法
        ├── normalizeProp.ts        # 存放标准化 style、 class 的 方法
        ├── patchFlags.ts           # 存放标记 patch 的类型
        ├── shapeFlags.ts           # 存放标记节点的类型
        ├── slotFlags.ts            # 存放标记 slot 的类型
        └── toDisplayString.ts      # 将传入值转成 String
```

---

- main.js 中使用了 `createApp()`，这个入口在 `runtime-dom/src/index.ts` 中。

  ```js
  const app = createApp(APP);
  /**
   * APP = {
      __file: "/Users/zacharyzhang/Projects/vue-next-study/src/App.vue",
      __hmrId: "7ba5bd90",
      data(),
      render: ...
   }
  */
  ```

- `createApp` 中使用了， `ensureRenderer().createApp()`

  - `ensureRenderer()` 中返回了一个 `renderer`， 若不存在调用 `createRenderer(renderOptions)` 创建 `renderer`

    - 传入的 `renderOptions` 是 `nodeOptions` 和 `{patchProp, forcePatchProp}` 和并集。 `nodeOptions` 包含了所有 DOM 节点的操作，如创建、删除、插入、选取、克隆等操作。 `patchProp` 包含节点属性的操作，如 class、 style、 event 等。

    - `createRenderer` 是从 `runtime-core/src/renderer.ts` 中导出。调用了 `baseCreateRenderer(options)`

    - `baseCreateRenderer` 返回了 `render` 函数和 `createApp` 函数。同时里面实现 render 需要的所有操作， 如 patch、process、mount、unmount、update、remove、move 等操作。 `createApp` 是调用 `createAppAPI(render, hydrate)` 得到， 该方法来至 `runtime-core/src/apiCreateApp.ts`。

      - `createAppAPI` 返回 `createApp` 方法。 `createApp` 先创建空的的 context 和 installPlugins（插件 Set）。接着初始化 `context.app`， 将 `APP` 挂载到 `app._component`, `context` 挂载到 `app._context`。最后定义了 `use()`、`mixin()`、`component()`、`deractive()`、`mount()`、`unmount()`、`provide()` 方法。最后返回 app 对象

      ```js
      export function createAppAPI<HostElement>(
      render: RootRenderFunction,
      hydrate?: RootHydrateFunction
      ): CreateAppFunction<HostElement> {
          return function createApp(rootComponent, rootProps = null) {
              const context = createAppContext()
              const installedPlugins = new Set()

              let isMounted = false

              const app: App = (context.app = {
              _uid: uid++,
              _component: rootComponent as ConcreteComponent,
              _props: rootProps,
              _container: null,
              _context: context,

              version,

              get config() {
                  return context.config
              },

              set config(v) {
                  if (__DEV__) {
                  warn(
                      `app.config cannot be replaced. Modify individual options instead.`
                  )
                  }
              },

              use(plugin: Plugin, ...options: any[]) {/**...*/},
              mixin(mixin: ComponentOptions) {/**...*/},
              component(name: string, component?: Component): any {/**...*/},
              directive(name: string, directive?: Directive) {/**...*/},
              mount(rootContainer: HostElement, isHydrate?: boolean, isSVG?: boolean): any {/**...*/},
              unmount() {/**...*/},
              provide(key, value) {/**...*/}
              })
              return app
          }
      }
      ```

  - `ensureRenderer().createApp()` 中先取到 app 中的 moun() 进行重写，然后返回 app 对象。

- 调用 `mount('#app')`

  ```js
  // runtime-dom/src/index.ts

  const { mount } = app;
  // 第一次的使用 containerOrSelector 是 #app
  app.mount = (containerOrSelector: Element | ShadowRoot | string): any => {
    // 调用normalizeContainer获取根元素容器；
    const container = normalizeContainer(containerOrSelector);
    if (!container) return;
    const component = app._component; // 获取 component
    if (!isFunction(component) && !component.render && !component.template) {
      // 是否为函数组件，并且没有 render(), 没有 template
      component.template = container.innerHTML;
    }
    // clear content before mounting
    container.innerHTML = "";
    // 调用 app 原本的 mount()              - 判断是否为 svg
    const proxy = mount(container, false, container instanceof SVGElement);
    if (container instanceof Element) {
      container.removeAttribute("v-cloak");
      container.setAttribute("data-v-app", "");
    }
    return proxy;
  };
  ```

  - 拿到 `#app` 的 DOM 元素节点，调用原本的 `mount()`

    ```js
    // runtime-core/src/apiCreateApp.ts

    mount(
        rootContainer: HostElement,
        isHydrate?: boolean,
        isSVG?: boolean
    ): any {
        if (!isMounted) {
        const vnode = createVNode(rootComponent as ConcreteComponent, rootProps)// 创建 VNode

        // 在根VNode上存储应用程序上下文。这将在初始装载时在根实例上设置。
        vnode.appContext = context // app._context

        // HMR root reload
        if (__DEV__) {
            context.reload = () => {
            render(cloneVNode(vnode), rootContainer, isSVG)
            }
        }

        if (isHydrate && hydrate) {
            hydrate(vnode as VNode<Node, Element>, rootContainer as any)
        } else {
            render(vnode, rootContainer, isSVG) // 将 vnode 渲染到 DOM 上
        }
        isMounted = true
        app._container = rootContainer
        // for devtools and telemetry
        ;(rootContainer as any).__vue_app__ = app

        if (__DEV__ || __FEATURE_PROD_DEVTOOLS__) {
            devtoolsInitApp(app, version)
        }

        return vnode.component!.proxy
        } else if (__DEV__) {
        warn(
            `App has already been mounted.\n` +
            `If you want to remount the same app, move your app creation logic ` +
            `into a factory function and create fresh app instances for each ` +
            `mount - e.g. \`const createMyApp = () => createApp(App)\``
        )
        }
    },
    ```

    - 判断 `isMounted`， 初始化时 `isMounted = false`， 所以进入分支。 调用 `createVNode(rootComponent)` 创建 vnode， 这里的 rootComponent 就是上面的 `APP`

    ```js
    // runtime-core/src/vnode.ts

    function _createVNode(
      type: VNodeTypes | ClassComponent | typeof NULL_DYNAMIC_COMPONENT,
      props: (Data & VNodeProps) | null = null,
      children: unknown = null,
      patchFlag: number = 0,
      dynamicProps: string[] | null = null,
      isBlockNode = false
    ): VNode {
      if (!type || type === NULL_DYNAMIC_COMPONENT) {
        if (__DEV__ && !type) {
          warn(`Invalid vnode type when creating vnode: ${type}.`);
        }
        type = Comment;
      }

      if (isVNode(type)) {
        // createVNode receiving an existing vnode. This happens in cases like
        // <component :is="vnode"/>
        // #2078 make sure to merge refs during the clone instead of overwriting it
        const cloned = cloneVNode(type, props, true /* mergeRef: true */);
        if (children) {
          normalizeChildren(cloned, children);
        }
        return cloned;
      }

      // class component normalization.
      if (isClassComponent(type)) {
        type = type.__vccOpts;
      }

      // class & style normalization.
      if (props) {
        // for reactive or proxy objects, we need to clone it to enable mutation.
        if (isProxy(props) || InternalObjectKey in props) {
          props = extend({}, props);
        }
        let { class: klass, style } = props;
        if (klass && !isString(klass)) {
          props.class = normalizeClass(klass);
        }
        if (isObject(style)) {
          // reactive state objects need to be cloned since they are likely to be
          // mutated
          if (isProxy(style) && !isArray(style)) {
            style = extend({}, style);
          }
          props.style = normalizeStyle(style);
        }
      }

      // encode the vnode type information into a bitmap
      const shapeFlag = isString(type)
        ? ShapeFlags.ELEMENT
        : __FEATURE_SUSPENSE__ && isSuspense(type)
        ? ShapeFlags.SUSPENSE
        : isTeleport(type)
        ? ShapeFlags.TELEPORT
        : isObject(type)
        ? ShapeFlags.STATEFUL_COMPONENT
        : isFunction(type)
        ? ShapeFlags.FUNCTIONAL_COMPONENT
        : 0;

      if (
        __DEV__ &&
        shapeFlag & ShapeFlags.STATEFUL_COMPONENT &&
        isProxy(type)
      ) {
        type = toRaw(type);
        warn(
          `Vue received a Component which was made a reactive object. This can ` +
            `lead to unnecessary performance overhead, and should be avoided by ` +
            `marking the component with \`markRaw\` or using \`shallowRef\` ` +
            `instead of \`ref\`.`,
          `\nComponent that was made reactive: `,
          type
        );
      }

      // 初始化 vnode，几乎为空，
      // 除了 __v_isVNode, _v_skip, type, shapeFlag, patchFlag,
      const vnode: VNode = {
        __v_isVNode: true,
        [ReactiveFlags.SKIP]: true,
        type,
        props,
        key: props && normalizeKey(props),
        ref: props && normalizeRef(props),
        scopeId: currentScopeId,
        slotScopeIds: null,
        children: null,
        component: null,
        suspense: null,
        ssContent: null,
        ssFallback: null,
        dirs: null,
        transition: null,
        el: null,
        anchor: null,
        target: null,
        targetAnchor: null,
        staticCount: 0,
        shapeFlag,
        patchFlag,
        dynamicProps,
        dynamicChildren: null,
        appContext: null,
      };

      // validate key
      if (__DEV__ && vnode.key !== vnode.key) {
        warn(`VNode created with invalid key (NaN). VNode type:`, vnode.type);
      }

      normalizeChildren(vnode, children);

      // normalize suspense children
      if (__FEATURE_SUSPENSE__ && shapeFlag & ShapeFlags.SUSPENSE) {
        const { content, fallback } = normalizeSuspenseChildren(vnode);
        vnode.ssContent = content;
        vnode.ssFallback = fallback;
      }

      if (
        shouldTrack > 0 &&
        !isBlockNode &&
        currentBlock &&
        (patchFlag > 0 || shapeFlag & ShapeFlags.COMPONENT) &&
        patchFlag !== PatchFlags.HYDRATE_EVENTS
      ) {
        currentBlock.push(vnode);
      }

      return vnode;
    }
    ```

       - 前面的判断都不成立， 走到 `shapeFlag` 得到 `shapeFlag = 4`

       - 接着初始化 vnode

       - 初始化 children， 此时没有

       - 最后返回 vnode

    - 调用 `render(vnode, rootContainer, isSVG)` 将 vnode 渲染到 DOM 上
    ```js
    // runtime-core/src/renderer.ts

    const render: RootRenderFunction = (vnode, container, isSVG) => {
    // 若 vnode 不存在，container._vnode (已挂载的 vnode) 存在，就需要取消挂载
    if (vnode == null) {
        if (container._vnode) {
        unmount(container._vnode, null, null, true);
        }
    } else {
        // 若 vnode 存在， 打补丁
        patch(
        container._vnode || null,
        vnode,
        container,
        null,
        null,
        null,
        isSVG
        );
    }
    flushPostFlushCbs(); //
    container._vnode = vnode;
    };
    ```

       - 此时为第一次渲染， `vnode` 存在， `container._vnode`（old vnode） 不存在。 调用 `patch()`

       - `patch()` 中比较后，进入 processComponent(), 第一次渲染 n1 == null, 进入 mountComponent()

         - 先调用 `createComponentInstance()` 创建实例, 创建时会击中 beforeCreate()

         ```js
         // runtime-core/src/component.ts

         export function createComponentInstance(
         vnode: VNode,
         parent: ComponentInternalInstance | null,
         suspense: SuspenseBoundary | null
         ) {
         const type = vnode.type as ConcreteComponent
         // inherit parent app context - or - if root, adopt from root vnode
         const appContext =
             (parent ? parent.appContext : vnode.appContext) || emptyAppContext

         const instance: ComponentInternalInstance = {
             uid: uid++,
             vnode,
             type,
             parent,
             appContext,
             root: null!, // to be immediately set
             next: null,
             subTree: null!, // will be set synchronously right after creation
             update: null!, // will be set synchronously right after creation
             render: null,
             proxy: null,
             exposed: null,
             withProxy: null,
             effects: null,
             provides: parent ? parent.provides : Object.create(appContext.provides),
             accessCache: null!,
             renderCache: [],

             // local resovled assets
             //本地可回收资产
             components: null,
             directives: null,

             // resolved props and emits options
             //解析的道具和发射选项
             propsOptions: normalizePropsOptions(type, appContext),
             emitsOptions: normalizeEmitsOptions(type, appContext),

             // emit
             emit: null as any, // to be set immediately
             emitted: null,

             // state
             ctx: EMPTY_OBJ,
             data: EMPTY_OBJ,
             props: EMPTY_OBJ,
             attrs: EMPTY_OBJ,
             slots: EMPTY_OBJ,
             refs: EMPTY_OBJ,
             setupState: EMPTY_OBJ,
             setupContext: null,

             // suspense related
             suspense,
             suspenseId: suspense ? suspense.pendingId : 0,
             asyncDep: null,
             asyncResolved: false,

             // lifecycle hooks
             // not using enums here because it results in computed properties
             //生命周期挂钩
             //这里不使用枚举，因为它会导致计算属性
             isMounted: false,
             isUnmounted: false,
             isDeactivated: false,
             bc: null,   // beforeCreate
             c: null,    // created
             bm: null,   // beforeMounte
             m: null,    // mounted
             bu: null,   // beforeUpdate
             u: null,    // updated
             um: null,   // unmounted
             bum: null,  // beforeUnmounte
             da: null,   // deactived
             a: null,    // actived
             rtg: null,  // rendertrigered
             rtc: null,  // rendertracked
             ec: null    // errorCaptured
         }
         if (__DEV__) {
             instance.ctx = createRenderContext(instance)
         } else {
             instance.ctx = { _: instance }
         }
         instance.root = parent ? parent.root : instance
         instance.emit = emit.bind(null, instance)

         return instance
         }
         ```

         - 接着调用 `setupComponent()` 初始化 props 和 slots

         - 调用 `setupRenderEffect()`, 里面设置了 component 的 update() 方法， 其实是一个 effect。 这个 effect 用 `reactivity/src/effect/ts` 中的 `effect()` 生成。

         ```js
         export function effect<T = any>(
         fn: () => T,
         options: ReactiveEffectOptions = EMPTY_OBJ
         ): ReactiveEffect<T> {
             if (isEffect(fn)) {
                 fn = fn.raw
             }
             const effect = createReactiveEffect(fn, options)
             if (!options.lazy) {
                 effect()
             }
             return effect
         }
         function createReactiveEffect<T = any>(
         fn: () => T,
         options: ReactiveEffectOptions
         ): ReactiveEffect<T> {
             const effect = function reactiveEffect(): unknown {
                 if (!effect.active) {
                 return options.scheduler ? undefined : fn()
                 }
                 if (!effectStack.includes(effect)) {
                 cleanup(effect)
                 try {
                     enableTracking()
                     effectStack.push(effect)
                     activeEffect = effect
                     return fn()
                 } finally {
                     effectStack.pop()
                     resetTracking()
                     activeEffect = effectStack[effectStack.length - 1]
                 }
                 }
             } as ReactiveEffect
             effect.id = uid++
             effect.allowRecurse = !!options.allowRecurse
             effect._isEffect = true
             effect.active = true
             effect.raw = fn
             effect.deps = []
             effect.options = options
             return effect
         }
         ```
            
           - effect 接受两个参数，第一个是 effect 时需要的操作方法， 第二个是一些选项。

           - 这里面，先判断你是不是一个 effect，是就返回你的原型。不是就调用 `createReactivityEffect()` 创建。

           - 如果不是 lazy， 默认会执行一次。 执行时会调用传入的 `fn`
           
         - 这时就回到 `setupRenderEffect()` 中的 `componentEffect()`。在里面注意 调用 renderComponentRoot(instance)，拿到了 APP 的 vnode

         -  调用 `patch` 挂载 DOM， 此时 patch 中会走到 `processElement()`

           - `processElemt` 中 `n1 == null`， 走到 `mountElement`

             - `mountElement` 中 el 还不存在， 所有 先 调用 hostCreateElement 创建 并 挂载到 `vnode.el` 上

             - 接着 判断情况挂载 children， 这里目前只是 字符串， 所以调用 `hostSetElementText`

             - 接着 调用 `hostInsert` 将当前 DOM 插入到 父 DOM 中， 此时 parent 时 `#app`， 此时初次渲染完毕。


