# keel.sh

This site is built with [hexo](http://hexo.io/). Site content is written in Markdown format located in `src`. Pull requests welcome!

## Developing

Start a dev server at `localhost:4000`:

```
$ npm install -g hexo-cli
$ npm install
$ hexo server
```
## Optimise images

Feel free to use anything that compresses images (Trimage for Linux, ImageOptim for Mac)


Example:

```
trimage -d src/images/
```
