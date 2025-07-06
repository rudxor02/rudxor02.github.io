# rudxor02.github.io

## 설치 (macOS)

### 1. Homebrew 설치 (이미 설치되어 있다면 건너뛰기)

```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 2. Ruby 설치

```shell
brew install ruby
```

### 3. PATH 설정

```shell
echo 'export PATH="/usr/local/opt/ruby/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### 4. Bundler 설치

```shell
gem install bundler
```

### 5. 의존성 설치

```shell
bundle install
```

## 실행

```shell
bundle exec jekyll serve
```

서버가 실행되면 `http://localhost:4000`에서 사이트를 확인할 수 있습니다.

## 문제 해결

### 권한 오류가 발생하는 경우

시스템 Ruby 대신 Homebrew로 설치한 Ruby를 사용하고 있는지 확인하세요:

```shell
which ruby
# /usr/local/opt/ruby/bin/ruby 가 출력되어야 합니다
```

### 개발 도구 오류가 발생하는 경우

Xcode Command Line Tools가 설치되어 있는지 확인하세요:

```shell
xcode-select --install
```

### Ruby 3.4에서 csv 라이브러리 오류가 발생하는 경우

Jekyll 버전을 업데이트하세요:

```shell
# Gemfile에서 jekyll 버전을 "~> 4.4.0"으로 변경 후
bundle update
```

### 서버 종료

Jekyll 서버를 종료하려면:

```shell
pkill -f jekyll
```
