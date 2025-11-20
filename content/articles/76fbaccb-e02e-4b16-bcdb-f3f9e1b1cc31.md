+++
date = '2022-07-15'
title = "Using v-model on a prop"
tags = ['Vue.js', 'Tutorials']
+++

With Vue.js, passing data to a child component is generally achieved using props. However, mutating props from a child component is generally considered bad practice. Instead, child components must invoque the prop update from their parent using $emit.

For this purpose, computed values can be leveraged as they provide ways to define custom getters and setters. In this case, the get() method can simply return the value passed as prop by the parent while set() emits the desired new value.

```
<template>
  <div>
    <input v-model="myComputedProp" />
  </div>
</template>

<script>
export default {
  name: 'ChildComponent',
  props: {
    value: String
  },
  computed: {
    myComputedProp: {
      get() { return this.value },
      set(newVal) { this.$emit('input', newVal)}
    }
  }
}
</script>
```

In this example, the prop is named value and its value updates are emitted using the input event. Consequently the parent component can directly achieve bi-directional syncing using v-model.

```
<template>
  <div>
    <ChildComponent v-model="myVar" />
  </div>
</template>

<script>
import ChildComponent from './components/ChildComponent.vue'

export default {
  name: 'ParentComponent',
  components: {
    ChildComponent
  },
  data: () => ({
    myVar: 'Hello'
  })
}
</script>
```