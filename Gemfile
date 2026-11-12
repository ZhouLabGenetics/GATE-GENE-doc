# For local preview only. GitHub Pages builds this site itself from the
# branch selected in Settings > Pages; it does not read this file.
#
#   bundle install
#   bundle exec jekyll serve
#
source "https://rubygems.org"
gem "github-pages", group: :jekyll_plugins
gem "jekyll-remote-theme"
gem "jekyll-seo-tag"
# Ruby 3 dropped webrick from the standard library; `jekyll serve` needs it.
gem "webrick"
# Ruby 3.4 removed these from the default gems. The jekyll 3.x chain that the
# github-pages gem pins still requires them, so a local build on a modern Ruby
# needs them named explicitly. GitHub Pages does not read this file and builds
# on its own Ruby, where they are still present.
gem "base64"
gem "csv"
gem "logger"
gem "ostruct"
gem "bigdecimal"
