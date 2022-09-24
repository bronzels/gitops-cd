# AI测评-概要设计-test_orientation字符串改整数枚举值

## 1 libadsusertestsys工程的libadsusertestsys/entity/evaluation_entity.py类TestOrientation
```
TestOrientation = {
    placement_test = 0,
    unit_test = 1

    @classmethod
    def get_variables(cls):
        return [getattr(cls, attr) for attr in dir(cls) if not callable(getattr(cls, attr)) and not attr.startswith("__")]
}
```

## 2 libadsusertestsys工程的libadsusertestsys/orm/user_test_sys_orm.py的TestResult类
    test_orientation = Column(Integer, nullable=False)

## 3 手工修改数据库表TestResult，修改test_orientation数据类型，update值为0

## 4 app工程的evaluation/service/evaluation_service.py的generate_estimators函数
```
    estimators[TestOrientation.placement_test.value] = speaking_estimators
```

## 4 libadsusertestsys工程的libadsusertestsys/service/evaluation_service.py
### _get_test_orientation_json函数test_orientation_str = "placement-test"改成test_orientation_str = "placement_test"
### _get_test_orientation_form函数test_orientation_str = "placement-test"改成test_orientation_str = "placement_test"
### get_question/_update_answer/reset的test_orientation = TestOrientation(test_orientation_str)，都改写成
```
                if test_orientation_str is None:
                    raise MyError(ERR_CODE_INF_INPUT_ABSENT, 'test_orientation')
                test_orientation = TestOrientation.__dict__.get(test_orientation_str) if test_orientation_str is not None else None
                if test_orientation is None:
                    raise MyError(ERR_CODE_INF_INPUT_INVALID, 'test_orientation:{}'.format(test_orientation_str))
```
### get_question函数，test_orientation.value替换成test_orientation_str
### reset函数，test_orientation.value替换成test_orientation_str
