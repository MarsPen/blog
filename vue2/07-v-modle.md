
<h1 align="center">📖 v-model 原理分析</h1>

在日常开发中我们使用 v-model 来实现数据的双向绑定，来看一下下面的例子

```js
 new Vue({
  el: '#app',
  template: '<div>'
  + '<input v-model="message" placeholder="请输 message">' +
  '<span> message is: {{ message }}</span>' +
  '</div>',
  data() {
    return {
      message: ''
    }
  }
})
```

上面的例子其实非常简单，当我们在输入框中输入内容的时候，span 元素中也会同步变化。那它是怎么做到的呢，让我们来看一下源码。

首先是 parse 阶段调用 genDirectives 方法，该方法定义在 `src/compiler/codegen/index.js` 中

```js
function genDirectives (el: ASTElement, state: CodegenState): string | void {
  const dirs = el.directives
  if (!dirs) return
  let res = 'directives:['
  let hasRuntime = false
  let i, l, dir, needRuntime
  for (i = 0, l = dirs.length; i < l; i++) {
    dir = dirs[i]
    needRuntime = true
    const gen: DirectiveFunction = state.directives[dir.name]
    if (gen) {
      // compile-time directive that manipulates AST.
      // returns true if it also needs a runtime counterpart.
      needRuntime = !!gen(el, dir, state.warn)
    }
    if (needRuntime) {
      hasRuntime = true
      res += `{name:"${dir.name}",rawName:"${dir.rawName}"${
        dir.value ? `,value:(${dir.value}),expression:${JSON.stringify(dir.value)}` : ''
      }${
        dir.arg ? `,arg:"${dir.arg}"` : ''
      }${
        dir.modifiers ? `,modifiers:${JSON.stringify(dir.modifiers)}` : ''
      }},`
    }
  }
  if (hasRuntime) {
    return res.slice(0, -1) + ']'
  }
}
```

从源码中可以看到 genDirectives 方法就是遍历 el.directives，然后调用 `const gen: DirectiveFunction = state.directives[dir.name]` 获取每个指令对应的方法。而 state 则是实例化 CodegenState 的时候通过 option 传入的。这个 option 就是编译相关的配置，它在不同的平台下配置不同，在 web 环境下的定义在 `src/platforms/web/compiler/options.js 下`

```js
export const baseOptions: CompilerOptions = {
  expectHTML: true,
  modules,
  directives,
  isPreTag,
  isUnaryTag,
  mustUseProp,
  canBeLeftOpenTag,
  isReservedTag,
  getTagNamespace,
  staticKeys: genStaticKeys(modules)
}
```

directives 定义在 `src/platforms/web/compiler/directives/index.js` 中

```js
export default {
  model,
  text,
  html
}
```

那么对于 v-model 而言，对应的 directive 函数是在 `src/platforms/web/compiler/directives/model.js` 中定义的 model 函数


```js
export default function model (
  el: ASTElement,
  dir: ASTDirective,
  _warn: Function
): ?boolean {
  warn = _warn
  const value = dir.value
  const modifiers = dir.modifiers
  const tag = el.tag
  const type = el.attrsMap.type
  // ...

  if (el.component) {
    genComponentModel(el, value, modifiers)
    // component v-model doesn't need extra runtime
    return false
  } else if (tag === 'select') {
    genSelect(el, value, modifiers)
  } else if (tag === 'input' && type === 'checkbox') {
    genCheckboxModel(el, value, modifiers)
  } else if (tag === 'input' && type === 'radio') {
    genRadioModel(el, value, modifiers)
  } else if (tag === 'input' || tag === 'textarea') {
    genDefaultModel(el, value, modifiers)
  } else if (!config.isReservedTag(tag)) {
    genComponentModel(el, value, modifiers)
    // component v-model doesn't need extra runtime
    return false
  } else if (process.env.NODE_ENV !== 'production') {
    // ...
  }

  // ensure runtime directive metadata
  return true
}

```

从上面可以看到 model 函数会根据 AST 元素节点的不同情况去执行不同的逻辑。我们来看下 genDefaultModel 函数

