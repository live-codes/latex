# LaTeX in the Browser

Typeset **LaTeX entirely in the browser** — no upload, no install, and no build step. Two
demos, two different engines:

| Demo | Engine | Output | Backend needed |
| --- | --- | --- | --- |
| [`index.html`](index.html) | [LaTeX.js](https://latex.js.org/) — pure JavaScript | LaTeX-styled **HTML** | none |
| [`swiftlatex/`](swiftlatex/) | [SwiftLaTeX](https://github.com/SwiftLaTeX/SwiftLaTeX) — pdfTeX in WebAssembly | a real **PDF** | a TeX Live file server |

Both are proofs of concept for adding a `latex` language to [LiveCodes](https://livecodes.io),
in the shape of the [`typst`](../typst) and
[`browser-cobol`](https://github.com/live-codes/browser-cobol) spikes.

## Running

A static server is required (ES modules and workers can't load over `file://`), but no special
headers and **no cross-origin isolation**:

```bash
npx serve .          # or: python -m http.server
```

Then open `/` for the LaTeX.js demo and `/swiftlatex/` for the SwiftLaTeX demo. The headers
link to each other.

## Demo 1 — LaTeX.js (`index.html`)

```
LaTeX source  →  LaTeX.js (pure JavaScript, PEG parser)  →  HTML + CSS + KaTeX  →  rendered
```

One HTML file; the compiler is a single ES module from jsDelivr. Nothing is vendored and
nothing is built. Press **Run** (or `Ctrl`/`Cmd` + `Enter`); the result renders into an
`iframe`, editing re-runs automatically (debounced), and **Download HTML** / **Open in new
tab** export the document.

- **Genuine LaTeX semantics for a subset**: sections, `title`/`author`/`date` with
  `\maketitle`, `abstract`, `quote`/`quotation`/`verse`, `itemize`, `enumerate`,
  `description`, alignment environments, font commands, groups, `\label`/`\ref`, and — via
  bundled KaTeX — inline `$…$` and display `$$…$$` math.
- **Real parse diagnostics** with position, e.g. `Parse error at line 3, column 8` /
  `environment 'itemize' is missing its end`.
- **Fast** — the default document renders in roughly 15–25 ms.

**What it does not do.** It is a translator, not a TeX engine, so `.sty` packages cannot be
parsed and must exist in the library: math uses `$$…$$` (no `equation`/`align`
*environments*), there are no `tabular`/`table`/`figure` environments, and `\usepackage`
works only for the packages it ships (`comment`, `multicol`, `hyperref`, `graphicx`,
`color`/`xcolor`, `latexsym`, `textcomp`, `textgreek`, `gensymb`, `stix`, `calc`, `pict2e`,
`picture`). The output is HTML, not a PDF, and page breaks are meaningless.

## Demo 2 — SwiftLaTeX (`swiftlatex/`)

```
LaTeX source  →  pdfTeX (WebAssembly, in a worker)  →  real PDF  →  <iframe>
```

The real pdfTeX engine, compiled to WebAssembly: PDFTeX 0.3.0, TeX Live packages, actual
PDF output. The engine files are vendored in this folder because the wrapper spawns its worker
by a page-relative path, so a CDN URL cannot be dropped in:

| File | |
| --- | --- |
| `PdfTeXEngine.js` | the engine wrapper (worker API) |
| `swiftlatexpdftex.js` | the worker + Emscripten runtime (~85 KB) |
| `swiftlatexpdftex.wasm` | the pdflatex engine (~1.7 MB) |

No `SharedArrayBuffer` and no threads are involved, so none of this needs COOP/COEP.

### It needs a TeX Live file server — and that server is currently down

The engine runs locally, but LaTeX's *files* are **not** in the wasm. The engine fetches them
on demand from a TeX Live server, at URLs like:

```
<endpoint>pdftex/<kpse-code>/<name>       e.g.  .../pdftex/10/swiftlatexpdftex.fmt
```

The default endpoint, `https://texlive2.swiftlatex.com/`, is **not answering** — it returns
Cloudflare 522 and times out over both HTTP and HTTPS (checked 2026-09-23). SwiftLaTeX's own
demo page depends on the same host, so it is broken too, and there is no public mirror. That
means **this demo cannot produce a PDF out of the box**: it loads the engine, then stops at
the first missing file.

Rather than hang — the engine's own package requests use a 150 s timeout — the page probes for
`pdftex/10/swiftlatexpdftex.fmt` (the first file the engine asks for) with an 8 s timeout and,
when it fails, says which URL it tried and what to do about it.

**To make it compile**, point the *TeX Live server* field (or just type a new one) at a server
that serves the same paths. Self-host one with
[SwiftLaTeX/Texlive-Ondemand](https://github.com/SwiftLaTeX/Texlive-Ondemand) against a local
TeX Live installation; `setTexliveEndpoint` in the docs refers to this. Notes from building
this:

- The engine's `setTexliveEndpoint()` posts the URL to the worker and then **drops its worker
  handle**, leaving the engine unable to compile. This page therefore posts the
  `settexliveurl` message to the live worker directly — see the comment in `swiftlatex/index.html`.
- The format it wants is `swiftlatexpdftex.fmt` (its own preloaded format), not
  `pdflatex.fmt`.

Verified in a headless browser: the engine loads from the vendored files, a document reaches
pdfTeX, and real pdfTeX output is surfaced to the page. What is *not* verified is a successful
compile, because that needs a live TeX Live server.

### Licensing

The SwiftLaTeX repository README says AGPL-3.0, while `PdfTeXEngine.js` itself carries an
EPL-2.0 / GPL-2.0-with-Classpath-exception header. Either way it is not the permissive
licence LiveCodes is used to — worth settling before this ships.

## Layout

```
index.html                 the LaTeX.js demo — the whole thing in one file
swiftlatex/index.html      the SwiftLaTeX demo — editor, PDF preview, endpoint field
swiftlatex/*.js,*.wasm     the vendored pdfTeX engine
```

## Status

Two spikes, and they answer the fidelity question from opposite ends.

**LaTeX.js** works today with no backend at all, but it is a subset translator that cannot run
arbitrary packages — the usual expectation for a LaTeX playground.

**SwiftLaTeX** is the real thing: pdfTeX, real PDFs, real TeX Live packages. But it is only
half self-contained. The engine is local; the LaTeX files are not. Making it work means either
hosting a TeX Live file server (the thing that is currently down) or pre-loading the required
files into the engine's virtual filesystem. For comparison, [BusyTeX](https://github.com/busytex/busytex)
/ [TeXlyre](https://github.com/TeXlyre/texlyre-busytex) attack exactly this by shipping TeX Live
as downloadable data packages — also AGPL-3.0, and 30–400 MB.

Open questions:

- **Fidelity vs. dependencies.** A subset that always works, or a real engine that needs
  ~2 MB of wasm plus a file server (or tens of MB of preloaded TeX Live)?
- **Licensing.** Neither real engine is permissive.
- **Editor.** A CodeMirror/Monaco mode for LaTeX would replace the plain `<textarea>`.

## License

MIT © Hatem Hosny. The engines keep their own licenses — LaTeX.js is MIT (its bundled fonts
under the SIL Open Font License), and SwiftLaTeX is AGPL-3.0 per its repository, with the
EPL-2.0 / GPL-2.0-with-Classpath-exception header on the engine file itself.
