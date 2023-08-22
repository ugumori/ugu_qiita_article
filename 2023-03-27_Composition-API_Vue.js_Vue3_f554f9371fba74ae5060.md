<!--
title:   Vue3 CompositionAPIで子コンポーネントの関数を呼び出す
tags:    Composition-API,Vue.js,Vue3
id:      f554f9371fba74ae5060
private: false
-->
Vue3 CompositionAPIで親コンポーネントから子コンポーネントの関数を呼び出したい場合、子コンポーネントに `ref` を設定し、関数を呼び出す。

```Vue:ChildComponent.vue
<template>
  <div>
    <button @click="onButtonClick">Child Button</button>
  </div>
</template>

<script>
import { defineComponent } from 'vue';

export default defineComponent({
  name: 'ChildComponent',
  setup() {
    function onButtonClick() {
      console.log('Child button clicked');
    }

    return { onButtonClick };
  },
});
</script>
```
```Vue:ParentComponent.vue
<template>
  <div>
    <ChildComponent ref="childComponent" />
    <button @click="clickChildButton">Click Child Button</button>
  </div>
</template>

<script>
import { defineComponent, ref } from 'vue';
import ChildComponent from './ChildComponent.vue';

export default defineComponent({
  name: 'ParentComponent',
  components: {
    ChildComponent,
  },
  setup() {
    const childComponent = ref(null);

    function clickChildButton() {
      // 子コンポーネントのメソッドを呼び出す
      childComponent.value.onButtonClick();
    }

    return {
      childComponent,
      clickChildButton,
    };
  },
});
</script>
```

本題はここから。
CompositionAPIを使う場合、より簡潔に書くために `setup` シンタックスシュガーを使いたいところ。しかしこの場合、各プロパティがデフォルトで公開されないため、 `defineExpose` 関数を使って公開してあげる必要がある。
親も子も `setup` を使った場合の例はこちら。

```Vue:ChildComponent.vue
<template>
  <div>
    <button @click="onButtonClick">Child Button</button>
  </div>
</template>

<script setup lang="ts">
const onButtonClick = () => {
  console.log('Child button clicked');
}
defineExpose({ onButtonClick })
</script>
```

```Vue:ParentComponent.vue
<template>
  <div>
    <ChildComponent ref="childComponent" />
    <button @click="clickChildButton">Click Child Button</button>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import ChildComponent from './ChildComponent.vue';

const childComponent = ref(null);

const clickChildButton = () => {
  // 子コンポーネントのメソッドを呼び出す
  childComponent.value.onButtonClick();
}
</script>
```

ただし今更ではあるが、親から子の関数を直接呼び出す方法はVueのベストプラクティスに反するため、避けられる場合は避けるべき。データフローとイベントを適切に使って親子間のコミュニケーションを行うことが推奨される。