source "https://rubygems.org"

# GitHub Pages가 지원하는 Jekyll·플러그인 버전을 한 번에 맞춰주는 메타 gem.
# (github.io 자동 빌드 환경과 동일하게 로컬을 구성)
gem "github-pages", group: :jekyll_plugins

# _config.yml의 plugins와 일치
group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
end

# Windows / JRuby에서 타임존 데이터
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Windows 파일 변경 감지(선택)
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]
