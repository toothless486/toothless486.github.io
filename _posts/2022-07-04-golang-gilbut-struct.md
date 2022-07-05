---
title:  "구조체"
excerpt: "구조체, 리시버, 임베딩, 인터페이스 사용"

toc: true
toc_sticky: true

published: true

categories:
  - Books
tags:
  - 가장 빨리 만나는 Go언어
sitemap:
  changefreq: daily
  priority: 0.8
---

# 1. 구조체
```go
package main

import "fmt"

type Person struct {
    age int
    name string
}

func main() {
    var person Person = Person{20, "Lisa"}
    fmt.Printf("%d : %s", person.age, person.name)    
}
```

# 2. 리시버
리시버를 통해서 현재 인스턴스의 값에 접근할 수 있다.
Person 구조체에 GetName 함수를 연결하여 사용한다.
```go
package main

import "fmt"

type Person struct {
    age int
    name string
}
func (person *Person) GetName() string {
    return person.name
}

func main() {
    person := Person{20, "Lisa"}
    fmt.Println(person.GetName())
}
```

# 3. 임베딩
임베딩은 클래스의 상속과 같은 효과를 낼 수 있다.
## 3.1. Has-a
```go
package main

import "fmt"

type Person struct {
    age int
    name string
}
func (person *Person) SayHello() {
    fmt.Println("Hi, I'm Person")
}

type Student struct {
    p Person  // Has-a 관계 (Student가 Person을 가지고 있음)
    grade int
}

func main() {
    var s Student
    s.p.SayHello()
}
```

## 3.2. Is-a
```go
package main

import "fmt"

type Person struct {
    age int
    name string
}
func (person *Person) SayHello() {
    fmt.Println("Hi, I'm Person")
}

type Student struct {
    Person  // Is-a 관계 (Student가 Person 타입을 포함)
    grade int
}
func (student *Student) SayHello() { // 오버라이딩
    fmt.Println("Hi, I'm Student")
}

func main() {
    var s Student

    s.Person.SayHello() // Hi, I'm Person
    s.SayHello()        // Hi, I'm Student
}
```

# 4. 인터페이스
덕 타이핑 : 실제 타입은 상관하지 않고 구현된 메서드로만 타입을 판단
```go
type Quacker interface {
    quack()
    feathers()
}

type Duck struct {
}
func (d Duck) quack() {
    fmt.Println("꽥 ~")
}
func (d Duck) feathers() {
    fmt.Println("하얗고 귀여워")
}

type Person struct {
}
func (p Person) quack() {
    fmt.Println("사람이 꽥")
}
func (p Person) feathers() {
    fmt.Println("사람이지만 오리처럼")
}

func main() {
    var donald Duck
    var lisa Person
    var q Quacker

    q = donald
    q.quack()   // 꽥
    q.feathers  // 하얗고 귀여워

    q = lisa
    q.quack()   // 사람이 꽥
    q.feathers  // 사람이지만 오리처럼
}

```