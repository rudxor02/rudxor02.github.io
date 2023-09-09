---
layout: post
type: tags
title: "[Python] Inheritance with wrapped __init__ and type hint"
date: 2023-09-09 15:00:00 +0900
comments: true
toc: true
tags: python
---

## Mypy

우선 vscode가 python 타입을 어떻게 아는지부터 알아야 한다. 기본적으로 타입을 명시해야 하는 언어가 아니다 보니 외부 패키지를 사용하는데, vscode를 깔고 python 확장을 설치하면 자동으로 타입 검사를 해주는 Pylance가 설치된다.

![Untitled](/assets/images/post/2023-09-09-inheritance-with-wrapped-init-and-type-hint-20230909153116.png)

Pylance는 pyright라는 패키지를 쓰고 이와 비슷한 것으로 Mypy가 있는데, 주로 이 둘이 비교되고 vscode는 같은 pyright를 사용한다. [[참고1]](https://www.infoworld.com/article/3575079/4-python-type-checkers-to-keep-your-code-clean.html)

Mypy에서는 앞서 보았던 type annotation에 사용되는 데코레이터의 종류를 `dataclass`, `dataclass_transform`으로 제한해 놓았다. [[참고2]](https://github.com/python/mypy/blob/e0f16ed027ff7bcd30d9a4dffebe95c101447b8d/mypy/plugins/dataclasses.py#L23C51-L23C51) Mypy에서 type annotation에 사용될 수 있는 데코레이터가 저 2개로 제한돼있다는 건 알겠는데, 왜 vscode에서도 다른 데코레이터는 동작을 안 할까? 바로 pyright에서 Mypy의 plugin을 갖다 쓰기 때문이다. [[참고3]](https://github.com/microsoft/pyright/blob/main/docs/mypy-comparison.md#plugins)

## 좀 더 고쳐보면

그럼 이미 만들어진 거 쓰는 수밖에 없다… 그냥 `dataclass_transform` 데코레이터로 좀 더 코드를 고쳐보면

```python
# 4.py
from abc import ABC, ABCMeta
from functools import wraps
from typing import Any, Callable, Optional, dataclass_transform

@dataclass_transform()
class NewModelMetaclass(ABCMeta):
    ...

def change_vehicle_child_init_func(init: Callable[..., Any]):
    @wraps(init)
    def new_init(
        self: Vehicle,
        *args: Any,
        name: str,
        engine: Optional[Engine] = None,
        **kwargs: Any,
    ):
        self.name = name
        if engine is None:
            self.engine = Engine(name=f"{name}_engine")
        else:
            self.engine = engine

    return new_init

def change_new_base_model_child_init_func(init: Callable[..., Any]):
    @wraps(init)
    def new_init(self: NewBaseModel, *args: Any, **kwargs: Any):
        self.__dict__.update(kwargs)

    return new_init

class NewBaseModel(metaclass=NewModelMetaclass):
    def __init_subclass__(cls) -> None:
        cls.__init__ = change_new_base_model_child_init_func(cls.__init__)  # type: ignore

    def __repr__(self) -> str:
        return f"{self.__class__.__name__}({self.__dict__})"

class Engine(NewBaseModel):
    name: str

mock_engine = Engine(name="mock_engine")

print(mock_engine)
# Engine({'name': 'mock_engine'})

class Vehicle(NewBaseModel, ABC):
    name: str
    engine: Engine = mock_engine

    def __init_subclass__(cls) -> None:
        cls.__init__ = change_vehicle_child_init_func(cls.__init__)  # type: ignore

class Car(Vehicle):
    pass

class Ship(Vehicle):
    pass

car = Car(name="car")
ship = Ship(name="ship")
print(car)
# Car({'name': 'car', 'engine': Engine({'name': 'car_engine'})})
print(ship)
# Ship({'name': 'ship', 'engine': Engine({'name': 'ship_engine'})})
```

🤔 pydantic에서의 경우처럼 NewModelMetaclass란 metaclass를 만들어 그걸 NewBaseModel을 만들어내는 metaclass로 등록을 했다. 그럼 mock_engine이 만들어질 때 `change_new_base_model_child_init_func` 로 바뀐 `__init__`이 불려져서 name을 ‘mock_engine’으로 설정해 준다. Vehicle을 상속받는 Car와 Ship에서도 `change_vehicle_child_init_func`로 바뀐 `__init__`으로 engine까지 잘 만들어지는 걸 볼 수 있다. 음 그런데…

1. Engine과 (Car, Ship)은 같은 NewBaseModel을 물려받았음에도 init이 서로 다르다. 그럼 Car, Ship의 `__init__`은`change_new_base_model_child_init_func` 로 바뀐 init도 call해야 자연스럽지 않을까?
2. Engine, Car, Ship 모두 name을 갖고 있는데 Engine의 경우는 상관없지만, Car와 Ship에서는 `__init__`함수의 시그니처에 name을 넣지 않으면 실행할 때 unexpected keyword라며 에러가 난다. name을 분리하고 싶다. 1번에서 `__init__`을 불러줄 때 name을 넘겨줄 수 있을 것 같다.

그럼 마지막으로 한번만 더 고쳐보면

```python
# 5.py
from abc import ABC, ABCMeta
from functools import wraps
from typing import Any, Callable, Optional, dataclass_transform

@dataclass_transform()
class NewModelMetaclass(ABCMeta):
    ...

def change_vehicle_child_init_func(init: Callable[..., Any]):
    @wraps(init)
    def new_init(
        self: Vehicle,
        *args: Any,
        engine: Optional[Engine] = None,
        **kwargs: Any,
    ):
        super(Vehicle, self).__init__(*args, **kwargs)
        if engine is None:
            self.engine = Engine(name=f"{self.name}_engine")
        else:
            self.engine = engine

    return new_init

def change_new_base_model_child_init_func(init: Callable[..., Any]):
    @wraps(init)
    def new_init(self: NewBaseModel, *args: Any, **kwargs: Any):
        self.__dict__.update(kwargs)

    return new_init

class NewBaseModel(metaclass=NewModelMetaclass):
    def __init_subclass__(cls) -> None:
        cls.__init__ = change_new_base_model_child_init_func(cls.__init__)  # type: ignore

    def __repr__(self) -> str:
        return f"{self.__class__.__name__}({self.__dict__})"

class HaveName(NewBaseModel, ABC):
    name: str

class Engine(HaveName):
    pass

mock_engine = Engine(name="mock_engine")

print(mock_engine)
# Engine({'name': 'mock_engine'})

class Vehicle(HaveName, ABC):
    engine: Engine = mock_engine

    def __init_subclass__(cls) -> None:
        cls.__init__ = change_vehicle_child_init_func(cls.__init__)  # type: ignore

class Car(Vehicle):
    pass

class Ship(Vehicle):
    pass

car = Car(name="car")
ship = Ship(name="ship")
print(car)
# Car({'name': 'car', 'engine': Engine({'name': 'car_engine'})})
print(ship)
# Ship({'name': 'ship', 'engine': Engine({'name': 'ship_engine'})})
```

1. HaveName이란 name을 가지는 클래스로 분리되었고
2. Vehicle을 상속받아서 바뀌는 `__init__`도 부모 클래스의 `__init__`을 call한다.

그럼 이제 Vehicle을 상속받으면 알아서 engine도 name 따라서 구성되고 type annotation도 되니까 다 된걸까? 원하는 기능은 다 완성됐지만 한 가지 아쉬운 건 validation이 없다. 예를 들어 Car 인스턴스를 생성할 때 name2를 주면 type error는 나지만 실행은 잘 되고 Car는 `Car({'name': 'car', 'name2': 'car2', 'engine': Engine({'name': 'car_engine'})})` 와 같이 만들어진다.

![Untitled](/assets/images/post/2023-09-09-inheritance-with-wrapped-init-and-type-hint-20230909153159.png)

`change_new_base_model_child_init_func`에서 바꿔준 `__init__`이 아무 kwargs나 다 넣어주기 때문이다. 이렇게 하려면 선언한 property 목록을 가져와서 validation을 해야 한다. 이건 언젠가 이 기능을 구현하고싶어질 미래의 나에게 맡기겠다 ㅎㅎ..

## References

1. [4 python type checkers](https://www.infoworld.com/article/3575079/4-python-type-checkers-to-keep-your-code-clean.html)
2. [mypy dataclass plugin github source code](https://github.com/python/mypy/blob/e0f16ed027ff7bcd30d9a4dffebe95c101447b8d/mypy/plugins/dataclasses.py#L23C51-L23C51)
3. [pyright using mypy plugin](https://github.com/microsoft/pyright/blob/main/docs/mypy-comparison.md#plugins)
4. [rudxor02/change-child-init-py github repository](https://github.com/rudxor02/change-child-init-py)
