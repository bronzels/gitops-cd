# AI测评-概要设计-qrcode文件接口直接下载文件内容不保存

## 1 新增缺省临时文件保存目录，下载大小
```
FILE_DL_TMPSAVED_PATH = '/tmp'
FILE_DL_SIZE = 9000
#20 * 1024 * 1024# 每次读取20M
last_dl_wx_qrcode_path:str = None
```

## 2 usertestsys-general工程的evaluation/service/evaluation_service.py的get_wx_qrcode函数
```
import io
            if qrcode == "500":
                message = "unable to get qr code"
                response = make_response(jsonify({"error": message}))
                response.headers['Access-Control-Allow-Origin'] = '*'
                return response
            filename = QRCODE_ABSPATH_ROOT+"/{}_{}_{}.jpg".format(
                    APPID, width, path.replace("/", "_").replace("?", "_"))    
            target_file = io.BytesIO(qrcode)
            def send_chunk():  # 流式读取
                while True:
                    chunk = target_file.read(FILE_DL_SIZE)
                    if not chunk:
                        break
                    yield chunk
            from flask import Response
            response = Response(send_chunk(), content_type='application/octet-stream')
            response.headers["Content-disposition"] = 'attachment; filename=%s' % filename
            return response
        return get_internal_error_catched({}, _req_fn, path, width)
```
