---
title: Doing layouts with Vue
description: Learn how to a dynamic layout system with Vue
date: '2021-02-15'
tags: [vuejs, layout, dynamic]
---

Whenever you create a large application, you'll often have to deal with different layouts. In this post, we'll explore how we can leverage
Vue's powerful component features that allows each component to specify which layout to use.

We are going to be using a standard Vue 2 setup with the vue router package installed. The code is available <a href="https://codesandbox.io/s/vue-doing-layouts-with-vue-gxf1c" target="_blank">here</a>.

## Layout components
As an example, let's say that we are building a site that has two different layouts: one for a standard page and one for blog posts.

*src/layouts/StandardLayout.vue*

```markup

<template>
  <div>
    <h1>My Website</h1>
    <div>
      <router-link to="/">Home</router-link> |
      <router-link to="/blog">Blog</router-link>
    </div>
    <slot></slot>
  </div>
</template>

<script>
export default {
  name: "StandardLayout",
};
</script>
```

The `StandardLayout` component simply offers a default slot for other components to display their content.


*src/layouts/BlogLayout.vue*

```markup

<template>
  <div>
    <div>
      <a href="/blog">Back to blog</a>
    </div>
    <slot></slot>
  </div>
</template>

<script>
export default {
  name: 'BlogLayout'
}
</script>
```

## Setting the layout

The next step is to modify the root App component and wrap the `router-view` with a dynamic component:

*src/App.vue*
```markup

<template>
  <div id="app">
    <component :is="layout">
      <router-view></router-view>
    </component>
  </div>
</template>

<script>
import StandardLayout from "./layouts/StandardLayout";
import BlogLayout from "./layouts/BlogLayout";

export default {
  name: "App",
  components: {
    "standard-layout": StandardLayout,
    "blog-layout": BlogLayout,
  },
  data() {
    return {
      layout: "standard-layout",
    };
  },
};
</script>
```

Lets look at the code in detail:

```html

<component :is="layout">
  <router-view></router-view>
</component>
```

Here we are using the special `component` tag which is Vue's way of implementing dynamic components (add link to vue doc).
The `is` property determines which component gets rendered, so whenever that value changes, the component gets
re-rendered.

The advantage of this technique is that when navigating from one component to another with the same layout,
the whole layout component doesn't get re-rendered.


```js

  components: {
    "standard-layout": StandardLayout,
    "blog-layout": BlogLayout,
  },
  data() {
    return {
      layout: "standard-layout",
    };
  },
```

Here, we register the layout components to tag names and set the `layout` data variable to the standard layout tag. This
provides a default if a component doesn't specify a layout.


## Using the layouts

Now that we have our basic layouts set up We can create some page components:

*src/components/Home.vue*
```markup

<template>
  <div>
    <h2>Home</h2>
    <p>This is the home page</p>
  </div>
</template>

<script>
export default {
  name: "Home",
};
</script>
```

And a basic blog page

*src/components/Blog.vue*
```markup

<template>
  <div>
    <h1>Blog</h1>
    <h2>First Post</h2>
    <p>First blog post</p>
  </div>
</template>
<script>
export default {
  name: "Blog",
};
</script>
```


We'll register the routes in the main entry point file of our app:

*src/main.js*

```js

import Vue from "vue";
import VueRouter from "vue-router";
import App from "./App.vue";
import Home from "./components/Home.vue";
import Blog from "./components/Blog.vue";

Vue.config.productionTip = false;

Vue.use(VueRouter);

const routes = [
  { path: "/", component: Home },
  { path: "/blog", component: Blog }
];

const router = new VueRouter({ routes });

new Vue({
  router,
  render: (h) => h(App)
}).$mount("#app");
```

## Specifying the layout

Navigating to the home page, we see this:

![Home Page](/images/doing-layouts-with-vue/01-home-page.png)

So far so good. Let's see the blog page:

![Blog Page](/images/doing-layouts-with-vue/02-blog-page.png)

We can see that the it's not using the `BlogLayout` layout component, which is what we expect as don't
have a way for the page components to set the `layout` property in the `App` component.

The way we'll do this is by setting an option called `layout` in our components:

*src/components/Home.vue*
```js
export default {
  name: "Home",
  layout: 'standard-layout',
};
```

*src/components/Blog.vue*
```js
export default {
  name: "Blog",
  layout: 'blog-layout',
};
```

## Updating the layout

So how do we use this option to update our layout component? The answer is to add a global navigation guard
to our view router configuration:

```js
const router = new VueRouter({ routes });
const layout = new Vue.observable({ name: "standard-layout" });

router.layout = layout;

router.afterEach((to) => {
  if (to.matched.length === 0) {
    return "standard-layout";
  }

  const layoutName = to.matched[0].components.default.layout;
  if (layoutName) {
    router.layout.name = layoutName;
  }
});

```

There's a few things going on here, so let's go step by step. We first create a new reactive object using
Vue's `observable` method:

```js

const layout = new Vue.observable({ name: "standard-layout" });
```

And we then attach it to the router object
```js

router.layout = layout;
```

The reactive object acts like a very lightweight data store for specifying our layout. We could have used
something more sophisticated, like an injected plugin or even a Vuex store module, but for our purposes this will suffice.

This means that we can use the `$router.layout` property in any component and changing the `name` member will
update the layout.

Now let's have a look at the navigation guard:

```js

router.afterEach((to) => {
  if (to.matched.length === 0) {
    return "standard-layout";
  }

  const layoutName = to.matched[0].components.default.layout;
  if (layoutName) {
    router.layout.name = layoutName;
  }
});
```

We first check if we actually matched a component during the routing, and return the default layout name
if no components matched. The next step is to get the layout from the component object and then update
the reactive `layout.name` property.


As a last step, we need to change our `App` component to use the router's layout property:

*src/App.vue*
```markup

<template>
  <div id="app">
    <component :is="$router.layout.name">
      <router-view></router-view>
    </component>
  </div>
</template>
```

Now if we check, the blog page, we can see it is using the correct blog layout:

![Blog Page](/images/doing-layouts-with-vue/03-blog-page.png)


And thats about it. An extra step could be to ensure that the layout actually exists


