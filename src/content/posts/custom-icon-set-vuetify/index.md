---
title: "Custom icon set in Vuetify"
published: 2026-01-30
draft: false
description: "How to add a custom icon set in Vuetify"
series: "Vuetify Tips"
tags: ["typescript", "Vue.js", "Vuetify"]
---

## Introduction

While Vuetify has very good documentations, there's a few areas that require a
bit of research and tinkering. One that I came across was custom icon sets.
Vuetify supports several icon font libraries like Material Design Icons,
Material Icons and Font Awesome.

If you want a different set, you need to set up vuetify to load the icons. There
is some documentation (here)
[https://vuetifyjs.com/en/features/icon-fonts/#creating-a-custom-icon-set] but
its not 100% clear what you have to do.

:::note

You can find the code for this series
[here](https://stackblitz.com/edit/vitejs-vite-ftvgj6cm).

:::

## The icon set

We'll use the Fluent icons library as an example. The first step is to install
it:

```sh
npm i @vicons/fluent @vicons/utils
```

Next, where we need to update how we initialise our vuetify plugin. The usual
setup with Vuetify is:

```ts title="src/main.ts"
import * as components from "vuetify/components";
import * as directives from "vuetify/directives";
import "vuetify/styles";

export const vuetify = createVuetify({
    components,
    directives,
});
```

The first step is to import our icons.

```ts
import * as FluentIcons from "@vicons/fluent";
```

Next we specify an IconSet object which consistents of a single `component`
member:

```ts
import { h, type IconProps, type IconSet } from "vuetify";

const fluent: IconSet = {
    component: (props: IconProps) => {
        return h(props.tag, [
            h(FluentIcons[props.icon as string], { class: "v-icon__svg" }),
        ]);
    },
};
```

The IconProps interface is defined as:

```ts
interface IconProps {
    tag: string | JSXComponent;
    icon?: IconValue;
    disabled?: boolean;
}
```

We use the `tag` property to render the wrapper element with the icon as the
single child with the Vuetify `v-icon__svg` class. This allows us to use the
icon name in `icon` attributes for components:

```vue title="src/components/"
<template>
  <div class="d-flex justify-center">
    <v-btn prepend-icon="AnimalRabbit28Filled">Custom Icon</v-btn>
  </div>
</template>
```

Then we need to add the icon set definition to Vuetify options:

```ts
export const vuetify = createVuetify({
    components,
    directives,
    icons: {
        aliases: {
            prev: "ChevronLeft24Regular",
            next: "ChevronRight24Regular",
            calendar: "CalendarLtr24Regular",
            subgroup: "CaretDown24Regular",
        },
        defaultSet: "fluent",
        sets: {
            fluent,
        },
    },
});
```

We can now see our icon:

![Button with custom icon](./custom-icon.png 'Custom icon set example')

The full setup file is show below:

```ts title="src/main.ts"
import { createApp, h } from "vue";
import { createVuetify, type IconProps, type IconSet } from "vuetify";
import "vuetify/styles";
import App from "./App.vue";
import * as components from "vuetify/components";
import * as directives from "vuetify/directives";
import * as FluentIcons from "@vicons/fluent";

const fluent: IconSet = {
    component: (props: IconProps) => {
        return h(props.tag, [
            h(FluentIcons[props.icon as string], { class: "v-icon__svg" }),
        ]);
    },
};

export const vuetify = createVuetify({
    components,
    directives,
    icons: {
        aliases: {
            prev: "ChevronLeft24Regular",
            next: "ChevronRight24Regular",
            calendar: "CalendarLtr24Regular",
            subgroup: "CaretDown24Regular",
        },
        defaultSet: "fluent",
        sets: {
            fluent,
        },
    },
});

createApp(App).use(vuetify).mount("#app");
```
