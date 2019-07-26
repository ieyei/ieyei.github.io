







## Jekyll on Ubuntu

<https://jekyllrb.com/docs/installation/ubuntu/>



dependencies

```
sudo apt-get install ruby-full build-essential zlib1g-dev
```



environment variables

```
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```



install jekyll

```
gem install jekyll bundler
```



<https://nachwon.github.io/jekyllblog/>

github-pages 설치

```
gem install github-pages
```



로컬에서 시작

```
mkdir jekyll
cd jekyll
jekyll new myblog
```



github-pages 설정

```
cd myblog
vi Gemfile

#gem "jekyll", "~> 3.8.6"

...
gem "github-pages", group: :jekyll_plugins

```



변경 적용

```
bundle install
bundle update
```



브라우저 외부 접속 설정

```
vi _config.yml

# 추가
host: 0.0.0.0
port: 4000
```



##### 로컬 테스트 서버 실행하기

```
bundle exec jekyll serve
```

<http://hostip:4000/>

