# AI测评-概要设计-每道题的测评结果和每一轮的TestResult关联

## 1 libadsusertestsys工程的libadsusertestsys/orm/user_test_sys_orm.py
表：TestQuestionAndAnswer，SpeakingTestQuestionAndAnswer
```
    test_result_id = Column(Integer) #=TestResult的主键
```

## 2 libadsusertestsys工程的libadsusertestsys/dao/evaluation_dao.py的save_test_result函数，改成返回插入记录的主键
```
	db.session.add(test_result)
	db.session.commit()
	ret = test_result.id
	return ret
```

## 3 libadsusertestsys工程的libadsusertestsys/service/evaluation_service.py的reset函数
	把存TestResult挪到存TestQuestionAndAnswer/SpeakingTestQuestionAndAnswer的前面
	保存TestResult返回的id记录到变量
	batch_create_user_question_and_answer增加TestResult返回的id记录的参数

## 4 libadsusertestsys工程的libadsusertestsys/dao/evaluation_dao.py的batch_create_user_question_and_answer函数
	batch_create_user_question_and_answer增加TestResult返回的id记录的参数
	b，TestQuestionAndAnswer，SpeakingTestQuestionAndAnswer的对象构造中增加test_result_id
