# AI测评-概要设计-流的方式上传口语录音大文件（上传wav文件）

## 1 libadsusertestsys工程的libadsusertestsys/orm/user_test_sys_orm.py
只列出部分修改字段
```
class TestResult(db.Model):
"""用户测评结果记录表"""
test_orientation = Column(String(50), nullable=False)
score = Column(Float, nullable=True)
level = Column(String(20), nullable=True)
acadsoc_level = Column(Integer, nullable=True)
rate_of_beaten = Column(Float, nullable=True)
```

## 2 libadsusertestsys工程的libadsusertestsys/entity/evaluation_entity.py
### 修改类
```
class EntityTestQuestionAndAnswer:
    def __init__(self, question, answer=None, m_correct=None):
        self.question = question
        self.answer = answer
        self.m_correct = m_correct
```

## 3 libadsusertestsys工程新增python包unfinished
### 新增文件entity
```
class UnfinishedTest:
    def __init__(self, test_result_id, questions_len, is_last_question_answered):
        self.test_result_id = test_result_id
        self.questions_len = questions_len
        self.is_last_question_answered = is_last_question_answered       
```

### 新增文件dao
```
from sqlalchemy import or_
from libadsusertestsys.entity.evaluation_entity import ConfusionsType, EntityTestQuestionAndAnswer
def get_test_unfinished(orm_entity, test_type, user_id, estimator):
    test_results_unfinished = TestResult.query \
        .filter(test_user_id=user_id) \
        .filter(TestResult.score.is_(None)) \
        .order_by(TestResult.id.desc()).limit(1)
    if test_results_unfinished is None:
        return None

    test_result_unfinished = test_results_unfinished[0]

    if test_type == TestType.SPEAKING_EVALUATION:
        test_question_and_answers_unfinished = SpeakingTestQuestionAndAnswer.query\
        .filter(test_result_id = test_result_unfinished.id).all()
    else:
        test_question_and_answers_unfinished = TestQuestionAndAnswer.query\
            .filter(test_result_id=test_result_unfinished.id).all()

    if test_question_and_answers_unfinished is None:
        mylog.logger.error(
            'no unfinished questions per test result, id:{}, test_user_id:{}, test_orientation:{}, create_time:{}',
            test_result_unfinished.id, test_result_unfinished.test_user_id, test_result_unfinished.test_orientation,
            test_result_unfinished.create_time)
        return None

    queston_and_answer_all_unfinished = []
    queston_and_answer_unfinished = {}
    for orm_question_and_answer in test_question_and_answers_unfinished:
        orm_test_questions = orm_entity.query.filter(orm_entity.id == orm_question_and_answer.test_question_id).all()
        if orm_test_questions is None:
            mylog.logger.error(
                'no question details per test question index, id:{}, test_user_id:{}, test_answer:{}, test_question_id:{}, create_time:{}',
                orm_question_and_answer.id, orm_question_and_answer.test_user_id, orm_question_and_answer.test_answer,
                orm_question_and_answer.test_question_id, test_result_unfinished.create_time)
            continue
        orm_test_question = orm_test_questions[0]
        if "option_type" not in orm_test_question.__dict__.keys():
            orm_test_question.option_type = ConfusionsType.Text.value
        test_question = estimator._form_test_question(orm_test_question)
        if test_type == TestType.SPEAKING_EVALUATION:
            question_and_answer = EntityTestQuestionAndAnswer(test_question, orm_question_and_answer.test_answer)
        else:
            question_and_answer = EntityTestQuestionAndAnswer(test_question,
                                                              {'category': orm_question_and_answer.category,
                                                               'test_user_audio_path': orm_question_and_answer.test_user_audio_path,
                                                               'is_nonsense': orm_question_and_answer.is_nonsense,
                                                               'accuracy_score': orm_question_and_answer.accuracy_score,
                                                               'standard_score': orm_question_and_answer.standard_score,
                                                               'fluency_score': orm_question_and_answer.fluency_score,
                                                               'integrity_score': orm_question_and_answer.integrity_score,
                                                               'total_score': orm_question_and_answer.total_score}
                                                              )
        if orm_question_and_answer.test_answer is not None:
            question_and_answer.m_correct = question_and_answer.is_correct(test_type)
        queston_and_answer_all_unfinished.append(question_and_answer)
        queston_and_answer_unfinished[question_and_answer.question.bucket] = question_and_answer
    return queston_and_answer_unfinished, queston_and_answer_all_unfinished, test_result_unfinished
```

