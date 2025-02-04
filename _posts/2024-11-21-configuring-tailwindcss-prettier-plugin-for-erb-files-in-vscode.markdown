---
title: Configuring TailwindCSS Prettier Plugin for ERB Files in VS Code
date: 2024-11-21 11:10:00 -0300
categories: [Frontend, Tools]
tags: [TailwindCSS, Prettier, ERB, VSCode, Configuration]
---

Sometime ago I configured the [TailwindCSS](https://tailwindcss.com/) plugin to work with [Prettier](https://prettier.io/). I think I followed this tutorial: https://fwuensche.medium.com/how-to-auto-format-erb-files-on-vscode-c82a03565d22, but I'm not paying Medium anymore, so I don't know for sure.

The package used in the post above is the [prettier-plugin-erb](https://github.com/adamzapasnik/prettier-plugin-erb), it doesn't receive updates in the last 3 years and isn't compatible with Prettier 3.

I installed the `prettier` and `prettier-plugin-tailwindcss` anyway and created my `.prettierrc` file with the following content:

```json
{
  "plugins": ["prettier-plugin-tailwindcss"],
  "overrides": [
    {
      "files": "*.erb",
      "options": {
        "parser": "html"
      }
    },
    {
      "files": "erb",
      "options": {
        "parser": "html"
      }
    },
    {
      "files": "*.html.erb",
      "options": {
        "parser": "html"
      }
    }
  ]
}
```

I wasn't able to make it auto format my ERB files, but I was able to use the command `Format Document (Forced)`, and it simply worked on my ERB files, until recently were it started to give an error about an invalid ERB syntax in one of my files.

Before giving up to use this `Prettier` I decided to search out about something new in this land and found [@4az/prettier-plugin-html-erb](https://www.npmjs.com/package/@4az/prettier-plugin-html-erb). After installing it and the gems `prettier_print` and `syntax_tree` for my project I was able to define the `Prettier` plugin as my default ERB formatter on my `VS Code` settings with the following:

```json-doc
{
    // other VS Code settings
    "[erb]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode"
    }
}
```

Now when I save my files (not autosave, I don't like to use it with autosave) `Prettier` does it job perfectly. The `.prettierrc` in this case can also be simpler:

```json
{
  "plugins": ["prettier-plugin-tailwindcss", "@prettier/plugin-ruby", "@4az/prettier-plugin-html-erb"]
}
```

I also found a plugin that can be useful without the need of `Prettier`: [Headwind](https://marketplace.visualstudio.com/items?itemName=heybourn.headwind). I didn't test it, but I will keep it on the radar if something brake again.