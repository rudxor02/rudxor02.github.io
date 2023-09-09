---
layout: post
type: tags
title: "[Python] Inheritance with wrapped __init__"
date: 2023-09-09 15:00:00 +0900
comments: true
toc: true
tags: python
---

*원래 python을 혐오하는 사람 중 한 명이었으나 회사에서 3개월 넘게 쓰면서 ~~미운~~정이 들기도 했고 이 주제를 공부하면서 강력한 OOP 언어가 될 수 있다는 것을 알게 되었다. 우선 찾아보게 된 배경 + 앞으로 나올 글의 흐름은 다음과 같다.*

1. 회사에서 llm 서비스를 개발하게 되었음
2. Haystack이 llm 서비스를 만드는 오픈소스 프로젝트인걸 개발하다 얼핏 알게 되었고, 프로젝트가 어느 정도 마무리 되고 다른 곳은 어떻게 하나 참고할 겸 Haystack 레포를 살펴봤음
3. `__init_subclass__`라는 아주아주 이상한 매직 메소드를 처음 알게 됐음
4. 와 부모 클래스에서 자식 클래스의 `__init__`을 바꿔끼울 수 있다니 정말 신기하다…
5. 근데 `pydantic.BaseModel`에서는 class variable을 정의하는 것만으로도 `__init__` 함수의 signature를 자동으로 정의할 수 있는데 이건 어떻게 하는거지?

## Signature란

참고로 signature란 함수의 모양을 말한다. (`inspect.signature`는 함수의 signature를 쉽게 확인할 수 있는 함수)

```python
# signature.py
from inspect import signature

def add(x: int, y: int) -> int:
 return x + y

print(signature(add))

# (x: int, y: int) -> int
```

## \_\_init_subclass\_\_

Haystack이란 2018년부터 시작된 Deepset이라는 nlp 회사에서 만든 오픈소스 프로젝트 이름이다. 이 글에서 Haystack이 무슨 프로덕트인지 중요한 건 아니고 아무튼 기능 참고하려고 레포를 살펴보다가 자식클래스의 `__init__`함수를 wrapping하는 당시 내 얕은 지식로서는 충격적인 코드를 보았다.

```python
# haystack/nodes/base.py
from abc import ABC
from functools import wraps

def exportable_to_yaml(init_func):
    @wraps(init_func)
    def wrapper_exportable_to_yaml(self, *args, **kwargs):
        ...
        init_func(self, *args, **kwargs)

    return wrapper_exportable_to_yaml

class BaseComponent(ABC):
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        ...
        cls.__init__ = exportable_to_yaml(cls.__init__)
        ...
```

이때 `__init_subclass__`라는 매직 메소드를 보게 됐는데, 기존에 알고 있던 `__init__` 메소드처럼 instantiate될 때 작동하는 게 아니라 자식 클래스에서 ‘상속받을 때’ 작동한다고 한다. 이때 `__init__` 함수를 바꿔끼우는 게 매우 흥미로웠다. 왜냐하면 필자가 알고 있던 보통의 상속의 의미라 하면 메소드를 예를 들면 abstract method로 signature 혹은 이름만 정의하고 구현은 자식 클래스에서 하는 게 자연스러운 것이고, abstract method 말고 부모 클래스에서 정의한 메소드를 이어서 하고 싶다면 자식 클래스에서 이렇게 정의해줘야 하기 때문이다.

```python
class Parent:
    def __init__(self, *args, **kwargs):
        pass

class Child(Parent):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        ...
```

