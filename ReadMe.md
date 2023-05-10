This is the source of blog

# start a post
with `post_asset_folder` set to true, any posts with internal images should create a a folder under the name of it's post name, then we can use relative path to site the image in the markdown file
```
+ demo.md
+ demo
  +- xxx.png
  +- xxx2.png
```

we can use `hexo new post xxx` to create a new post if `hexo` is installed, or we should do it by ourself

# pushlib
leverage the power of github actions, just push to source branch and let github actions do the left. And of course a github secret of workflow permission is required



# refrence
[How to Add Image to Hexo Blog Post](https://liolok.com/how-to-add-image-to-hexo-blog-post/)