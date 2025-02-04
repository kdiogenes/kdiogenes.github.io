---
layout: post
title: Developing a local NPM package and using it on Rails
categories: [Full Stack, Tools]
tags: [JavaScript, NPM, Rails, Development]
---
How do you install a NPM package from your local filesystem?

`yarn add file:/path/to/my/local/npm/package`

More info on <https://yarnpkg.com/lang/en/docs/cli/add/>

Cool, now I made changes to my local NPM package, how can I get these modifications in my Rails packs?

I'm developing for the [@fnix/gcs-browser-upload](https://github.com/fnix/gcs-browser-upload) package.

This package has a Makefile and I need to run `make compile` to get the file `dist/Upload.js`. After this, in my Rails
application I need to run the yarn upgrade command: `yarn add file:/path/to/my/local/npm/package`

Is this the end?

No, unfortunately! Probably you are running `webpack-dev-server`, so you have to restart it to get recompiled packs. Now
you can hit F5 and get your fresh code running!

Probably there are better workflows for this, but for now is what I can do :\|

If you have any improvement to share, I will be glad to hear!
