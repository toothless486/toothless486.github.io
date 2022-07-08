---
title:  "Primer"

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

# Primer

### Assertions

ASSERTION_* 을 사용했을 때 실패가 발생하면, 현재 테스트 중인 함수를 중단시키고 실패 메시지를 출력합니다. EXPECT_* 을 사용했을 때 실패가 발생하면, 현재 테스트 중인 함수를 중단하지 않습니다. 실패 메시지를 custom 할 수 있는데, 사용법은 `<<` 연산자를 사용해주면 됩니다.

```elm
ASSERT_EQ(x.size(), y.size()) << "Vectors x and y are of unequal length";

for (int i = 0; i < x.size(); ++i) {
  EXPECT_EQ(x[i], y[i]) << "Vectors x and y differ at index " << i;
}
```

### Simple Tests

TEST()의 첫 번째 argument는 test suite의 이름입니다. 두번째 argument는 test suite 내의 test의 이름입니다. 해당 argument에는 underscores (_)가 포함되면 안됩니다.

test의 풀네임은 testsuite와 각각의 test name으로 구성이 됩니다.

다음의 test suite은 FactorialTest으로 정해져 있고, test name은 각각 HandlesZeroInput, HandlesPositiveInput 으로 구성되어 있습니다.

```elm
// Tests factorial of 0.
TEST(FactorialTest, HandlesZeroInput) {
  EXPECT_EQ(Factorial(0), 1);
}

// Tests factorial of positive numbers.
TEST(FactorialTest, HandlesPositiveInput) {
  EXPECT_EQ(Factorial(1), 1);
  EXPECT_EQ(Factorial(2), 2);
  EXPECT_EQ(Factorial(3), 6);
  EXPECT_EQ(Factorial(8), 40320);
}
```

### ****Test Fixtures: Using the Same Data Configuration for Multiple Tests****

같은 test data를 이용하여 다양한 테스트 케이스를 만들고 싶을 때, test fixture을 사용합니다.

예시로 Qeue의 함수를 테스트 하도록 하겠습니다.

```elm
template <typename E>  // E is the element type.
class Queue {
 public:
  Queue();
  void Enqueue(const E& element);
  E* Dequeue();  // Returns NULL if the queue is empty.
  size_t size() const;
  ...
};
```

첫 번째로, fixture class 를 정의합니다. Convention에 따르면, Queue를 테스트하는 class라면 QueueTest라고 class이름을 정합니다. `::testing::Test` 를 상속받고 body는 `protected` 로  시작합니다. SetUp()을 통해서 test data를 초기화합니다. 그리고 테스트가 끝날 때 TearDown()이 실행되어 test data를 정리합니다. SetUp()에서 n 변수를 할당하고 TearDown에서 `delete` 해주었습니다.

```elm
class QueueTest : public ::testing::Test {
 protected:
  void SetUp() override {
			n = new int;
     q1_.Enqueue(1);
     q2_.Enqueue(2);
     q2_.Enqueue(3);
  }

	// void TearDown() override
	{
			if(n) delete n;
	}

  

  Queue<int> q0_;
  Queue<int> q1_;
  Queue<int> q2_;
	int *n = nullptr;
};
```

가각의 teest는 TEST_F() 로 선언합니다. googletest는 TEST_F의 runtime 시간 동안 test fixture를 SetUp()을 통해서 생성합니다. 그리고 test fixture를 delete 하면서 TearDown()을 호출합니다. 

TEST()의 첫 번째 매개변수는 test suite 의 이름이었습니다. TEST_F()의 첫 번째 매개변수는 test fixture class 의 이름이 들어가야 합니다. 두 번째 매개변수는 마찬가지로 test의 이름이 들어갑니다.

```elm
TEST_F(QueueTest, IsEmptyInitially) {
  EXPECT_EQ(q0_.size(), 0);
}

TEST_F(QueueTest, DequeueWorks) {
int* n= q0_.Dequeue();
  EXPECT_EQ(n, nullptr);

  n= q1_.Dequeue();
  ASSERT_NE(n, nullptr);
  EXPECT_EQ(*n, 1);
  EXPECT_EQ(q1_.size(), 0);
delete n;

  n= q2_.Dequeue();
  ASSERT_NE(n, nullptr);
  EXPECT_EQ(*n, 2);
  EXPECT_EQ(q2_.size(), 1);
delete n;
}
```

### Known Limitations

> Google Test is designed to be thread-safe. The implementation is thread-safe on systems where the `pthreads` library is available. It is currently *unsafe* to use Google Test assertions from two threads concurrently on other systems (e.g. Windows). In most tests this is not an issue as usually the assertions are done in the main thread. If you want to help, you can volunteer to implement the necessary synchronization primitives in `gtest-port.h` for your platform.
>