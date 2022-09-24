# AI测评-概要设计-流的方式上传口语录音大文件（上传wav文件）

## 1 libadsusertestsys工程的libadsusertestsys/service/evaluation_service.py
### _update_answer
    增加is_by_post:bool参数
```
            else:
                if test_type == TestType.SPEAKING_EVALUATION:
                    #......
                    code, message, d_data_2deco, answer = evaluation_estimator.get_result_by_auto_recog_service(openid, user_last_question, audio_file, is_by_post)
                else:
                    if not is_by_post:
                            answer = str(answer, encoding='utf-8')
                if code == 0:
```
### update_answer调用_update_answer时is_by_post=true
### update_fileanswer
    调用_update_answer时is_by_post=false
    update_fileanswer
    取消byte转str处理：
        answer_read = str(answer.stream.read(), encoding='utf-8')
        answer_read = answer.stream

## 2 usertestsys-speakingevaluation工程的evaluation/entity/evaluation_entity.py的WebsocketReq类
### 构造函数
	注释掉self.initial_ws()
    audio_file变量重命名位audio_file_name=None,挪到参数列表最后
    后增加audio_stream=Noneqw的成员变量并赋值
    判断以上2个变量都是None或都不是None，raise ValueError(翻译)
### on_open/run函数
```
            start = int(time.time() * 1000)
            with open(self.audio_file, 'rb') if audio_stream is None else audio_stream as f:
                while True:
```

## 3 usertestsys-speakingevaluation工程的evaluation/model/placement_test_speaking_estimator.py的get_result_by_auto_recog_service函数
```
    def get_pcm_file(self, openid, category, text, audio_file):
        today = time.localtime()
        nonabs_path = '/{}'.format(openid) \
                      + '/' + str(today.tm_year) + str(today.tm_mon) + str(today.tm_mday)
        origin_audio_path = AUDIOGEN_ABSPATH_REC_AUDIO_POOL_ORIGIN + nonabs_path
        pcm_audio_path = AUDIOGEN_ABSPATH_REC_AUDIO_POOL_PCM + '/{}'.format(openid) \
                         + '/' + str(today.tm_year) + str(today.tm_mon) + str(today.tm_mday)
        # 当category=read_wo设置text特定格式
        if category == 'word':
            text = '[word]\t' + text
        # 没有当日日期文件夹则创建之
        # if not os.path.exists(self.pcm_dir):
        #     os.makedirs(self.pcm_dir)
        #
        # if not os.path.exists(self.origin_dir):
        #     os.makedirs(self.origin_dir)
        #     mylog.logger.info('mkdir done.')

        # 没有当日日期文件夹则创建之
        if not os.path.exists(origin_audio_path):
            os.makedirs(origin_audio_path)

        if not os.path.exists(pcm_audio_path):
            os.makedirs(pcm_audio_path)
            mylog.logger.info('mkdir done.')

        # 将base64码转音频
        # 录音命名：openid + time
        audio_name = str(openid) + '_' + str(time.time())
        db_path = os.path.join(nonabs_path, audio_name)
        if audio_file[:10] == 'data:audio':
            origin_file_path = base64_to_file(base64_data=audio_file, file_path=origin_audio_path, file_name=audio_name)
            mylog.logger.info('base64_to_file done.')
            # 转化音频pcm格式
            audio_pcm = os.path.join(pcm_audio_path, audio_name)
            # audio_pcm = "D:/python/usertestsys-speakingevaluation/files/audiogen/spe/origin/3/202133/3_1614737643.1522696"
            pcm_transformer(origin_file_path, audio_pcm)
            return audio_pcm
        else:
            return None

    def get_result_by_auto_recog_service(self, openid, last_unanswered_question2answer, audio_input, is_by_post):
        last_unanswered_question = last_unanswered_question2answer.question
        category = last_unanswered_question.category
        text = last_unanswered_question.stem

        if is_by_post:
            audio_pcm = self.get_pcm_file(openid, category, text, audio_input)
            if audio_pcm is None:
                mylog.logger.error('录音出错！')
                return format_response(data=ResponseResultData(
                    openid, "record failure"), code=3002, msg="录音出错")
            websocket_req = WebsocketReq(
                appid=XUNFEI_APPID,
                apisecret=XUNFEI_APISECRET,
                apikey=XUNFEI_APIKEY,
                audio_file_name=audio_pcm + '.pcm',
                text=text,
                category=category,
                ent='en_vip',
                result_xml_form=RESULT_XML_FORM
            )
        else:
            websocket_req = WebsocketReq(
                appid=XUNFEI_APPID,
                apisecret=XUNFEI_APISECRET,
                apikey=XUNFEI_APIKEY,
                audio_stream=audio_input,
                text=text,
                category=category,
                ent='en_vip',
                result_xml_form=RESULT_XML_FORM
            )

        start = int(time.time() * 1000)
        # 调用讯飞语音评测api
        websocket_req.initial_ws()
        
        if websocket_req.err_code != None:
            raise MyError(websocket_req.err_code, subcode=websocket_req.err_subcode)
        end = int(time.time() * 1000)
        mylog.logger.info('xunfei inf lag, start:{}, lag:{}, end:{}'.format(start, end - start, end))
        if RESULT_XML_FORM == 'base64':
            xml_parse = websocket_req.return_data
        else:
            xml_parse = read_ise_xml_simple(websocket_req.return_data)

        if RESULT_XML_FORM == 'base64':
```
