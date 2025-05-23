---
id: bf074448-1967-4c83-8926-9f2005029974
slug: fetch-data-from-directus-with-nuxt
title: Fetch Data from Directus with Nuxt
authors:
  - name: Kevin Lewis
    title: Director, Developer Experience
technologies:
  - nuxt
description: Learn how to integrate Directus in your Nuxt web application.
---
[Nuxt](https://nuxt.com/) is a popular JavaScript framework based on Vue.js. In this tutorial, you will learn how to
build a website using Directus as a [Headless CMS](https://directus.io/solutions/headless-cms). You will store,
retrieve, and use global metadata such as the site title, create new pages dynamically based on Directus items, and
build a blog.

## Before You Start

You will need:

- To install Node.js and a code editor on your computer.
- To sign up for a Directus Cloud account.
- Some knowledge of Vue.js and Nuxt.

Create a new Directus Cloud project - any tier and configuration is suitable for this tutorial.

Open your terminal and run the following commands to create a new Nuxt project and the Directus JavaScript SDK:

```
npx nuxt init my-website
cd my-website
npm install
npm install @directus/sdk
```

Open `my-website` in your code editor and type `npm run dev` in your terminal to start the Nuxt development server and
open http://localhost:3000 in your browser.

## Create a Plugin for the SDK

To expose an Node.js package available globally in your Nuxt project you must create a plugin. Create a new directory
called `plugins` and a new file called `directus.js` inside of it.

```js
import { createDirectus, rest, readItem, readItems } from '@directus/sdk';

const directus = createDirectus('https://directus.example.com').with(rest());

export default defineNuxtPlugin(() => {
	return {
		provide: { directus, readItem, readItems },
	};
});
```

Ensure your Project URL is correct when initializing the Directus JavaScript SDK.

Inside of your `app.vue` entry file, add the following to the bottom to test that your plugin works:

```vue
<script setup>
const { $directus } = useNuxtApp();
console.log($directus);
</script>
```

Refresh your browser, and check the console. You should see the Directus instance logged, which means you have access to
all of the Directus JavaScript SDK methods by using the `useNuxtApp()` composable in any page or component.

Once you've confirmed this, remove the `<script>` from `app.vue` before continuing.

## Using Global Metadata and Settings

In your Directus project, navigate to Settings -> Data Model and create a new collection called `global`. Under the
Singleton option, select 'Treat as a single object', as this collection will have just a single entry containing global
website metadata.

Create two text input fields - one with the key of `title` and one `description`.

Navigate to the content module and enter the global collection. Collections will generally display a list of items, but
as a singleton, it will launch directly into the one-item form. Enter information in the title and description field and
hit save.

![A form named "Global" has two inputs - a title and a description, each filled with some text.](/img/7ea2d6b3-d7ca-4a71-bdaa-cd2ce8c75ec1.webp)

By default, new collections are not accessible to the public. Navigate to Settings -> Access Policies -> Public and give
Read access to the Global collection.

In `app.vue`, remove `<NuxtWelcome />` and replace it with `<NuxtPage />`. This tells Nuxt to use file-based routing.

Create a new directory called `pages` and a new file called `index.vue` inside of it.

```vue
<template>
	<h1>{{global.title}}</h1>
	<p>{{global.description}}</p>
</template>

<script setup>
const { $directus, $readItem } = useNuxtApp()

const { data: global } = await useAsyncData('global', () => {
  return $directus.request($readItem('global'))
})
</script>
```

Refresh your browser. You should see data from your Directus Global collection in your page.

## Creating Pages With Directus

Create a new collection called `pages` - make a text input field called `slug`, which will
correlate with the URL for the page. For example `about` will later correlate to the page `localhost:3000/about`.

Create another text input field `title` and a WYSIWYG input field called `content`. In Access Policies, give the Public
role read access to the new collection. Create 3 items in the new collection -
[here's some sample data](https://github.com/directus-community/getting-started-demo-data).

Inside of `pages`, create a new file called `[slug].vue`. This is a dynamic route, so a single file can be used for all
of the top-level pages.

```vue
<template>
	<h1>{{page.title}}</h1>
	<div v-html="page.content"></div>
</template>

<script setup>
const { $directus, $readItems } = useNuxtApp()
const route = useRoute()

const { data: page } = await useAsyncData('page', () => {
  return $directus.request($readItems('pages', [{slug}]))[0]
})

if (!page.value) throw createError({
  statusCode: 404,
  statusMessage: 'Page Not Found'
})
</script>
```

Go to http://localhost:3000/about, replacing `about` with any of your item slugs. Using the Directus JavaScript SDK, the
 item with that slug is retrieved, and the page should show your data. `readItems()` checks all pages that have the specified `slug` field.

_Note that we check if a returned value exists, and return a 404 if not. Please also note that
[`v-html` should only be used for trusted content](https://vuejs.org/api/built-in-directives.html#v-html)._

## Creating Blog Posts With Directus

Create a new collection called `authors` with a single text input field called `name`. Create one or more authors.

Then, create a new collection called `posts` - add a text input field called `slug`,
which will correlate with the URL for the page. For example `hello-world` will later correlate to the page
`localhost:3000/blog/hello-world`.

Create the following additional fields in your `posts` data model:

- a text input field called `title`
- a WYSIWYG input field called `content`
- an image relational field called `image`
- a datetime selection field called `publish_date` - set the type to 'date'
- a many-to-one relational field called `author` with the related collection set to `authors`

In Access Policies, give the Public role read access to the `authors`, `posts`, and `directus_files` collections.

Create 3 items in the posts collection -
[here's some sample data](https://github.com/directus-community/getting-started-demo-data).

### Create Blog Post Listing

Inside of the `pages` directory, create a new subdirectory called `blog` and a new file called `index.vue` inside of it.

```vue
<template>
	<h1>Blog</h1>
</template>

<script setup>
const { $directus, $readItems } = useNuxtApp()

const { data: posts } = await useAsyncData('posts', () => {
  return $directus.request(
	$readItems('posts', {
		fields: ['slug', 'title', 'publish_date', { 'author': [ 'name' ] }],
		sort: ['-publish_date']
	})
  )[0]
})
</script>
```

This query will retrieve the first 100 items (default), sorted by publish date (descending order, which is latest
first). It will only return the specific fields we request - `slug`, `title`, `publish_date`, and the `name` from the
related `author` item.

Update the `<template>` section:

```vue
<template>
	<h1>Blog</h1>
	<ul>
		<li v-for="post in posts" :key="post.id">
			<NuxtLink :href="`/blog/${post.slug}`">
				<h2>{{post.title}}</h2>
			</NuxtLink>
			<span>{{post.publish_date}} &bull; {{post.author.name}}</span>
		</li>
	</ul>
</template>
```

Visit http://localhost:3000 and you should now see a blog post listing, with latest items first.

![A page with a title of "Blog". On it is a list of three items - each with a title, author, and date. The title is a link.](/img/bcbab1ac-e4b3-4614-b4b5-2be97e71a2ae.webp)

### Create Blog Post Pages

Each blog post links to a page that does not yet exist. In the `pages/blog` directory, create a new file called
`[slug].vue`:

```vue
<template>
	<img :src="`${$directus.url}assets/${post.image.filename_disk}?width=600`" alt="" />
	<h1>{{post.title}}</h1>
	<div v-html="post.content"></div>
</template>

<script setup>
const { $directus, $readItem } = useNuxtApp()
const route = useRoute()

const { data: post } = await useAsyncData('post', () => {
  return $directus.request(
    $readItem('posts', {
			fields: ['*', { slug: route.params.slug, image: ['filename_disk']}],
    })
  )
})

if (!post.value) throw createError({
  statusCode: 404,
  statusMessage: 'Post Not Found'
})
</script>
```

Some key notes about this code snippet.

- In the `<img>` tag, `$directus.url` is the value provided when creating the Directus plugin.
- The `width` attribute demonstrates Directus' built-in image transformations.
- Once again, `v-html` should only be used if all content is trusted.
- Because almost-all fields are used in this page, including those from the `image` relational field, the `fields`
  property when using the Directus JavaScript SDK can be set to `*.*`.

Click on any of the blog post links, and it will take you to a blog post page complete with a header image.

![A blog post page shows an image, a title, and a number of paragraphs.](/img/5603-4992-9c5b-b08765a9186a.webp)

## Add Navigation

While not strictly Directus-related, there are now several pages that aren't linked to each other. In `app.vue`, above
the `<NuxtPage />` component, add a navigation. Don't forget to use your specific page slugs.

```vue-html
<nav>
	<NuxtLink to="/">Home</NuxtLink>
	<NuxtLink to="/about">About</NuxtLink>
	<NuxtLink to="/conduct">Code of Conduct</NuxtLink>
	<NuxtLink to="/privacy">Privacy Policy</NuxtLink>
	<NuxtLink to="/blog">Blog</NuxtLink>
</nav>
```

## Next Steps

Through this guide, you have set up a Nuxt project, created a Directus plugin, and used it to query data. You have used
a singleton collection for global metadata, dynamically created pages, as well as blog listing and post pages.

If you want to change what is user-accessible, consider setting up more restrictive roles and accessing only valid data
at build-time.

If you want to build more complex dynamic pages made out of reusable components, check out
[our recipe on doing just this](/tutorials/getting-started/create-reusable-blocks-with-many-to-any-relationships).

If you want to see the code for this project, you can find it
[on GitHub](https://github.com/directus/examples/blob/main/website-nuxt3).
