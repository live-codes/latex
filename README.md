# LaTeX in the Browser

Typeset **LaTeX entirely in the browser** — no server, no upload, no install, and no build
step. A document typed into the page is translated to HTML and rendered in that tab.

It is a proof of concept for adding a `latex` language to [LiveCodes](https://livecodes.io),
in the same shape as the [`typst`](../typst) and
[`browser-cobol`](https://github.com/live-codes/browser-cobol) spikes were for their languages.

```
LaTeX source  →  LaTeX.js (pure JavaScript, PEG parser)  →  HTML + CSS + KaTeX  →  rendered
```

The whole demo is a single `index.html`. The compiler is
[LaTeX.js](https://latex.js.org/) — an MIT-licensed, pure-JavaScript LaTeX implementation —
loaded as one ES module from jsDelivr. Nothing is vendored and nothing is built.

## Demo

A static server is required (ES modules cannot be loaded over `file://`), but no special
headers and no cross-origin isolation:

```bash
npx serve .          # or: python -m http.server
```

Open the page, edit the document, press **Run** — or `Ctrl`/`Cmd` + `Enter` in the editor. The
typeset result appears on the right, as a real HTML page inside an `iframe`. Editing re-runs
automatically (debounced), and **Download HTML** / **Open in new tab** export the generated
document.

## What you get

- **Client-side typesetting.** Nothing is uploaded; the parser and generator run in the tab.
- **No build step, no vendor mount.** One HTML file; LaTeX.js comes from jsDelivr.
- **Real LaTeX semantics** for the supported subset: sections and subsections, `title`/
  `author`/`date` with `\maketitle`, `abstract`, `quote`/`quotation`/`verse`, `itemize`,
  `enumerate`, `description`, alignment environments, font commands, groups, `\label`/`\ref`,
  and — via bundled KaTeX — inline `$…$` and display `$$…$$` math.
- **Genuine parse diagnostics** with position, shown instead of the output:

  ```
  Parse error at line 3, column 8
  environment 'itemize' is missing its end, found 'document' instead
  ```

- **Fast.** The default document renders in roughly 15–25 ms.

## Supported, by LaTeX.js

LaTeX.js implements a subset of LaTeX in JavaScript; native `.sty` packages cannot be parsed,
so their macros must exist in the library. In practice this means:

- **Math** is rendered by KaTeX, not TeX. Display math uses `$$…$$`; the `equation` and
  `align` *environments* are **not** available.
- **No `tabular`/`table`/`figure`** environments.
- **`\usepackage` works only for packages LaTeX.js ships**: `comment`, `multicol`,
  `hyperref`, `graphicx`/`graphics`, `color`/`xcolor`, `latexsym`, `textcomp`, `textgreek`,
  `gensymb`, `stix`, `calc`, `pict2e`, `picture`. Anything else errors.
- **Documents, not PDFs.** The output is HTML + CSS styled to look like LaTeX. There is no
  `.pdf` and no page model — page breaks are meaningless in HTML.
- The default document avoids the unsupported constructs above; they are noted here so they
  are not a surprise.

## Layout

```
index.html   the whole demo — styles, editor, preview, and the LaTeX.js driver
```

## Status

Spike complete: `latex.js@0.12.6` renders a realistic document from a single CDN-loaded file
with no build step. Open questions for turning this into a LiveCodes language:

- **Fidelity.** LaTeX.js is a translator, not a TeX engine, so it cannot run arbitrary
  packages — the usual expectation for a LaTeX playground. Producing a real PDF would mean a
  WebAssembly engine instead ([SwiftLaTeX](https://github.com/SwiftLaTeX/SwiftLaTeX) or
  [BusyTeX](https://github.com/busytex/busytex)/[TeXlyre](https://github.com/TeXlyre/texlyre-busytex)),
  both of which are AGPL-3.0 and need ~30–400 MB of engine assets hosted rather than a single
  CDN file.
- **Editor.** A CodeMirror/Monaco mode for LaTeX would replace the plain `<textarea>`.

## License

MIT © Hatem Hosny. The compiler keeps its own license — LaTeX.js is MIT, and its bundled
fonts are under the SIL Open Font License.
