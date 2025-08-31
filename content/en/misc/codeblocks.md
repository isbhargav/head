---
title: Codeblocks
submitted: false
summary: Examples of copy button on codeblocks.

---

## Copy code button on codeblocks

``` yaml
title: "Byte Garden"
baseURL: "https://byte-garden.xdeb.org/"
theme: "github.com/frjo/hugo-theme-zen/v5"
copyright: '<!--Creative Commons License-->This site is licensed under a <a href="https://creativecommons.org/licenses/by-sa/4.0/">CC-BY-SA 4.0</a> licence.<!--/Creative Commons License-->'
languageCode: "en-GB"
defaultContentLanguage: "en"
pluralizelisttitles: false
removePathAccents: true
rssLimit: 20
pagination:
  pagerSize: 10
  path: page

markup:
  goldmark:
    parser:
      attribute:
        block: true
    renderer:
      unsafe: true
```

``` go-html-template
{{ .Page.Store.Set "codeblocks" true -}}
{{ $result := transform.HighlightCodeBlock . -}}
<div data-codeblock itemscope itemtype="https://schema.org/SoftwareSourceCode" {{ range $k, $v := .Attributes }}{{ printf " %s=%q" $k $v | safeHTMLAttr }}{{ end }}>
<button data-codebutton class="codebutton" title="{{ i18n "copy_code" }}"><span>⧉</span></button>
<meta itemprop="codeSampleType" content="snippet">
<meta data-codetext itemprop="text" content="{{ .Inner | jsonify }}">
{{ if .Type -}}
<meta itemprop="programmingLanguage" content="{{ .Type }}">
<pre tabindex="0" class="chroma">
<code class="language-{{ .Type }}" data-lang="{{ .Type }}">{{ $result.Inner }}</code>
</pre>
{{ else -}}
{{ $result.Wrapped }}
{{ end -}}
</div>
```

```python
import os

def main():
    print("Hello, World!")
    os.system("echo Hello, World!") # This is a comment

```

```typescript
import { useState } from "react";

function App() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}
```
