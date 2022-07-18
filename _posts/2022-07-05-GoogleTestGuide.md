---
title:  "Google Test 시작하기"
classes:  wide
categories:
  - References
tags:
  - GoogleTest
---

# GoogleTest

[GoogleTestGuides]([http://google.github.io/googletest/quickstart-bazel.htm](http://google.github.io/googletest/quickstart-bazel.html))

# 1. Prerequisites

**Linux, maxOS, Windows**

**C++ Compiler**

적어도 C++14 이상은 되어야 한다.

**CMake 설치**

우선 google test를 빌드하기 위해 cmake가 설치되어야 한다. cmake를  설치하기 위해서는 g++ 컴파일러, make, OpenSSL이 사전에 설치되어 있어야 한다. 

[Ubuntu에서 CMake 설치 방법](https://mong9data.tistory.com/124)

```bash
sudo apt-get install g++
sudo apt-get install build-essential

wget <https://github.com/Kitware/CMake/releases/download/v3.24.0-rc2/cmake-3.24.0-rc2.tar.gz>
tar -xvf cmake-3.24.0-rc2.tar.gz
cd cmake-3.24.0-rc2/

./bootstrap --prefix=/usr/local
```

# 2. Set up a project

CMake는 프로그램을 빌드하기 위해 `CMakeLists.txt` 라는 이름의 파일을 사용합니다. GoolgeTest의 의존성을 명시하기 위해 이 파일을 사용하게 됩니다. 

일단, 작업 디렉토리를 만듭니다.

```bash
mkdir gtest && cd gtest
```

그 다음으로, CMakeLists.txt 파일을 생성하고 GoogleTest의 의존성을 명시합니다. CMake  ecosystem안에 의존성을 나타내는 많은 방법이 있습니다. 지금은 `FetchContent CMake Module`을 사용해서 나타냅니다. 

```elm
cmake_minimum_required(VERSION 3.14)
project(my_project)

# GoogleTest requires at least C++14
set(CMAKE_CXX_STANDARD 14)

include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/609281088cfefc76f9d0ce82e1ff6c30cc3591e5.zip
)
# For Windows: Prevent overriding the parent project's compiler/linker settings
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)
```

위의 설정은 GitHub에서 가져와 GoogleTest의 의존성을 명시합니다.

# 3. Create and run a binary

build 디렉토리를 만들어줍니다. 그대로 작업 디렉토리에 build 를 해도 상관은 없습니다.

```elm
mkdir build && cd build && cmake ..
```

소스를 컴파일하고 생성된 바이너리를 설치합니다.

```elm
make && make install
```

설치가 끝나고 나면 `/usr/local/lib` , `/usr/local/include` 에 gtest와 관련된 라이브러리와 헤더파일이 생성됩니다.

이제 테스트할 항목을 만듭니다. 간단하게 util.h 와 main.cpp 를 만들었습니다.

```elm
// util.h
int sum(int a, int b)
{
        return a+b;
}
```

```elm
// main.cpp
#include <gtest/gtest.h>
#include "util.h"

TEST(Sum, SumTwoNumber) {
        EXPECT_EQ(sum(2, 3), 6);
}

int main(int argc, char **argv) {
        ::testing::InitGoogleTest(&argc, argv);
        return RUN_ALL_TESTS();
}
```

```elm
g++ -o test main.cpp -lgtest -lpthread
```

위와 같이 컴파일을 한뒤 test를 실행하면 `Sum.SumTwoNumber` 항목에 대한 테스트가 FAILED 로나옵니다.  2와 3을 더하면 5가 나와야 하는데 기댓값으로 6을 입력하였기 때문에 FAILED로 나오네요.

`EXPECT_EQ(sum(2, 3), 5)` 로 수정을 하면 OK로 출력이 됩니다.

간편하게 Test코드를 빌드하고 컴파일하기 위해 작업 디렉토리에 CMakeLists.txt를 만듭니다.

```elm
cmake_minimum_required(VERSION 2.4)
set(CMAKE_CXX_STANDARD 14)

project(sum_test)

file(GLOB SRC
        "main.cpp"
        )

include_directories($ENV{HOME}/gtest/build/_deps/googletest-src/googlemock/include $ENV{HOME}/gtest/build/_deps/googletest-src/googletest/include)
link_directories($ENV{HOME}/gtest/build/lib)

add_executable(${PROJECT_NAME} ${SRC})
add_custom_target(test
        COMMAND ${PROJECT_NAME}
        )

target_link_libraries(${PROJECT_NAME} PUBLIC gtest pthread)
```

```elm
mkdir build && cd build && cmake .. && make test
```