```js
function genDefaultModel (
  el: ASTElement,
  value: string,
  modifiers: ?ASTModifiers
): ?boolean {
  const type = el.attrsMap.type

  // ....

  const { lazy, number, trim } = modifiers || {}
  const needCompositionGuard = !lazy && type !== 'range'
  const event = lazy
    ? 'change'
    : type === 'range'
      ? RANGE_TOKEN
      : 'input'

  let valueExpression = '$event.target.value'
  if (trim) {
    valueExpression = `$event.target.value.trim()`
  }
  if (number) {
    valueExpression = `_n(${valueExpression})`
  }

  let code = genAssignmentCode(value, valueExpression)
  if (needCompositionGuard) {
    code = `if($event.target.composing)return;${code}`
  }

  addProp(el, 'value', `(${value})`)
  addHandler(el, event, code, null, true)
  if (trim || number) {
    addHandler(el, 'blur', '$forceUpdate()')
  }
}

```

genDefaultModel 函数主要是通过传入的 modifiers 去处理 event 和 valueExpression 的值， 然后调用 genAssignmentCode。它的定义在 `src/compiler/directives/model.js` 中 

```js
export function genAssignmentCode (
  value: string,
  assignment: string
): string {
  const res = parseModel(value)
  if (res.key === null) {
    return `${value}=${assignment}`
  } else {
    return `$set(${res.exp}, ${res.key}, ${assignment})`
  }
}
```

genAssignmentCode 主要是通过 parseModel 方法对传入的 value 进行解析生成 code。生成 code 之后 genDefaultModel 方法执行了两个非常关键的方法

```js
addProp(el, 'value', `(${value})`)
addHandler(el, event, code, null, true)
```

通过修改 AST 元素，给 el 添加一个 prop，相当于我们在 input 上动态绑定了 value，又给 el 添加了事件处理，相当于在 input 上绑定了 input 事件，就相当于动态的绑定了值和事件，转换模版如下

```js
<input
  v-bind:value="name"
  v-on:input="name=$event.target.value">
```

当获取完指令的方法后，我们在回到 `genDirectives` 方法。如果需要返回运行时对象则返回 needRuntime 为 ture ，开成生成根据 dir 生成 res。

```js
  if (needRuntime) {
    hasRuntime = true
    res += `{name:"${dir.name}",rawName:"${dir.rawName}"${
      dir.value ? `,value:(${dir.value}),expression:${JSON.stringify(dir.value)}` : ''
    }${
      dir.arg ? `,arg:"${dir.arg}"` : ''
    }${
      dir.modifiers ? `,modifiers:${JSON.stringify(dir.modifiers)}` : ''
    }},`
  }
```

最后生成 with 可执行代码，对于我们的例子生成如下 code。

```js
with(this) {
  return _c('div',[_c('input',{
    directives:[{
      name:"model",
      rawName:"v-model",
      value:(message),
      expression:"message"
    }],
    attrs:{"placeholder":"请输入 message "},
    domProps:{"value":(message)},
    on:{"input":function($event){
      if($event.target.composing)
        return;
      message=$event.target.value
    }}}),_c('span',[_v("message is: "+_s(message))])
    ])
}

```

看完了 v-model 的语法糖，我们再来看一下流程中的关键函数

```js
// 将优化过后的 ast 树转化成执行的代码
generate(
  ast: ASTElement | void,
  options: CompilerOptions
) {
  const code = ast ? genElement(ast) : '_h("div")'
  // ...
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: currentStaticRenderFns
  }
}
// 生成 code
genElement(el: ASTElement) {
  if (el.component) {
    code = genComponent(el.component, el)
  } else {
    const data = el.plain ? undefined : genData(el)

    const children = el.inlineTemplate ? null : genChildren(el)
    code = `_h('${el.tag}'${
      data ? `,${data}` : '' // data
    }${
      children ? `,${children}` : '' // children
    })`
  }
  // module transforms
  for (let i = 0; i < transforms.length; i++) {
    code = transforms[i](el, code)
  }
  return code
}
// 生成 data
genData(el: ASTElement) {
  const dirs = genDirectives(el)
  if (dirs) data += dirs + ','
  // ...
  return data
}
// 生成 dir
genDirectives (el: ASTElement){}
```

最后用 with 生成的 code 传入 render 中执行 `new Function()`
```js
  const compiled = compile(template, options) 
  res.render = createFunction(compiled.render, fnGenErrors) 
  function createFunction (code, errors) { 
    try { return new Function(code) 
    } catch (err) { 
      errors.push({ err, code })
      return noop 
  } }
```

### 总结
通过上面对源代码的分析了解到它是 Vue 双向绑定的真正实现，本质上就是一种语法糖，它即支持原生表单元素，也支持自定义组件，本章中只分析了表单元素，对于自定义组件后面也可以慢慢了解。

<img src="/assets/vue-vmodel.png"></img>