## 4 重构出多种缓存支持
    鼠标右键单击包目录libadsusertestsys/redis_module，选择Refactor/Rename...，改名为cache_module
    鼠标右键文件libadsusertestsys/cache_module/common_test_redis_tool.py，选择Refactor/Rename...，改名为redis
    全程替换libadsusertestsys/service/evaluation_service.py文件内的estimator_redis(whole word)为evaluation_cache
    全程替换libadsusertestsys/service/evaluation_service.py文件内的evaluation_redis(whole word)为evaluation_cache

## 5 libadsusertestsys工程增加libadsusertestsys/cache_module，新增文件memory.py
```
import logging

from libpycommon.common import mylog
from libpycommon.common import misc
from libadsusertestsys.entity.evaluation_entity import TestOrientation

EXPIRE_TIME = int(misc.get_env('EXPIRE_TIME', 60 * 60 * 24 * 3))

class TestMemoryTool:
    def __init__(self, test_type):
        self.test_type = test_type
        self.d_queston_and_answer = {}
        self.d_queston_and_answer_all = {}
        self.d_test_result = {}
        for test_orientation in TestOrientation.get_variables():
            self.d_queston_and_answer[test_orientation.value] = {}
            self.d_queston_and_answer_all[test_orientation.value] = {}
            self.d_test_result[test_orientation.value] = {}

    def exists_question_and_answer(self, test_orientation_value, user_id):
        if user_id in self.d_queston_and_answer_all[test_orientation_value]:
            return True

    def exists_test_result(self, test_orientation_value, user_id):
        if user_id in self.d_test_result[test_orientation_value]:
            return True

    def get_last_question(self, test_orientation_value, user_id):
        return self.d_queston_and_answer_all[test_orientation_value][user_id][-1]

    def get_test_result(self, test_orientation_value, user_id):
        return self.d_test_result[test_orientation_value][user_id].__dict__

    def add_last_question(self, test_orientation_value, user_id, question):
        if user_id not in self.d_queston_and_answer_all[test_orientation_value]:
            l_queston_and_answer_all = []
            self.d_queston_and_answer_all[test_orientation_value][user_id] = l_queston_and_answer_all
            d_bucket_queston_and_answer = {}
            self.d_queston_and_answer[test_orientation_value][user_id] = d_bucket_queston_and_answer
        else:
            l_queston_and_answer_all = self.d_queston_and_answer_all[test_orientation_value][user_id]
            d_bucket_queston_and_answer = self.d_queston_and_answer[test_orientation_value][user_id]
        l_queston_and_answer_all.append(question)

        bucket = question.question.bucket
        if bucket not in d_bucket_queston_and_answer:
            l_queston_and_answer = []
            d_bucket_queston_and_answer[bucket] = l_queston_and_answer
        else:
            l_queston_and_answer = d_bucket_queston_and_answer[bucket]
        l_queston_and_answer.append(question)

    def get_all_questions_in_order(self, test_orientation_value, user_id):
        return self.d_queston_and_answer_all[test_orientation_value][user_id]

    def add_test_result(self, test_orientation_value, user_id, test_result):
        self.d_test_result[test_orientation_value][user_id] = test_result

    def delete_question_and_answer_key_all(self, test_orientation_value, user_id):
        self.d_queston_and_answer[test_orientation_value].pop(user_id)
        self.d_queston_and_answer_all[test_orientation_value].pop(user_id)

    def delete_test_result(self, test_orientation_value, user_id):
        self.d_test_result[test_orientation_value].pop(user_id)
```

