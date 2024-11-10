## Instructions
This repository contains files for [Hugo](https://gohugo.io/), a static site generator. These files are used to generate the blog located at blog.geekycat.in.

Tested on: hugo v0.123.7-312735366b20d64bd61bff8627f593749f86c964+extended linux/amd64 BuildDate=2024-03-01T16:16:06Z VendorInfo=gohugoio
Theme: https://github.com/hugo-sid/hugo-blog-awesome

### Steps to add a post
1. `git clone --recurse-submodules https://github.com/Geeky-cat/geekycat-blog.git`
2. `hugo new content/post/newpost.md`
3. `hugo server` to test the server locally
4. `hugo` to compile and export the contents
5. `git add .` -> `git commit -m "message"` -> `git push`
6. `git clone https://github.com/Geeky-cat/geekycat.github.io`
7. `cp -r geekycat-blog/public/* geekycat.github.io/` -> `cd geekycat.github.io`
8. `git add .` -> `git commit -m "message"` -> `git push`  

The edit should be reflected at https://blog.geekycat.in  
