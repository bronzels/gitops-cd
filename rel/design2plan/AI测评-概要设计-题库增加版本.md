# AI测评-概要设计-题库增加版本

## 1 libadsusertestsys工程的libadsusertestsys/orm/user_test_sys_orm.py
表（除了TestUser、TestResult）：
```
	TestQuestion
	SpeakingTestQuestion
	VocabularyTestQuestion
	GrammarTestQuestion
	ListeningTestQuestion
	ReadingTestQuestionMaterial
	ReadingTestQuestion
```
增加字段：
```
	question_rev_id Column(Integer, nullable=False) #整型，TestQuestionRev的外键，对应id
```
新增表：
```
CREATE TABLE TestQuestionRev
(
[id] INT IDENTITY(1,1),
[question_rev] VARCHAR(50) NOT NULL,
[obsolete] BIT NULL,
[create_time] datetime DEFAULT (getdate()) NULL
)
Go
CREATE UNIQUE INDEX index_question_rev
ON TestQuestionRev (question_rev)
Go
```
```
class TestQuestionRev(db.Model):
    """测评题库版本表"""
    __tablename__ = 'TestQuestionRev'
    id = Column(Integer, autoincrement=True, primary_key=True)
    question_rev Column(String(50), nullable=False, unique=True)	#字符串，和目前usertestsys文件目录名保持一致，rel_时间
    obsolete = Column(Boolean, default=False)
    create_time = Column(DateTime, server_default=func.now())
```

## 2 导入程序
	增加环境变量QUESTION_REV	#字符串，和目前usertestsys文件目录名保持一致，rel_时间
	判断如果相同question_rev已经在TestQuestionRev存在，不允许插入
	插入TestQuestionRev1条记录(obsolete=True)，拿到id，所有静态表插入时，新增question_rev_id #整型，TestQuestionRev的外键，对应id
	不同版本题目统一放在/cdhdata1/analysis_group/usertestsys/下rel_xxx目录下
    UPDATE TestQuestionRev SET obsolete=False WHERE question_rev = '' --上一个版本（课程部产品经理无多版本需求，多版本只是为了不停服升级）

## 3 gitops-cd
### overlays/beta/setup/configmap-servpy-usertestsys.yaml（prod的一样处理）：<br>
	增加环境变量DEFAULT_QUESTION_REV	#字符串，和目前usertestsys文件目录名保持一致，rel_时间
	取消环境变量FILE_ROOT

## 4 libadsusertestsys工程的libadsusertestsys/entity/evaluation_entity.py
### 类：
	EntityTestQuestion, MultipleChoiceTestQuestion，TrueOrFalseQuestion
### 构造函数的bucket后面增加成员变量：
	question_rev_id #整型，TestQuestionRev的外键，对应id

## 5 app工程的evaluation/entity/evaluation_entity.py
### 类：
	grammar/GrammarMultipleChoiceTestQuestion
	reading/ReadingMultipleChoiceTestQuestion
	listening/ListeningMultipleChoiceTestQuestion
	speaking/SpeakingDirectAnswerTestQuestion
	vocabularysize/VocabTestMultipleChoiceQuestion