## 6 libadsusertestsys工程的libadsusertestsys/service/evaluation_service.py的entry函数
```
BOOL_CACHE_MEM_SWITCH = misc.get_env('CACHE_MEM_SWITCH', 'on') == 'on'
from libadsusertestsys.cache_module.redis import TestRedisTool
from libadsusertestsys.cache_module.memory import TestMemoryTool
    
    from libadsusertestsys.orm.user_test_sys_orm import app
    if BOOL_CACHE_MEM_SWITCH:
        evaluation_cache = TestMemoryTool(test_type)
    else:
        evaluation_cache = TestRedisTool(test_type)
    serve_forever(app, fn_generate_estimators(), test_type, evaluation_cache, myver)
```

## 7 libadsusertestsys工程libadsusertestsys/service/evaluation_service.py新增函数
```
def get_test_orientation_cache_index(test_orientation)
    return test_orientation.value if BOOL_CACHE_MEM_SWITCH else test_orientation.key

def get_test_orientation_str(test_orientation)
    return test_orientation.value if BOOL_CACHE_MEM_SWITCH else test_orientation.key
```

## 8 libadsusertestsys工程的所有原来redis调用传入参数test_orientation.value
    包括函数set_question_in_cache/set_batch_question_in_cache/set_test_result_in_cache/evaluate_algorithm/_update_answer
    不包括_get_estimator
    test_orientation.value换成get_test_orientation_cache_index(test_orientation)

## 9 libadsusertestsys工程的libadsusertestsys/service/evaluation_service.py的serve_forever函数
### 修改_update_answer函数
```
                    user_last_question = evaluation_redis.get_last_question(get_test_orientation_cache_index(test_orientation), openid)
```
### 新增_update_answer函数
```
                    user_last_question = evaluation_redis.get_last_question(get_test_orientation_cache_index(test_orientation), openid)
```
### 新增prexit接口
```
    INF_ROUTE_prexit = "/prexit"
    INF_LAG_prexit = _get_metrics_summary(INF_ROUTE_prexit)
    INF_COUNTER_prexit = _get_metrics_counter(INF_ROUTE_prexit)
    @app.route(INF_ROUTE_prexit, methods=["GET"])
    # @INF_LAG_prexit.time()
    def prexit():
        # INF_COUNTER_prexit.inc()
        def _req_fn():
            d_data = {}
            if not BOOL_CACHE_MEM_SWITCH:
                return get_serv_resp(d_data)
            for test_orientation in TestOrientation:
                d_queston_and_answer_all, d_test_result = evaluation_cache.get_all(test_orientation.value)
                l_rsp_result = []
                l_rsp_queston_and_answer = []
                for openid,user_question_and_answers in d_queston_and_answer_all.items():
                    if openid in d_test_result:
                        user_test_result = d_test_result[openid]
                    else:
                        user_test_result = UserTestResult(openid, None, None, None)
                    mylog.logger.debug('user_test_result:{}'.format(user_test_result))
                    d_result = {'openid': openid, 'user_test_result': user_test_result}
                    l_rsp_result.append(d_result)
                    test_result_id = save_test_result(openid, test_type.value, test_orientation.value,
                                                      float(user_test_result['score']),
                                                      int(user_test_result['level'].split("_")[1]),
                                                      float(user_test_result['rate_of_beaten']))
                    mylog.logger.debug('test_result_id:{}'.format(test_result_id))

                    mylog.logger.debug('openid:{}, user_question_and_answers:{}'.format(openid, user_question_and_answers))
                    d_question_and_answer = {'openid': openid, 'user_question_and_answers': user_question_and_answers}
                    batch_create_user_question_and_answer(test_type, openid, test_result_id, user_question_and_answers)
                    l_rsp_queston_and_answer.append(d_question_and_answer)
                d_data[test_orientation.key] = {'user_questions': l_rsp_queston_and_answer, 'user_result': l_rsp_result}
            return get_serv_resp(d_data)
        return get_internal_error_catched({}, _req_fn)
```
### 修改get_question函数
```
from libadsusertestsys.unfinished.dao import get_test_unfinished
                evaluation_estimator = _get_estimator(openid, test_orientation)
                unfinished_test, queston_and_answer_unfinished, queston_and_answer_all_unfinished = get_test_unfinished(test_orientation.value, test_type, openid)
                rqdata_result = evaluate_algorithm(openid, test_type, test_orientation, evaluation_estimator, evaluation_redis, question_rev_id)
```

