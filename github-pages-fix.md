# GitHub Pages Fix

What happened: the file currently named `index.html` is actually a Markdown code bundle. Browsers do not run Markdown as an app, so GitHub Pages prints the code text on the page.

Use this instead:

- Upload `outputs/index.html` as the repository root file named `index.html`.
- Do not rename `cut-command-code.md` to `index.html`.
- Keep the file extension exactly `.html`.
- GitHub Pages should be set to deploy from the `main` branch and `/root`.

This standalone `index.html` has the app code, styles, and JavaScript in one file, so it does not need npm, React, or a build step.
