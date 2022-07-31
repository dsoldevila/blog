```bash
docker run -ti   --volume="$PWD:/srv/jekyll:Z"   -p 4000:4000   --entrypoint /bin/bash jekyll/jekyll
jekyll serve
```