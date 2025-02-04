---
title: UI Components for TailwindCSS
date: 2025-02-03 20:10:00 -0300
categories: [Frontend, UI Components]
tags: [TailwindCSS, Rails, Hotwire, React, Vue, JavaScript]
---

Recently I released a project called [WaveCaption](https://wavecaption.com) that uses [TailwindUI](https://tailwindui.com). It was a simple project, so I didn't use React or Vue, just Rails with Hotwire. It worked well, but I get a bit annoyed about having to write JS to handle dropdowns, drawers, and other UI components.

Recently, I read [Build a (progressively enhanced) drawer component with Hotwire](https://thoughtbot.com/blog/hotwire-drawer)[^1] and started to think if my life would be simpler if I have used the React or Vue components. I never did a project that used only React or Vue in the frontend, but I know a bit about them, but not enough to change the example on this post to use them without spending quite some time.

There are two ways someone can adapt this example to use React:

1. Use the same workflow and use some strategy to mount the UI components.
2. Develop an API and write the frontend in React.

There are some nice approaches to integrate React with Rails, like the ones explained in [How to integrate React with Rails 7](https://thoughtbot.com/blog/how-to-integrate-react-rails) and [The art of Turbo Mount: Hotwire meets modern JS frameworks](https://evilmartians.com/chronicles/the-art-of-turbo-mount-hotwire-meets-modern-js-frameworks). If I'm not missing anything, this would allow to use the same code presented in the drawer component example[^1], but we could use this TailwindUI component: <https://tailwindui.com/components/application-ui/overlays/drawers>. The advantage of this is that we don't need to write the JS to handle the drawer.

The second approach allows a much more simple frontend IMHO, at least for this simple example. All the page behavior can be written in React and the use of the [View Transition API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API) isn't necessary.

I think that the drawer component example[^1] is a bit convoluted on purpose to demonstrate how to use the View Transition API and also because it avoided using a bit more of JS. But changing the application to be an API and use React for the frontend is overkill for simple projects.

I really liked to work with TailwindCSS, so I started to think if there aren't any project that provides UI components for TailwindCSS and don't require a JS framework. I found some projects that really caught my attention:

1. [Meraki UI](https://merakiui.com/)
2. [daisyUI](https://daisyui.com/)
3. [FlyonUI](https://flyonui.com/)
4. [Flowbite](https://flowbite.com/)
5. [HyperUI](https://www.hyperui.dev/)
6. [preline](https://preline.co/)

I liked **Meraki UI**, but didn't find anything special about it. It doesn't have many components, but appears to be simple and uses [Alpine.js](https://alpinejs.dev/). The only downside is that it doesn't appear to have a good accessibility support.

I found fantastic how **daisyUI** and **HyperUI** implements lots of good components using only CSS, but I want components that come with JS to handle the behavior, so I don't think it's a good fit for me. **HyperUI** also have a very different style from the others, something that pleased me a lot.

**FlyonUI** uses **daisyUI** to provide components and **preline** to provide accessibility. It also provides a JS library to handle the behavior of the components.

**Flowbite** and **preline** appears to be very complete, they also have a business model similar to **TailwindUI**, something that I think is good to support the development of the projects.

I think that **FlyonUI**, **Flowbite** and **preline** are excellent alternatives to **TailwindUI**, but for my personal projects I'm really inclined to use **FlyonUI** due the use of simpler classes to define components, although it still leaves you with the freedom to apply any TailwindCSS classes.

So, after all this research, I have to experiment with **FlyonUI** to see how it feels on a Rails project. I expect to be tinkering with the drawer component example to see how **FlyonUI** fits in a Rails + Hotwire project. See you soon!

[^1]: <https://thoughtbot.com/blog/hotwire-drawer>