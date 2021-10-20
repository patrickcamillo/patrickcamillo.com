# Just Another Bear

A [Hugo](https://gohugo.io/)-theme based on [Hugo ʕ•ᴥ•ʔ Bear Blog](https://github.com/janraasch/hugo-bearblog), which is based on [Bear Blog](https://bearblog.dev).

> Free, no-nonsense, super-fast blogging.

## Author notes

This is still in development. I will improve everything you see below. But in this current state, the theme is already functional.

## Demo

Working on it! But [my own website](https://patrickcamillo.com) uses this theme.

## Screenshots

Working on it!

When the user's browser is running »dark mode«, the dark color scheme will be used automatically. The default is the light/white color scheme. Check out the [`style.html`](https://github.com/janraasch/hugo-bearblog/blob/master/layouts/partials/style.html)-file for the implementation.

## Installation

If you already have a Hugo site on your machine, you can simply add this theme via

```
git submodule add https://gitlab.com/pckcml/just-another-bear.git themes/just-another-bear
```

Then, adjust the `config.toml` as detailed below.

For more information, read the official [setup guide](https://gohugo.io/getting-started/installing) of Hugo.

## Adjust configuration / config.toml

Please check out the [config.toml](https://gitlab.com/pckcml/just-another-bear/-/blob/main/exampleSite/config.toml) included in the [exampleSite](https://gitlab.com/pckcml/just-another-bear/-/tree/main/exampleSite) of this theme.

## Content & structure

### Starting fresh

If you are starting fresh, simply copy over the contents of the `exampleSite`-directory included in this theme to your source directory. That should give you a good idea about how things work, and then you can go on from there to make the site your own.

### Adding / editing content

#### Index-Page

The contents of the `index`-page may be changed by editing your `content/_index.md`-file.

#### Page

You can add **a new page** via running

```
hugo new my-new-page.md
```

#### Blog-Post

You can add **a new blog-post** via running

```
hugo new blog/language/my-new-post.md
```

### Adding your branding / colors / css

Add a `custom_head.html`-file to your `layouts/partials`-directory. In there you may add a `<style>`-tag, *or* you may add a `<link>`-tag referencing your own `custom.css` (in case you prefer to have a separate `.css`-file). Check out the [`style.html`](https://gitlab.com/pckcml/just-another-bear/-/blob/main/layouts/partials/style.html)-file to find out which CSS-styles are applied by default.

## Issues / Feedback / Contributing
Please use [GitLab issues](https://gitlab.com/pckcml/just-another-bear/-/issues) and [Merge Requests](https://gitlab.com/pckcml/just-another-bear/-/merge_requests).

If you do not have a GitLab-account, please hit me up via [e-mail](mailto:patrick.camillo@outlook.com).

## Special Thanks 🎁

A special thank you goes out to [Herman](https://herman.bearblog.dev), for creating the original [ʕ•ᴥ•ʔ Bear Blog](https://bearblog.dev/) and [Jan Raasch](https://www.janraasch.com/) for creating the first iteration of the [Hugo ʕ•ᴥ•ʔ Bear Blog Theme](https://github.com/janraasch/hugo-bearblog).

## License
[MIT License](http://en.wikipedia.org/wiki/MIT_License) © [Patrick Camillo](https://patrickcamillo.com)