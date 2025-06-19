---
title: 记一次前后端分离项目开发的经验
author: sanenchen
date: 2025-06-19 16:52:00 +0800
categories: [前端, 后端, 打包]
tags: [Vite,Golang]
render_with_liquid: false
---
# 记一次前后端分离项目经验
这个项目用的是 Vue + Vite 作为前端 Golang 作为后端  
主要谈一下后端的事情  
## GIN的坑
当用Vite打包好静态文件，尝试将静态文件映射进去时候，发现常用方法是不可以的，因为静态文件路径为"/" 后端 api路径为"/api"，有一个办法就是把前后端api换做不同端口号，但不想这么做，于是用到了一个中间件：```"github.com/gin-contrib/static"```  
这个东西也有坑 由于使用embed将静态文件打包进可执行文件，static原生不支持embed 所以很尴尬 有人写过了static 项目组就是不接受pr  
实现代码：
``` golang
//go:embed dist
var dist embed.FS

//go:embed dist/index.html
var IndexHtml string

type embedFileSystem struct {
	http.FileSystem
}

func (e embedFileSystem) Exists(prefix string, path string) bool {
	_, err := e.Open(path)
	if err != nil {
		return false
	}
	return true
}

func EmbedFolder(fsEmbed embed.FS, targetPath string) static.ServeFileSystem {
	fileSys, err := fs.Sub(fsEmbed, targetPath)
	if err != nil {
		panic(err)
	}
	return embedFileSystem{
		FileSystem: http.FS(fileSys),
	}
}
```
后面再加一句
``` golang
func InitIndex(router *gin.Engine) {
	// 获取 assets
	router.Use(static.Serve("/", EmbedFolder(dist, "dist")))

	// 检测是否为html请求，如果是则返回index.html 就不转发了
	router.NoRoute(func(c *gin.Context) {
		accept := c.Request.Header.Get("Accept")
		flag := strings.Contains(accept, "text/html")
		if flag {
			c.Writer.WriteHeader(200)
			if _, err := c.Writer.Write([]byte(IndexHtml)); err != nil {
				panic(err)
			}
			c.Writer.Flush()
		}
	})
}
```
## 再来谈一谈交叉编译的问题
几经挫折后选择了 Github Actions
build.sh:
以下代码魔改自 alist 的打包程序
``` shell
appName="apps"

BuildDev() {
  rm -rf .git/
  mkdir -p "dist"
  # Build For arm64
  BASE="https://musl.nn.ci/"
  FILES=(aarch64-linux-musl-cross)
  for i in "${FILES[@]}"; do
    url="${BASE}${i}.tgz"
    curl -L -o "${i}.tgz" "${url}"
    sudo tar xf "${i}.tgz" --strip-components 1 -C /usr/local
  done
  OS_ARCHES=(linux-musl-arm64 windows-mingw-amd64)
  CGO_ARGS=(aarch64-linux-musl-gcc x86_64-w64-mingw32-gcc)
  for i in "${!OS_ARCHES[@]}"; do
    os_arch=${OS_ARCHES[$i]}
    cgo_cc=${CGO_ARGS[$i]}
    echo building for ${os_arch}
    export GOOS=${os_arch%%-*}
    export GOARCH=${os_arch##*-}
    export CC=${cgo_cc}
    export CGO_ENABLED=1
    go build -o ./build/$appName-$os_arch -tags=jsoniter .
  done
  # why? Because some target platforms seem to have issues with upx compression
  mv apps-* dist
  cd dist
  upx -9 ./apps-linux*
  upx -9 ./apps-windows*
  find . -type f -print0 | xargs -0 md5sum >md5.txt
  cat md5.txt
}


BuildRelease() {
  rm -rf .git/
  mkdir -p "build"
  # Build For arm64
  BASE="https://musl.nn.ci/"
  FILES=(aarch64-linux-musl-cross)
  for i in "${FILES[@]}"; do
    url="${BASE}${i}.tgz"
    curl -L -o "${i}.tgz" "${url}"
    sudo tar xf "${i}.tgz" --strip-components 1 -C /usr/local
  done
  OS_ARCHES=(linux-musl-arm64 windows-mingw-amd64)
  CGO_ARGS=(aarch64-linux-musl-gcc x86_64-w64-mingw32-gcc)
  for i in "${!OS_ARCHES[@]}"; do
    os_arch=${OS_ARCHES[$i]}
    cgo_cc=${CGO_ARGS[$i]}
    echo building for ${os_arch}
    export GOOS=${os_arch%%-*}
    export GOARCH=${os_arch##*-}
    export CC=${cgo_cc}
    export CGO_ENABLED=1
    go build -o ./build/$appName-$os_arch -tags=jsoniter .
  done
  # why? Because some target platforms seem to have issues with upx compression
  upx -9 ./apps-linux-amd64
  upx -9 ./apps-windows*
  mv apps-* build
}

MakeRelease() {
  cd build
  mkdir compress
  for i in $(find . -type f -name "$appName-linux-*"); do
    cp "$i" apps
    tar -czvf compress/"$i".tar.gz apps
    rm -f apps
  done
  for i in $(find . -type f -name "$appName-darwin-*"); do
    cp "$i" apps
    tar -czvf compress/"$i".tar.gz apps
    rm -f apps
  done
  for i in $(find . -type f -name "$appName-windows-*"); do
    cp "$i" apps.exe
    zip compress/$(echo $i | sed 's/\.[^.]*$//').zip apps.exe
    rm -f apps.exe
  done
  cd compress
  find . -type f -print0 | xargs -0 md5sum >md5.txt
  cat md5.txt
  cd ../..
}

if [ "$1" = "dev" ]; then
    BuildDev
  fi
elif [ "$1" = "release" ]; then
    BuildRelease
    MakeRelease
  fi
else
  echo -e "Parameter error"
fi

```
Build dev Yaml:
``` yaml
name: build

on:
  push:
    branches: [ 'main' ]
  pull_request:
    branches: [ 'main' ]

jobs:
  build:
    strategy:
      matrix:
        platform: [ubuntu-latest]
        go-version: [1.19]
    name: Build
    runs-on: ${{ matrix.platform }}
    steps:
      - name: Setup Go
        uses: actions/setup-go@v3
        with:
          go-version: ${{ matrix.go-version }}

      - name: Checkout
        uses: actions/checkout@v3

      - name: Install dependencies
        run: |
          sudo apt install gcc-mingw-w64 -y
          sudo apt install upx
      - name: Build
        run: |
          bash build.sh dev
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: apps
          path: dist
```

Release Yaml:
``` yaml
name: release

on:
  push:
    tags:
      - '*'

jobs:
  changelog:
    name: Create Release
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - run: npx changelogithub # or changelogithub@0.12 if ensure the stable result
        env:
          GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}
  release:
    needs: changelog
    strategy:
      matrix:
        platform: [ubuntu-latest]
        go-version: [1.19]
    name: Release
    runs-on: ${{ matrix.platform }}
    steps:
      - name: Setup Go
        uses: actions/setup-go@v3
        with:
          go-version: ${{ matrix.go-version }}

      - name: Checkout
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Install dependencies
        run: |
          sudo apt install gcc-mingw-w64 -y
          sudo apt install upx
      - name: Build
        run: |
          bash build.sh release
      - name: Release
        uses: softprops/action-gh-release@v1
        with:
          files: build/compress/*
```


### 新的选择
经过一段时间的沉淀，发现使用GoFrame作为后端开发框架更加优雅，打包的话用GoRelease更好一些，但是涉及C++混编也就是cgo方面依旧很复杂。。