构造函数def __init__和父类构造函数super().__init__(的bucket后面增加成员变量：question_rev_id

## 6 app工程的_form_test_question函数
	grammar/reading/listening/speaking/vocabularysize的model/placement_test_${test_type}_estimator的_form_test_question函数
	test_question = 后构造函数bucket后面增加
```
            question_rev_id=question.question_rev_id #整型，TestQuestionRev的外键，对应id
```

## 7 libadsusertestsys工程的libadsusertestsys/dao/evaluation_dao.py
### 新增函数，查询get_test_question_rev，过滤obsolete，返回1个dict<question_rev:id>，1个list<id>
### 修改函数get_placement_test_questions_by_test_type_and_level_interval
	增加参数l_question_rev_id，filter中增加判断question_rev_id in l_question_rev_id

## 8 libadsusertestsys工程的libadsusertestsys/orm/config.py
新增代码
```
from libadsusertestsys/dao/evaluation_dao import get_test_question_rev
d_question_rev2id, l_question_rev_id = get_test_question_rev
DEFAULT_QUESTION_REV = misc.get_env('DEFAULT_QUESTION_REV', '')
default_question_rev_id = d_question_rev2id.get(DEFAULT_QUESTION_REV)
if default_question_rev_id is None: raise ValueError('environment variable DEFAULT_QUESTION_REV is not matched with non obsolete question_rev in table  TestQuestionRev')
```
修改代码
```
_dockerv_k8spvc_mounted_files_path_root = '/mnt/analysis_group/usertestsys'
FILES_URL_PREFIX = misc.get_env('FILES_URL_PREFIX')
if DEFAULT_QUESTION_REV != '':
    _dockerv_k8spvc_mounted_files_path_root += '/' + DEFAULT_QUESTION_REV
   FILES_URL_PREFIX +=  '/' + DEFAULT_QUESTION_REV
```

## 9 在libadsusertestsys/vocabularysize工程中中，把questions_by_bucket全程替换为questions_by_rev_bucket

## 10 libadsusertestsys工程的libadsusertestsys/model/placement_test_estimator.py的load_evaluation_questions函数
新增代码
```
from libadsusertestsys.orm.config import l_question_rev_id
```
调用函数get_placement_test_questions_by_test_type_and_level_interval，增加参数l_question_rev_id
```
        for question in all_questions:
            if "option_type" not in question.__dict__.keys():
                question.option_type = ConfusionsType.Text.value
            test_question = self._form_test_question(question)
            question_rev_id = test_question.question_rev_id #新代码

            #修改代码start
            rev_bucket_questions = self.questions_by_rev_bucket[  # 初始为空字典
                test_question.question_rev_id] \
                if test_question.question_rev_id in self.questions_by_rev_bucket else []
            if test_question.bucket in rev_bucket_questions:    
                bucket_questions = rev_bucket_questions[  # 初始为空字典
                    test_question.bucket]
            else:     
                bucket_questions = []
                self.questions_by_rev_bucket[test_question.question_rev_id] = bucket_questions
            bucket_questions.append(test_question)
            #修改代码end
            questions_count += 1
        mylog.logger.debug("{} test question loaded! {} questions available".format(self.test_type, questions_count))
```

## 11 libadsusertestsys工程的libadsusertestsys/model/placement_test_beta_distribution_based_estimator.py
### 函数_get_next_available_questions
	增加参数question_rev_id
	self.questions_by_bucket替换为self.questions_by_rev_bucket[question_rev_id]
### 函数get_next_question
	增加参数question_rev_id
	调用self._get_next_available_questions(，增加参数question_rev_id

## 12 libadsusertestsys工程的libadsusertestsys/service/evaluation_service.py
### 函数evaluate_algorithm
	2处调用evaluation_estimator.get_next_question(，增加参数question_rev_id
### 函数get_question
新增代码
```
from libadsusertestsys/orm/config import d_question_rev2id 
            #修改代码start
            test_orientation = TestOrientation(test_orientation_str)
            question_rev_id = default_question_rev_id
            question_rev = request_data.get("question_rev")	
            if  question_rev is not None:
                        if question_rev in d_question_rev2id :  
                                    question_rev_id = d_question_rev2id[question_rev]
                        else: raise MyError(ERR_CODE_INF_INPUT_INVALID, additional_msg='question_rev:{}'.format(question_rev))
            mylog.logger.info('test_orientation:{}'.format(test_orientation.__dict__))
            #修改代码end
```
	调用evaluate_algorithm(，增加参数question_rev_id

## 13 usertestsys-md工程的
### introduction.md
	1.3.6 code状态码对照libadsusertestsys工程的libadsusertestsys/common/utils.py的ERR_CODE_修改一致
### api/*.md
	x.3 请求参数，增加1列，必选，值：是/否，具体格式见下文，譬如
		openid的“必选”列的值都是“是”
		usergeneral.md/4. 登录/4.3 请求参数的inviter_openid的“必选”列的值是“否”
### api/*.md（除usergeneral.md以外）
1.3 请求参数

字段       |字段类型       |字段说明       |必选
------------|-----------|-----------|-----------
openid       |string        |用户openid        |是
question_rev       |string        |题库版本        |否

字段       |枚举取值       |枚举取值说明
------------|-----------|-----------
question_rev       |rev_xxx        |目前无多版本需求下最新版本题库