다시 말해서 기존에 알고 있던 지식으로는 부모 클래스에서 자식 클래스가 `__init__`할 때 공통으로 뭔가를 했으면 좋겠을 때 적용할 수 있는 방법으로는 위와 같이 자식 클래스에서 `__init__`함수를 직접 구현해서 super를 통해 부모 클래스의 `__init__`을 직접 부르는 방법밖에 없었던 것이었는데, Haystack에서 봤던 방식은 부모 클래스에서 `__init_subclass__`를 정의해놓으면 자식 클래스에서 상속할 때 `__init__` 함수를 wrapping한 함수로 자동으로 바꿔끼우는 방식인 것이다. 이를 통하면 자식 클래스에서 일일이 부르는 것보다 부모 클래스 한 곳에서만 공통으로 하고 싶은 작업을 정의해서 코드 재사용률이 낮아지지 않을까…?

사실 코드 재사용률 측면에서보다는 클래스를 상속받아 구현하는 사람(미래의 나 혹은 동료 개발자)의 실수를 줄인다는 측면에서 중요하다고 생각한다. 개인적으로 몇 년 전에 개발을 처음 시작할 때 typing이 거의 없던 순수 node js만으로 시작해서 개고생을 하다 최근 입사하고 나서 typescript를 처음 쓰고 *와 진짜 편하다…* 라고 느꼈던 경험이 코드를 짜는 것에 있어 typing, 더 넓게는 __개발자가 실수할 수 있는 부분을 시스템적으로 줄여나가는 것이 매우 중요하다__고 느끼는 계기가 되었다. 이 경우도 비슷한 상황이라고 느껴서 더 살펴봐야겠다고 생각이 들었다.

### use of \_\_init_subclass\_\_

그럼 위에서 저 생각을 어떤 불편함을 느껴서 중요하다고 생각했는지, 그리고 이를 `__init_subclass__`로 어떻게 해결하려 했는지 적어보겠다.

```python
# 1.py
from pydantic import BaseModel

class Engine(BaseModel):
    name: str

class Car(BaseModel):
    name: str
    engine: Engine

class Ship(BaseModel):
    name: str
    engine: Engine

car_engine = Engine(name="car_engine")
car = Car(name="car", engine=car_engine)

ship_engine = Engine(name="ship_engine")
ship = Ship(name="ship", engine=ship_engine)
```

🤔🤔🤔🤔 뭔가 되게 불편해 보인다…

1. name을 공통적인 속성으로 가지는데 이를 하나의 HaveName 클래스로 묶고싶다
2. 그런데 그러기엔 Engine이 Car, Ship에 들어가고 그 안에서 prefix `car_`와 `ship_`이 붙는다
3. BaseModel로 하기에는 무리가 있다

개발자가 name convention을 모른다면 engine을 생성하고 Car나 Ship에게 넣어줄 때, 즉 instantiate할 때 잘 못 지킬 수 있다. 왜냐하면 engine을 외부에서 주입받기 때문이다. 그럼 주입받지 말고 이렇게 하는 건 어떨까?

```python
# 2.py
from pydantic import BaseModel

class Engine(BaseModel):
    name: str

class Car:
    def __init__(self, name: str) -> None:
        self.name = name
        self.engine = Engine(name=f"{self.__class__.__name__}_engine")

class Ship:
    def __init__(self, name: str) -> None:
        self.name = name
        self.engine = Engine(name=f"{self.__class__.__name__}_engine")

car = Car(name="car")
ship = Ship(name="ship")
```

🤔🤔🤔 한결 나은 것 같다. 하지만

1. Car와 Ship의 `__init__`이 완전 똑같다. 이를 어떻게 해결할 수는 없을까?
2. Car나 Ship 말고 다른 클래스를 생성할 때 저 `__init__` 메소드를 똑같이 구현해야 한다. 개발자가 실수하지 않을까? 그리고 혹시 `__init__`메소드를 고칠 일이 있다면 모든 `__init__` 메소드를 다같이, 똑같이 고쳐야 한다
3. 여전히 name은 해결되지 않은 상태

그럼 `__init_subclass__`를 사용해 보면 어떨까?

