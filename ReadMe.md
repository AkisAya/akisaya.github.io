This is the source of blog

# start a post
with `post_asset_folder` set to true, any posts with internal images should create a a folder under the name of it's post name, then we can use relative path to site the image in the markdown file
```
+ demo.md
+ demo
  +- xxx.png
  +- xxx2.png
```
And you may read this post [How to Add Image to Hexo Blog Post](https://liolok.com/how-to-add-image-to-hexo-blog-post/) to understand images in hexo post

we can use `hexo new post xxx` to create a new post if `hexo` is installed, or we can do it manually

# publish
Just push to the `source` branch — GitHub Actions handles everything else. No personal access token is needed.

The repo keeps two responsibilities in two branches:

- `source` — everything you edit: markdown posts, configs, themes.
- `master` — build output only (`public/`), which GitHub Pages serves. Never edit it by hand.

Want to change the readme people see on `master`? Edit [`source/readme.md`](source/readme.md) — it is listed under `skip_render` in `_config.yml`, so Hexo copies it verbatim into `public/readme.md` instead of turning it into a page.

## how the action works

Defined in [`.github/workflows/build.yml`](.github/workflows/build.yml), triggered on every push to `source` (or manually from the Actions tab):

1. **Build** — checkout, Node 18, `npm install`, then `npm run build` (= `hexo generate`), which renders everything into `public/`.
2. **Auth** — before deploying, one line gives every git request to github.com a credential:

   ```sh
   git config --global --add http.https://github.com/.extraheader \
     "AUTHORIZATION: basic $(printf 'x-access-token:%s' "$GITHUB_TOKEN" | base64 | tr -d '\n')"
   ```

   `$GITHUB_TOKEN` is issued by GitHub itself for each run and dies with the job, so there is nothing to rotate or expire. The workflow asks for `contents: write`, which is what allows this token to push back to `master`.
   Putting the credential in a global git config (instead of rewriting the repo URL) means it also applies to the throwaway clone that `hexo-deployer-git` makes on its own.
3. **Deploy** — `npm run deploy` (= `hexo deploy`) commits `public/` into that clone and force-pushes it to `master`. GitHub Pages picks the change up and the site updates within a minute or two.

Want to preview before publishing? `npx hexo server` serves locally at <http://localhost:4000>.

> Note: a push made by `GITHUB_TOKEN` intentionally does **not** start another workflow run, so `master` commits never loop.
