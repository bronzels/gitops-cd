# AI测评-概要设计-estimator按照level而不是grade加载

## 1 grammar/reading/listening/speaking/vocabularysize的evaluation/service/evaluation_service.py的generate_estimators函数
以speaking为例
```
    for acadsoc_level in range(7):
        speaking_estimators[acadsoc_level] = SpeakingPlacementTestEstimator(
            begin_level=max(acadsoc_level, 0),
            max_level=min(acadsoc_level + 1, 6),
            min_question_request=7,
            max_question_request=10,
            max_question_request_bucket=8,
        )
```

## 2 libadsusertestsys工程的libadsusertestsys/service/evaluation_service.py的_get_estimator函数
```
        user_grade = user.grade
        if user_grade is None:
            raise MyError(ERR_CODE_GRADE_NOT_SET)
        acadsoc_level = grade.get_acadsoc_level()    
        mylog.logger.info('acadsoc_level:{}'.format(acadsoc_level))
        evaluation_estimator = evaluation_estimators[test_orientation.value][acadsoc_level]
        return evaluation_estimator
```