```python
# 3.py
from abc import ABC
from functools import wraps
from typing import Any, Callable

from pydantic import BaseModel

def change_init_func(init: Callable[..., Any]):
    @wraps(init)
    def new_init(self: Vehicle, *args: Any, name: str, **kwargs: Any):
        self.name = name
        self.engine = Engine(name=f"{self.__class__.__name__}_engine")
        init(self, *args, name, **kwargs)

    return new_init

class Engine(BaseModel):
    name: str

class Vehicle(ABC):
    name: str
    engine: Engine

    def __init_subclass__(cls) -> None:
        cls.__init__ = change_init_func(cls.__init__)  # type: ignore

class Car(Vehicle):
    def __init__(self, name: str) -> None:
        pass

class Ship(Vehicle):
    def __init__(self, name: str) -> None:
        pass

car = Car(name="car")
ship = Ship(name="ship")
```

🤔🤔 아… 뭔가 로직은 부모 클래스로 다 옮겨지긴 했는데…

1. name이 애매하게 묶였다. Engine까지 묶을 수 있는 방법이 없을까?
    - 근데 name으로 묶자니 Car랑 Ship은 Engine을 갖고 있고 각각이 받은 name으로부터 Engine을 실행해야 한다.
2. `__init__` 메소드의 signature가 안옮겨졌다.
    - signature까지 옮기면 이렇게 뜸 (typing이 매우 불편하다). 그러나 실행은 된다. `__init__`이 동적으로 할당돼서 signature까지 typing할 수 없나보다.
        ![Untitled](/assets/images/post/2023-09-09-inheritance-with-wrapped-init-20230909151738.png)

자 원래 원하는 대로 로직은 다 옮겨졌지만, 아직도 Vehicle의 `__init__` signature를 일일이 Car와 Ship에 똑같이 적어줘야 해서 뭔가 불편하다… 좀 더 뇌절해서 여기서 *아 signature까지 자식 클래스에서 자동으로 정의하고 싶은데… 그리고 자식 클래스에서도 BaseModel을 써서 instantiate할 때 name이랑 같이 인자로 전달할 수 있다면 얼마나 좋을까? 어 그런데 생각해보니 BaseModel에서는 `__init__`을 구현하지 않아도 자동으로 type hint를 제공하는데… 여기서 어떻게 하는 지 보고 그 부분만 가져와 고친다면 해결되지 않을까?* 라는 생각을 더 해봤다.

## Type hint from Pydantic?

class variable 형태로 프로퍼티를 정해주면 type hint를 어떻게 제공하는지 그 방법이 도저히 감이 안와서 무작정 pydantic 레포를 뒤져봤다. 제일 의심가는 프로퍼티로는 `__signature__`였다. pydantic에서 이러한 코드를 봤기 때문이다. 뭔가 함수 이름이랑 변수이름 등을 봤을때 제일 그럴듯해 보였다.

```python
# pydantic/v1/main.py#L283
cls.__signature__ = ClassAttribute('__signature__', generate_model_signature(cls.__init__, fields, config))
```

그러나 `__signature__`는 `__dict__`, `__doc__` 처럼 원래 클래스의 일반적인 속성이 아니기도 하고 하루종일 삽질한 결과 실제 type hint랑은 상관없다는 결론이 나왔다. (정확한 건 아니지만 `inspect.signature`를 했을 때 어떻게 보일 지 custom하게 설정하는 속성인 것 같다) 결국 레포 클론받아서 의심가는 부분 주석처리 해가며 확인해본 결과 이 부분임을 찾아냈다

```python
# pydantic/v1/main.py#L120
@dataclass_transform(kw_only_default=True, field_specifiers=(Field,))
class ModelMetaclass(ABCMeta):
 ...

# pydantic/v1/main.py#310
class BaseModel(Representation, metaclass=ModelMetaclass):
 ...
```

