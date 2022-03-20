# Serve with docker-compose
Host the website locally [on port 4000](http://localhost:4000):
```
docker-compose up
```

# Build and serve locally
- Install gems from `Gemfile.lock`: `bundle install`
- Build jekyll site: `bundle exec jekyll build`
- Build and serve jekyll site [on port 4000](http://localhost:4000): `bundle exec jekyll serve`