source "https://rubygems.org"

gem 'jekyll', '~> 3.9.5' # Add the jekyll gem explicitly
gem 'kramdown-parser-gfm'
gem "jekyll-paginate"
gem 'webrick'
gem 'rouge'
gem 'concurrent-ruby', '~> 1.3.4' # Add the missing gem
gem 'public_suffix', '~> 6.0.1' # Add the missing gem

group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.15.1" # Update to the latest version
end

# Add jekyll-theme-primer if you're using this theme
gem 'jekyll-theme-primer'

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.1", :platforms => [:mingw, :x64_mingw, :mswin]

gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]