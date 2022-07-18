---
title:  "간단한 문법 사용"
excerpt: "golang을 시작하기 위한 간단한 문법을 사용하여 예시문제 해결"
classes:  wide
categories:
  - Books
tags:
  - 가장 빨리 만나는 Go언어
---

# 1. FizzBuzz
```go
package main

import "fmt"

func main() {
	for i:=1; i<=100; i++ {
		if (i%3 == 0) && (i%5 == 0) {
			fmt.Println("FizzBuzz")
		}
		if i%3 == 0 {
			fmt.Println("Fizz")
		}
		if i%5 == 0 {
			fmt.Println("Buzz")
		}
	}
}
```

# 2. Bottles
```go
package main

import "fmt"

func main() {

	for i:= 99; i>=1; i-- {
		var bottles string = "bottle"
		if i > 2 {
			bottles = "bottles"
		}

		if i <= 98 {
			fmt.Printf("Take one donw, pass it around, %d %s of beer on the wall.\n", i, bottles)
		}
		fmt.Printf("%d %s of beer on the wall, %d %s of beer.\n", i, bottles, i, bottles)
	}

	fmt.Println("Take one down, ,,, ,, ")
}
```
