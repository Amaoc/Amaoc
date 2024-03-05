---
title: Vue必须会的组件封装
categories:
  - 前端开发
tags:
  - js
  - vue
toc: true # 是否启用内容索引
---

<span style="color: red;">作为一个前端开发，必须会的vue组件封装技能</span>

## 一、v-modal组件封装

vue2 的写法：
```vue
<!-- 父组件 -->
<template>
  <my-input v-model="msg"></my-input>
</template>
<script>
export default {
  data() {
    return {
      msg: ''
    }
  }
}
</script>

<!-- 子组件 -->
<template>
  <el-input v-model="inputVal"></el-input>
</template>
<script>
export default {
  computed： {
    inputVal: {
      get() {
        return this.props.value
      },
      set(val) {
        this.$emit('input', val)
      }
    }
  }
}
</script>
```

vue3 的写法：
```vue
<!-- 父组件 -->
<template>
  <my-input v-model="msg"></my-input>
</template>
<script setup lang="ts">
import { ref } from 'vue'
const msg = ref('hello')
</script>

<!-- 子组件 -->
<template>
  <el-input v-model="inputVal"></el-input>
</template>
<script setup lang="ts">
import { computed } from 'vue';
const props = defineProps({
  modelValue: {
    type: String,
    default: '',
  }
});

const emit = defineEmits(['update:modelValue']);

const inputVal = computed(() => {
  get() {
    return props.modelValue
  },
  set(val) {
    emit('update:modelValue', val)
  }
})
</script>
```

## 二、如何二次扩展组件
在开发 Vue 项目中我们一般使用第三方 UI 组件库进行开发，如 `Element-Plus`, 但是这些组件库提供的组件并不一定满足我们的需求，这时我们可以通过对组件库的组件进行二次封装，来满足我们特殊的需求。  
对于封装组件有一个大原则就是我们应该尽量保持原有组件的接口，除了我们需要封装的功能外，我们不应该改变原有组件的接口，即保持原有组件提供的接口（属性、方法、事件、插槽）不变。

### 属性透传 `$attrs`

`$attrs` 的官方解释：包含了父作用域中不作为组件 props 或自定义事件的 attribute 绑定和事件；当一个组件没有声明任何 prop 时，这里会包含所有父作用域的绑定，并且可以通过 `v-bind="$attrs"` 传入内部的 UI 组件中

```vue
<template>
  <div class="my-input">
    <el-input v-bind="filteredAttrs"></el-input>
  
    <!-- 如果不希望过滤掉某些属性 可以直接使用 $attrs -->
    <el-input v-bind="$attrs"></el-input>
  </div>
</template>

<script lang="ts" setup>
import {useAttrs,computed,ref } from 'vue'
import { ElInput } from 'element-plus'
defineOptions({
  name: 'MyInput'
})

// 接收 name，其余属性都会被透传给 el-input
defineProps({
  name: String
});

// 如果我们不希望透传某些属性比如class, 我们可以通过useAttrs来实现
const attrs = useAttrs()
const filteredAttrs = computed(() => {
  return { ...attrs, class: undefined };
});
</script>
```

属性传递时，在默认情况下，父组件的属性会直接渲染在子组件的根节点上，但是有些情况我们希望是渲染在指定的节点上，可以使用 `$attrs` 和 `inheritAttrs: false` 就可以完美的解决这个问题。


### Event事件继承 `$listeners`
```vue
// Vue2
<template>
  <div class="my-input">
    <el-input v-bind="$attrs" v-on="$listeners"></el-input>
  </div>
</template>

// Vue3
<template>
  <div class="my-input">
    <!-- 在 Vue3 中，取消了$listeners这个组件实例的属性，将其事件的监听都整合到了$attrs上 -->
    
    <!-- 因此直接通过v-bind=$attrs属性就可以进行props属性和event事件的透传 -->
    <el-input v-bind="$attrs"></el-input>
    
  </div>
</template>

```

### 插槽传递 `$slots`

```vue
<template>
  <div class="my-input">
    <el-input
      v-model="childSelectedValue"
      v-bind="attrs"
      v-on="$listeners"
    >
     <!-- 遍历子组件非作用域插槽，并对父组件暴露 -->
     <template v-for="(index, name) in $slots" v-slot:[name]>
        <slot :name="name" />
      </template>
      <!-- 遍历子组件作用域插槽，并对父组件暴露 -->
      <!-- 作用域插槽是什么？父组件通过在子组件中使用插槽填充内容的时候，可以通过slot-scope属性获取 子组件中<slot>标签 通过属性 传递过来的子组件的作用域 的值。 -->
      <template v-for="(index, name) in $scopedSlots" v-slot:[name]="data">
        <slot :name="name" v-bind="data"></slot>
      </template>
    </el-input>
  </div>
</template>

<!-- 在 Vue3 中，取消了作用域插槽 $scopedSlots，将所有插槽都统一在 $slots 当中： -->
<template>
  <div class="my-input">
    <el-input
      v-model="childSelectedValue"
      v-bind="attrs"
      v-on="$listeners"
    >
      <template #[slotName]="slotProps" v-for="(slot, slotName) in $slots" >
          <slot :name="slotName" v-bind="slotProps"></slot>
      </template>
    </el-input>
  </div>
</template>
```

### 使用第三方组件的Methods
有些时候我们想要使用组件的一些方法，比如 `el-table` 提供9个方法，如何在父组件（也就是封装的组件）中使用这些方法呢？其实可以通过 ref 链式调用，比如 `this.$refs.tableRef.$refs.table.clearSort()`，但是这样太麻烦了，代码的可读性差；更好的解决方法：将所有的方法暴露出来，供父组件通过 `ref` 调用！

在 Vue2 中，可以将 `el-table` 提供方法提取到实例上：

```vue
<template>
  <div class="my-table">
    <el-table ref="el-table"></el-table>
  </div>
</template>

<script>
export default {
  mounted() {
    this.extendMethod()
  },
  methods: {
    extendMethod() {
      const refMethod = Object.entries(this.$refs['el-table'])
      for (const [key, value] of refMethod) {
        if (!(key.includes('$') || key.includes('_'))) {
          this[key] = value
      }
    }
  },
};
</script>
```

在 Vue3 中的使用方法如下：

```vue
<template>
  <div class="my-table">
    <el-table ref="table"></el-table>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted } from 'vue'
import { ElTable } from 'element-plus'

const table = ref();

onMounted(() => { 
    const entries = Object.entries(table.value); 
    for (const [method, fn] of entries) { 
        expose[method] = fn; 
    } 
}); 
defineExpose(expose);
</script>
```
