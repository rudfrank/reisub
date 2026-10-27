---
title: 'Customizing Hugo and PaperMod: combined profile homepage and series navigation'
description: "Part 2."
date: '2026-10-26T17:45:18+01:00'
tags: ["hugo","theming","papermod"]
author: "rudfrank"
draft: false
showToc: true
series: ["hugo-blog-deploy"]
---

In [Part 1](posts/hugo1/), we installed Hugo and initiated our git repo, then added [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

PaperMod presents a choice for the homepage: either a profile hero (profileMode) **or** a post list, not both. \
Also, I write multi‑part series, and PaperMod has no built‑in way to navigate between posts in the same series.

So I made some layout changes:

1. A **combined homepage** based on profileMode, but also showing recent posts on the same page. We will also display the top five tags for quick navigation.
2. **Series support** that displays a collapsible list of all posts in the same series, plus the series description on each post.

Fortunately, in Hugo, theme layouts can be overridden simply by creating files in layouts/... with a matching name.
So we can copy original files from eg. themes/PaperMod/layouts/... then make our edits.

---

## Combined profile + post list homepage

By default, when `profileMode.enabled` is `true` in `hugo.yml`, PaperMod renders the profile hero and stops. Posts are hidden unless you navigate to the posts section.

Let's override the index.html with `layouts/index.html` to show both.

### homepage: `layouts/index.html`

```go-html-template
{{- define "main" }}

{{- if .IsHome }}

  {{- $pages := where site.RegularPages "Type" "in" site.Params.mainSections }}
  {{- $pages = where $pages "Params.hiddenInHomeList" "!=" "true" }}
  {{- $paginator := .Paginate $pages }}

  {{/* Show the profile hero only on the first page */}}
  {{- if and site.Params.profileMode.enabled (eq $paginator.PageNumber 1) }}
    {{- partial "index_profile.html" . }}
  {{- end }}

  {{/* Show the home info block (if you have homeInfoParams) */}}
  {{- if and site.Params.homeInfoParams (eq $paginator.PageNumber 1) }}
    {{- partial "home_info.html" . }}
  {{- end }}

  {{/* Post list */}}
  <section class="profile-posts">
    {{- range $paginator.Pages }}
      <article class="post-entry">
        <header class="entry-header">
          <h2 class="entry-hint-parent">
            <a href="{{ .Permalink }}">{{ .Title }}</a>
          </h2>
        </header>
        <section class="entry-content">
          <p>{{ .Summary | plainify | htmlUnescape }}{{ if .Truncated }}…{{ end }}</p>
        </section>
        <footer class="entry-footer">
          <time>{{ .Date.Format "January 2, 2006" }}</time>
        </footer>
      </article>
    {{- end }}
  </section>

  {{/* Pagination */}}
  {{- if gt $paginator.TotalPages 1 }}
  <footer class="page-footer">
    <nav class="pagination">
      {{- if $paginator.HasPrev }}<a class="prev" href="{{ $paginator.Prev.URL }}">« Prev</a>{{- end }}
      {{- if $paginator.HasNext }}<a class="next" href="{{ $paginator.Next.URL }}">Next »</a>{{- end }}
    </nav>
  </footer>
  {{- end }}

{{- else }}

  {{/* For non-home pages, keep PaperMod’s default behavior */}}
  {{- if .Content }}
  <div class="post-content">
    {{ .Content }}
  </div>
  {{- end }}
{{- end }}
{{- end }}
```

The markup is simple: just title, plain‑text summary, and date. Everything else stays out of the way.

### top tags: `layouts/partials/index_profile.html`

This is PaperMod’s original profile partial, but with one important addition: a **top tags** section.

I copied `themes/PaperMod/layouts/_partials/index_profile.html`, then at the end of the `.profile_inner` block, I added:

```go-html-template
{{- if site.Taxonomies.tags }}
  {{- $tagSlice := slice }}
  {{- range $name, $taxonomy := site.Taxonomies.tags }}
    {{- $tagSlice = $tagSlice | append (dict "name" $name "count" $taxonomy.Count) }}
  {{- end }}

  <div class="profile-tags">
    {{- range sort $tagSlice "count" "desc" | first 5 }}
      {{- $tag := .name }}
      <a class="profile-tag" href="{{ "tags/" | relLangURL }}{{ $tag | urlize }}">
        {{ $tag }}
      </a>
    {{- end }}
  </div>
{{- end }}
```

This builds a slice of `{name, count}` for all tags, sorts by count descending, and renders the top five as links. Since it’s inside the profile hero, the tags appear right under the social icons and action buttons.


## Series support

Besides tags, I needed a series taxonomy (for multi-part posts like this).

First, I set the taxonomies in `hugo.yml` under languages:
```yaml
    taxonomies:
      tag: tags
      series: series
```

Next I added a partial that collects posts belonging to the same series and displays them in a collapsible list.

### series partial: `layouts/partials/series.html`

```go-html-template
{{ $series := slice }}
{{ if isset .Params "series" }}
    {{ $postMainSeries := (index .Params.series 0) }}
    {{ $series = (where .Site.RegularPages "Params.series" "intersect" (slice $postMainSeries)).ByDate }}
    {{ $seriesLen := $series | len }}

    {{ $seriesTitle := $postMainSeries }}
    {{ with .Site.GetPage (printf "/series/%s" ($postMainSeries | urlize)) }}
        {{ with .Title }}
            {{ $seriesTitle = . }}
        {{ end }}
    {{ end }}

    {{ if gt $seriesLen 1 }}
    <details class="series-container" {{ if lt $seriesLen 5 }}open{{ end }}>
        <summary>
            <span class="title">Series: {{ $seriesTitle }}</span>
        </summary>
        <div class="inner">
            <ul class="series-list">
                {{- range $post := $series -}}
                <li><a href="{{$post.RelPermalink}}">{{ $post.Params.title }}</a></li>
                {{ end }}
            </ul>
        </div>
    </details>
    {{ end }}
{{ end }}
```

How it works:

- It takes the **first** item in the post’s `series` front matter array as the main series.
- It finds all regular pages whose `series` array intersects that value.
- It sorts them by date, so you can read them in chronological order.
- It looks for a taxonomy page at `/series/<slug>/` to get a nicer title.
- If there’s more than one post in the series, it renders a `<details>` element. The list is auto‑opened for short series (fewer than 5 posts), collapsed otherwise.

### using partial in post display: `layouts/single.html`

I copied `themes/PaperMod/layouts/single.html` then inserted the series partial before the table of contents:

```go-html-template
{{- partial "cover.html" (dict "cxt" . "IsSingle" true "isHidden" $isHidden) }}
{{ partial "series.html" . }}
{{- if (.Param "ShowToc") }}
{{- partial "toc.html" . }}
{{- end }}
```

I also wanted the **series description** to appear under the post description, giving readers a quick overview of what the series is about. In the post description block, I added:

```go-html-template
{{ if isset .Params "series" }}
  {{ $postMainSeries := (index .Params.series 0) }}
  {{ with .Site.GetPage (printf "/series/%s" ($postMainSeries | urlize)) }}
      {{ if .Description }}
          <div class="series-description">
              {{ .Description }}
          </div>
      {{ end }}
  {{ end }}
{{ end }}
```

So if you set a description on the series taxonomy page, it appears directly below the post’s own description.

When I want to use it, I just need to create `content/series/<series short title>/_index.md`:
```yaml
---
title: "Series human-readable title"
description: "Short description to show"
---
```
It can be omitted -- then it simply won't display and the short title (set in the post header in `series:`) will be used.

---

## What we achieved today

The homepage now shows our profile hero, top tags, and then the most recent posts — all on one page, with pagination still working normally.

Each post that belongs to a series displays a collapsible “Series: …” box with links to all posts in that series, plus an optional series description under the title.

All the layout logic is cleanly separated into partials, so the changes don’t interfere with PaperMod’s other features.

In the next part we will add some geeky styling with css and customize PaperMod further. Stay tuned.
