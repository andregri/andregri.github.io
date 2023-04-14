# Serve with docker-compose
Host the website locally [on port 4000](http://localhost:4000):
```
docker-compose up
```

# Build and serve locally

To setup github pages locally, refer to the official [guide](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll#installing-jekyll).

The main steps are:
- Install gems from `Gemfile.lock`: `bundle install`
- Install bundle webrick `bundle add webrick`
- Build jekyll site: `bundle exec jekyll build`
- Build and serve jekyll site [on port 4000](http://localhost:4000): `bundle exec jekyll serve`