---
title: 'Integrating Storybook and Inertia.js - Part 1'
published: 2025-09-03
draft: false
description: 'A guide to previweing components that rely on inertia.js functionality'
series: 'Storybook and Inertia.js'
tags: ['javascript', 'inertiajs', 'storybook']
---

If you are using Inertia for your project with a front end framework like Vue or React, creating stories for them in Storybook can cause issues if they rely on Inertia's APIs like usePage or the router.

Fortunately there is a simple way to make them work together. In this post we'll be using Vue.js components but this technique should
apply to other frameworks as well.

You can find the code for this solution [here](https://stackblitz.com/edit/vitejs-vite-tvhs1blb).

## Injecting Inertia page props

One of the main issues is when components need the Inertia page props. Let's say we have the following component:

`src/components/HelloWorld.vue`

```vue
<template>
  <div>Hello World: {{ message }}</div>
</template>

<script setup lang="ts">
import { usePage, router } from '@inertiajs/vue3'
import { computed } from 'vue'

const page = usePage()
const message = computed(() => page.props.message)
</script>
```

If we were to use this component in a Story:

`src/components/HelloWorld.stories.ts`

```ts
import type { Meta, StoryObj } from '@storybook/vue3'
import HelloWorld from './HelloWorld.vue'

const meta = { component: HelloWorld } satisfies Meta<typeof HelloWorld>

export default meta
type Story = StoryObj<typeof meta>

export const Main: Story = {}
```

We would get the following error as `page.props` would be undefined.

```
Error: Cannot read properties of undefined (reading 'message')
```

To make this work, we need to find a way for Storybook to set the Inertia page props. If we take a look at the Inertia [source code](https://github.com/inertiajs/inertia/blob/master/packages/vue3/src/app.ts):

```ts
//...
const page = ref<Page<any>>(null)
// ...
const App: InertiaApp = defineComponent({
  //...
  setup({
    initialPage,
    initialComponent,
    resolveComponent,
    titleCallback,
    onHeadUpdate,
  }) {
    component.value = initialComponent ? markRaw(initialComponent) : null
    page.value = initialPage

    //...
    if (!isServer) {
      router.init({
        initialPage,
        resolveComponent,
        swapComponent: async (args: VuePageHandlerArgs) => {
          component.value = markRaw(args.component)
          page.value = args.page
          key.value = args.preserveState ? key.value : Date.now()
        },
      })
    }
    //...
  },
})

export function usePage<SharedProps extends PageProps>(): Page<SharedProps> {
  return reactive({
    props: computed(() => page.value?.props),
    //...
  })
}
```

We can see that the `page` variable is only set intially on setup or when the router swaps the component. The approach we will use is to
set the initial page props for each story.

## Story wrapper

We'll begin by creating a Storybook wrapper component:

`.storybook/InertiaWrapper.vue`

```vue
<template>
  <div id="inertia" />
  <slot v-if="isReady" name="story" />
</template>

<script setup lang="ts">
import { createApp, h, onMounted, ref } from 'vue'
import { createInertiaApp } from '@inertiajs/vue3'

export interface Props {
  inertia: Record<string, unknown>
}

const { inertia } = defineProps<Props>()

const isReady = ref(false)

onMounted(() =>
  createInertiaApp({
    id: 'inertia',
    page: inertia,
    resolve: () => ({ render: () => h('div') }),
    setup: ({ App, props, el }) => {
      createApp({ render: () => h(App, props) }).mount(el)
      isReady.value = true
    },
  }),
)
</script>
```

There are several points to note:

- The wrapper components accepts an object that will contain the inertia props needed for the story to work and those are passed into the `page` parameter of `createInertiaApp`.
- Inside the `setup` callback we create a dummy Vue app, which then renders and mounts Inertia's `App` component, which as shown in the previous section executes the line:
- `createInertiaApp` is called when after mounting and we set the `isReady` ref to true after we have finished the Inertia setup to prevent the story (and hence our HelloWorld component) from being rendered too early.
- We are also passing a dummy function for the `resolve` parameter as we won't be using the router.

## Adding a decorator

`.storybook/preview.ts`

```ts
import { h } from 'vue'
import InertiaWrapper from './InertiaWrapper.vue'

const DEFAULT_INERTIA_PROPS = {
  storage: {
    url: 'http://example.com',
  },
}

export const withInertiaWrapper = (storyFn, context) => {
  const inertia = {
    props: { ...DEFAULT_INERTIA_PROPS, ...(context.globals.inertia || {}) },
    url: '',
  }
  const story = storyFn()

  return () => {
    return h(
      InertiaWrapper,
      { inertia },
      {
        story: () => h(story, { ...context.args }),
      },
    )
  }
}

export const decorators = [withInertiaWrapper]
```

The `DEFAULT_INERTIA_PROPS` contains inertia props that all components can use and is then merged by the `inertia` property inside `context.globals`. The `InertiaStoryWrapper` then wraps the story with the merged props passed in.

## Updating the story

We can now update our story for our HelloWorld component

`src/components/HelloWorld.stories.ts`

```ts
import type { Meta, StoryObj } from '@storybook/vue3'
import HelloWorld from './HelloWorld.vue'

const meta = {
  component: HelloWorld,
  globals: {
    inertia: {
      message: 'This is Inertia.js in Storybook',
    },
  },
} satisfies Meta<typeof HelloWorld>

export default meta
type Story = StoryObj<typeof meta>

export const Main: Story = {}
```

Setting the `inertia` property in the globals context allows our decorator to pass it on to the wrapper and onto Inertia.js.

You can now see the message displayed in the Story.