ModelMetaClass에 붙어있는 decorator 한 줄이 원인이었는데, 의외로 간단해서 놀랐고 metaclass도 처음 알았다. [[참고1]](https://alphahackerhan.tistory.com/34)

### dataclass_transform decorator

저 dataclass_transform이라는 데코레이터는 python 기본 라이브러리인 typing 안에 있었다. 사실 필자가 그동안 pydantic을 써온 가장 큰 이유였던 type hint 기능이 되게 간단하게 구현할 수 있다는 걸 알아서 좀 놀랐었다. [[참고2]](https://peps.python.org/pep-0681/#id3)

decorator 코드는 별 거 없는데, 주석 제외하면 이게 전부다.

```python
# python typing library code
def dataclass_transform(
    *,
    eq_default: bool = True,
    order_default: bool = False,
    kw_only_default: bool = False,
    field_specifiers: tuple[type[Any] | Callable[..., Any], ...] = (),
    **kwargs: Any,
) -> Callable[[T], T]:
    def decorator(cls_or_fn):
        cls_or_fn.__dataclass_transform__ = {
            "eq_default": eq_default,
            "order_default": order_default,
            "kw_only_default": kw_only_default,
            "field_specifiers": field_specifiers,
            "kwargs": kwargs,
        }
        return cls_or_fn
    return decorator
```

> At runtime, the `dataclass_transform` decorator’s only effect is to set an attribute named `__dataclass_transform__` on the decorated function or class to support introspection.

문서에 따르면 `__dataclass_transform__`이 type hint에 쓰이는 프로퍼티임을 유추할 수 있다. *아 그럼 저 데코레이터에서처럼 `__dataclass_transform__` 건들면 타이핑을 마음대로 할 수 있겠지? 이거 직접 만들어서 써봐야지~* 하고 코드를 써보면?

```python
# my_decorator_vs_standard_decorator.py
def my():
    from typing import Any, Callable, TypeVar

    T = TypeVar("T")

    def dataclass_transform(
        *,
        eq_default: bool = True,
        order_default: bool = False,
        kw_only_default: bool = False,
        field_specifiers: tuple[type[Any] | Callable[..., Any], ...] = (),
        **kwargs: Any,
    ) -> Callable[[T], T]:
        def decorator(cls_or_fn):
            cls_or_fn.__dataclass_transform__ = {
                "eq_default": eq_default,
                "order_default": order_default,
                "kw_only_default": kw_only_default,
                "field_specifiers": field_specifiers,
                "kwargs": kwargs,
            }
            return cls_or_fn

        return decorator

    @dataclass_transform()
    class NewBaseModel:
        pass

    class A(NewBaseModel):
        a: str

    # type error X - decorator not working
    A()
    # type error O - decorator not working
    A(a="a")

def standard():
    from typing import dataclass_transform

    @dataclass_transform()
    class NewBaseModel:
        pass

    class A(NewBaseModel):
        a: str

    # type error O - decorator working
    A()
    # type error X - decorator working
    A(a="a")
```

![Untitled](/assets/images/post/2023-09-09-inheritance-with-wrapped-init-20230909151854.png)

*어… 코드 똑같은데 내꺼는 왜 안되지…* 라는 의문 속에 몇일을 인터넷만 뒤졌던 것 같다. 글이 너무 길어졌으니 [다음 포스팅](/knowledge/2023/09/09/inheritance-with-wrapped-init-and-type-hint)에서 그 이유를 알아보도록 하겠다

## References

1. [metaclass 이해하는 데 도움되는 블로그 포스팅](https://alphahackerhan.tistory.com/34)
2. [PEP-681](https://peps.python.org/pep-0681/)
3. [Haystack github repository](https://github.com/deepset-ai/haystack)
4. [Pydantic github repository](https://github.com/pydantic/pydantic)
5. [Python typing library github source code](https://github.com/python/cpython/blob/main/Lib/typing.py)
6. [rudxor02/change-child-init-py github repository](https://github.com/rudxor02/change-child-init-py)
