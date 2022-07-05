---
title:  "간단한 문법 사용"
excerpt: "golang을 시작하기 위한 간단한 문법을 사용하여 예시문제 해결"

toc: true
toc_sticky: true

published: true

categories:
  - References
tags:
  - GoogleTest
sitemap:
  changefreq: daily
  priority: 0.8
---

# 1. CMake 설치
사전에 OpenSSL이 설치되어 있어야 한다고 함. 이거는 모지잉 .. ?
```
sudo apt-get install g++
sudo apt-get install build-essential

wget https://github.com/Kitware/CMake/releases/download/v3.24.0-rc2/cmake-3.24.0-rc2.tar.gz
tar -xvf cmake-3.24.0-rc2.tar.gz
cd cmake-3.24.0-rc2/

./bootstrap --prefix=/usr/local
```