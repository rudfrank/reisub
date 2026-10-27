---
title: 'Customizing Hugo and PaperMod: terminal-inspired touch with scanlines and custom fonts'
description: "Part 3."
date: '2026-10-27T11:23:50+01:00'
tags: ["hugo","css","theming","papermod"]
author: "rudfrank"
draft: false
showToc: true
series: ["hugo-blog-deploy"]
---

After setting up the layout modifications, I turned my attention to the visual identity of the blog.

I wanted a nerdy **retro terminal / CRT monitor** aesthetic with keeping PaperMod's minimalizm.
We will set self‑hosted monospace fonts for headings, some accent colors, scanlines, a subtle grid background, and tactile pseudo‑3D buttons.

All of this happens in two places:

- `assets/css/extended/custom.css` – the main theme overrides and additions.
- `layouts/partials/` – a small partial that preloads the custom font files.

Here’s a breakdown of the theming changes.

First we create the file `assets/css/extended/custom.css` -- here we can add our styles and override theme and Hugo provided defaults.

---

## Self‑hosted fonts

I prefer to host everything without third-party dependencies.
So after choosing my fonts from Google Fonts, I used the fantastic [google-webfonts-helper](https://gwfh.mranftl.com/fonts) to get them locally.

We will use two typefaces:

- [**Inter**](https://fonts.google.com/specimen/Inter) – sans-serif body font in regular, medium, bold, and italic variants.
- [**VT323**](https://fonts.google.com/specimen/VT323) – a retro terminal font used for headings, code, tags, and other "UI" elements.

### Font files

All fonts are placed in `static/fonts/`:

```
vt323-v18-latin_latin-ext-regular.woff2
inter-v20-latin_latin-ext-regular.woff2
inter-v20-latin_latin-ext-500.woff2
inter-v20-latin_latin-ext-700.woff2
inter-v20-latin_latin-ext-italic.woff2
inter-v20-latin_latin-ext-500italic.woff2
inter-v20-latin_latin-ext-700italic.woff2
```

### Preloading

I added a partial in `layouts/partials/` with preload hints for the most important font files, so the browser fetches them early:

```html
<link rel="preload" href="/fonts/vt323-v18-latin_latin-ext-regular.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/fonts/inter-v20-latin_latin-ext-regular.woff2" as="font" type="font/woff2" crossorigin>
```

### @font-face declarations

In `custom.css`, I declare each font with `font-display: swap` to avoid invisible text while fonts load:

```css
@font-face {
  font-family: 'VT323';
  font-style: normal;
  font-weight: 400;
  font-display: swap;
  src: url('/fonts/vt323-v18-latin_latin-ext-regular.woff2') format('woff2');
}

@font-face {
  font-family: 'Inter';
  font-style: normal;
  font-weight: 400;
  font-display: swap;
  src: url('/fonts/inter-v20-latin_latin-ext-regular.woff2') format('woff2');
}
/* … and the remaining weights/styles */
```

## Applying the fonts

I define two CSS variables to keep font usage consistent:

```css
:root {
  --font: 'Inter', 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  --geekyfont: 'VT323', monospace;
}
```

Then I apply them:

```css
body {
  font-family: var(--font);
  font-optical-sizing: auto;
}

header, .header, .profile, code, pre, kbd, samp, .post-tags, .terms-tags, .toc, .series-container {
  font-family: var(--geekyfont);
  text-shadow: 0 0 1px var(--secondary);
  font-size: 1.25rem;
}

h1, h2, h3, h4, h5, h6, footer nav {
  font-family: var(--geekyfont);
  font-weight: bold;
}
```

I also fine‑tune the heading sizes and code font size so the VT323 font remains readable:

```css
h1 { font-size: 1.8rem; }
h2 { font-size: 1.7rem; }
h3 { font-size: 1.4rem; }

.post-content code { font-size: 1rem; }
```

## Colors and theme variables

PaperMod already supports dark/light themes. I override the tertiary accent to a muted retro green:

```css
:root[data-theme="dark"] {
  --tertiary: #4d7c5a;   /* darker green for dark mode */
}

:root {
  --tertiary: #6fae8e;   /* brighter green for light mode */
}
```

I also remove rounded corners globally to make everything look sharper and more terminal‑like:

```css
:root {
  --radius: 0px;
}

.post-content pre, pre {
  border-radius: 0;
}
```

## CRT and terminal effects

### Global scanlines overlay

A fixed overlay adds subtle horizontal scanlines across the entire page:

```css
body::before {
  content: "";
  position: fixed;
  inset: 0;
  pointer-events: none;
  background: repeating-linear-gradient(
    to bottom,
    transparent 0px,
    transparent 2px,
    rgba(0, 0, 0, 0.03) 2px,
    rgba(0, 0, 0, 0.03) 4px
  );
  z-index: 9999;
}
```

### Animated scanlines on profile and logo

The profile hero, logo, and page header get a moving scanline and a static scanline pattern:

```css
.profile, .logo, header h1 {
  overflow: hidden;
  position: relative;
}

.profile::before, .logo::before {
  width: 100%;
  height: 2px;
  z-index: 9998;
  background-color: var(--theme);
  opacity: 0.66;
  animation: scanline 6s linear infinite;
}

.profile::after, .logo::after, header h1::after {
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  z-index: 9997;
  background: linear-gradient(
    to bottom,
    transparent 50%,
    var(--theme) 51%
  );
  background-size: 100% 4px;
  animation: scanlines 1s steps(60) infinite;
  opacity: 0.33;
}
```

The keyframes drive the moving line and the flickering scanline pattern:

```css
@keyframes scanline {
  0% { transform: translate3d(0, 200000%, 0); }
}

@keyframes scanlines {
  0% { background-position: 0 50%; }
}
```

### Terminal prompt-like decorations

Headings get a `#` prefix, the profile title and logo get a `$`, and the logo gets a blinking underscore cursor:

```css
header h1::before {
  content: "#";
  color: var(--tertiary);
  display: inline-block;
  padding-right: 0.5ex;
}

.profile h1::before, .logo a::before {
  content: "$";
  color: var(--tertiary);
  display: inline-block;
  padding-right: 0.5ex;
}

.logo a::after {
  content: "_";
  display: inline;
  margin-left: -1ex;
  animation: blinker 1s linear infinite;
}
```

We will reviwe the "\<blink\>" element (we use it for our cursor in our custom homepage) -- party like it's early 2000s.

```css
blink {
  animation: blinker 1s linear infinite;
}

@keyframes blinker {
  50% { opacity: 0; }
}
```

### Grid background

The main content area gets a very subtle green grid, inspired by graph paper:

```css
main {
  background-image:
    linear-gradient(rgba(0, 255, 0, 0.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(0, 255, 0, 0.03) 1px, transparent 1px);
  background-size: 13px 13px;
  background-position: 6.5px 10px;
}
```

## TUI-inspired pseudo‑3D buttons

Tags, pagination links, share buttons, and the profile buttons all get an offset shadow. On press they move into the shadow, on hover they lift slightly.


```css
.post-tags a,
.terms-tags a,
.pagination a,
.share-buttons a,
main button,
input[type="submit"],
.top-link, .button, .profile-tag, .post-entry {
  box-shadow: 3px 3px 0 var(--border);
  transition: transform 0.1s ease, box-shadow 0.1s ease;
}

.post-tags a:active,
.terms-tags a:active,
.pagination a:active,
.share-buttons a:active,
main button:active,
input[type="submit"]:active,
.top-link:active, .button:active, .profile-tag:active, .post-entry:active {
  transform: translate(3px, 3px);
  box-shadow: 0 0 0 var(--border);
}

.post-tags a:hover,
.terms-tags a:hover,
.pagination a:hover,
.share-buttons a:hover,
main button:hover,
input[type="submit"]:hover,
.top-link:hover, .button:hover, .profile-tag:hover, .post-entry:hover {
  transform: translate(-1px, -1px);
  box-shadow: 4px 4px 0 var(--border);
}
```

## Additional component styling

### Profile tags

Do not forget to style the tags added to our homepage in [Part 2](posts/hugo2/).

```css
.profile-tags {
  margin-top: 1.25rem;
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  justify-content: center;
}

.profile-tag {
  display: inline-block;
  padding: 0.25rem 0.75rem;
  font-size: 0.85em;
  border-radius: var(--radius);
  background: var(--entry);
  border: 1px solid var(--border);
  color: var(--primary);
  text-decoration: none;
  transition: background 0.2s, border-color 0.2s;
}

.profile-tag:hover, .button:hover {
  background: var(--primary);
  color: var(--theme);
  border-color: var(--primary);
}
```

### Reduced profile height

To keep posts higher on the homepage, I shrink the profile hero:

```css
.profile {
  min-height: auto;
  padding: 2rem 0;
}
```

### Series container

In [Part 2](posts/hugo2/) we set up Series feature, displaying a list when the post is part of a series. Let's style it.
We will style it similarly like TOC, but we will use our tertiary accent-color.

```css
details.series-container {
  margin-bottom: var(--content-gap);
  background: var(--tertiary);
  border-radius: var(--radius);
  border: 1px solid var(--border);
}

details.series-container summary {
  padding: 0.3rem 1.2rem;
  border-radius: var(--radius);
}

.series-container a:hover {
  text-underline-offset: 0.3rem;
  text-decoration: underline;
}
```

---

## What we achieved today

We added a geeky oldschool touch to our theme: green accents, monospace headings and terminal-like decorations, a blinking cursor in the logo, scanlines over the profile and header, a faint grid on the main area, and TUI-inspired buttons.

All of it is accomplished with a single CSS file and a few self‑hosted font files — no JavaScript, no external dependencies.

