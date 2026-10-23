# TrueSky Shader Reference — GitHub Pages site

A small, no-build documentation site that displays the two Markdown references in separate tabs. The original `.md` documents remain editable source files.

## Publish on GitHub Pages

1. Create a new GitHub repository, for example `truesky-shader-reference`.
2. Upload the **contents** of this folder to the repository root (not the enclosing folder). Make sure these four files are included:
   - `index.html`
   - `SQ_PARAMETER_SHADER_REFERENCE.md`
   - `SHADER_VARIABLES_REFERENCE.md`
   - `.nojekyll`

   You can also include `README.md` as the repository's setup notes.
3. Open the repository's **Settings → Pages**.
4. Under **Build and deployment**, choose **Deploy from a branch**.
5. Select branch `main` and folder `/(root)`, then click **Save**.
6. After the Pages deployment completes, use the site URL shown on the Pages settings page. It will normally look like `https://YOUR-USERNAME.github.io/REPOSITORY-NAME/`.

## Updating the documents

Edit either `.md` file and commit/push the changes. The site reads the Markdown files when a tab is opened; the HTML generally does not need to change.

## Notes

- The site uses Marked and DOMPurify from jsDelivr CDNs to render Markdown safely in the browser. An internet connection is required to load those libraries.
- GitHub Pages websites are publicly accessible. Do not publish content you intend to keep private.
- The site should be tested via GitHub Pages or a local HTTP server; opening `index.html` directly as a `file://` URL may block Markdown `fetch()` requests in the browser.
