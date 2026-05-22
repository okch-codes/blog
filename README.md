# blog

Personal website and blog of Omar Karim Chtioui, built with
[Hugo](https://gohugo.io/) and the
[PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

Live at <https://blog.okch.pw/>.

## Local development

Requires [Hugo extended](https://gohugo.io/installation/) (0.161+).

```bash
git clone --recurse-submodules https://github.com/okch-codes/blog.git
cd blog
hugo server -D        # http://localhost:1313/  (-D includes drafts)
```

## Writing a post

```bash
hugo new content posts/my-new-post.md
# edit the file, set `draft: false` when ready
git commit -am "post: my new post" && git push
```

## Deployment

Pushing to `main` triggers the GitHub Actions workflow in
`.github/workflows/hugo.yml`, which builds the site and publishes it to
GitHub Pages. Make sure Pages is set to deploy from **GitHub Actions** in
the repository settings.
