状态码字段在响应管理帧中用于指示请求操作的成功或失败。如果某项过程成功，该位的值就会被设定为 0 或 85(SUCCESS_POWER_SAVE_MODE)，否则设为非零值
![alt text](status_code.assets/image.png)

比如 sta 在 ap 的黑名单中, 在 sta 尝试连接时就会收到`status code!=0`的回复:
![alt text](status_code.assets/image-1.png)
