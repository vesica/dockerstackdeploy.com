# To build locally

```
docker run -it -v $(pwd)/.:/srv/jekyll -p 4000:4000 jekyll/jekyll jekyll serve --watch --drafts
```