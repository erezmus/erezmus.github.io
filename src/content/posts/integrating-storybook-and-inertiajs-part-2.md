---
title: "Integrating Storybook and Inertia.js - Part 2"
published: 2026-01-30
draft: false
description: "A guide to previewing components that rely on inertia.js functionality. In Part 2, we look at mocking the router"
series: "Storybook and Inertia.js"
tags: ["typescript", "inertiajs", "storybook"]
---

In [Part one](/posts/integrating-storybook-and-inertiajs-part-1), we went over
how to inject Inertia page props for components that use them by invoking the
`usePage` composable.

Inertia's router is another piece of functionality that is often used, whether
in links or forms. When interacting with stories, we need to prevent an error
whenever we click a button or link.

:::note

You can find the code for this series
[here](https://stackblitz.com/edit/vitejs-vite-tvhs1blb).

:::

Our contact form component makes use of the router to submit the form

```vue title="src/components/ContactForm.vue"
<template>
  <div>
    <input v-model="text" />
    <button @click="onSubmit">Submit</button>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { router } from '@inertiajs/vue3';

const text = ref('');

const onSubmit = () => router.post('/submit', { msg: text.value });
</script>
```

A simple story would be:

```ts title="src/components/ContactForm.stories.ts"
import type { Meta, StoryObj } from "@storybook/vue3";
import ContactForm from "./ContactForm.vue";

const meta = { component: ContactForm } satisfies Meta<typeof ContactForm>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Main: Story = {};
```

If you click on the button, you would get a "Not found" error.

## Mocking the router

To get around this, simply use Storybook's `mocked` function in the `preview.ts`
file

```ts title=".storybook/preview.ts"
//...
import { sb } from "storybook/test";

sb.mock(import("../src/router.ts"));

//... rest of the file
```

:::tip

Here we are mocking a local re-export of Inertia's router, as trying to mock the
router directly gave an error when i tried it.

:::

```ts title="src/router.ts"
export { router } from "@inertiajs/vue3";
```

## Updated component and story

We change our contact form component to use the local router:

```vue title="src/components/ContactForm.vue"
<template>
  <div>
    <input v-model="text" />
    <button @click="onSubmit">Submit</button>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { router } from '../router.ts'

const text = ref('')

const onSubmit = () => router.post('/submit', { msg: text.value })
</script>
```

We then update our story to change the `router.post` behaviour:

```ts title="src/components/ContactForm.stories.ts"
import type { Meta, StoryObj } from "@storybook/vue3";
import { mocked } from "storybook/test";
import { router } from "../router.ts";
import ContactForm from "./ContactForm.vue";

const meta = {
  component: ContactForm,
  beforeEach: () => {
    mocked(router.post).mockImplementation((_, payload) => {
      console.log(payload);
    });
  },
} satisfies Meta<typeof ContactForm>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Main: Story = {};
```

You can now click on the Submit button without triggering an error. If you open
the output in a new tab and open the console, you can see the payload when you
press submit:

```js
{
    "msg": "hello"
}
```
