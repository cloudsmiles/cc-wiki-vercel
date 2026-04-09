# 安装grpcurl

背景：在开发 grpc 接口的时候，希望在本地调试一下，可以使用 grpcurl 工具调用 grpc 接口。

1. 使用 brew 安装 grpcurl

`brew install grpcurl`

[https://zegocloud.feishu.cn/space/api/box/stream/download/asynccode/?code=NjllODg2YzFlNGIyOWI2MGMyOTY5NDVhZDRhYmE3OGFfbEdMT3IxREt2Z1QzWTFyc3U5Q2lDa0szY0JSQW9UU2RfVG9rZW46VzhiQ2IzdUR2b0lWV0Z4MEtDdGNuTnlDbldkXzE2ODc4NjUxODY6MTY4Nzg2ODc4Nl9WNA](https://zegocloud.feishu.cn/space/api/box/stream/download/asynccode/?code=NjllODg2YzFlNGIyOWI2MGMyOTY5NDVhZDRhYmE3OGFfbEdMT3IxREt2Z1QzWTFyc3U5Q2lDa0szY0JSQW9UU2RfVG9rZW46VzhiQ2IzdUR2b0lWV0Z4MEtDdGNuTnlDbldkXzE2ODc4NjUxODY6MTY4Nzg2ODc4Nl9WNA)

1. 查看所有的 grpc service，16161 是本地 grpc 服务的端口，可以在服务启动日志中查看，每个服务都不同。

`grpcurl -plaintext 127.0.0.1:16161 list`

[https://zegocloud.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDRmMjFiNmU0ZmFjNDlkMzVmYzdhYTQ3YjdmZjEyODlfbTg1RE84Wm9haEg5TDF3ejBWYkpPc0EzZTl0RmZyUm9fVG9rZW46WWpDS2JLMUVHb0ZIWm54MFZqVGN1YmdBbkpjXzE2ODc4NjUzOTM6MTY4Nzg2ODk5M19WNA](https://zegocloud.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDRmMjFiNmU0ZmFjNDlkMzVmYzdhYTQ3YjdmZjEyODlfbTg1RE84Wm9haEg5TDF3ejBWYkpPc0EzZTl0RmZyUm9fVG9rZW46WWpDS2JLMUVHb0ZIWm54MFZqVGN1YmdBbkpjXzE2ODc4NjUzOTM6MTY4Nzg2ODk5M19WNA)

1. 查看某个 service 中的 grpc 方法

`grpcurl -plaintext 127.0.0.1:16161 list service_name`

[https://zegocloud.feishu.cn/space/api/box/stream/download/asynccode/?code=NDk2NzRkM2ZlMzQ4ODgzN2M4MzlkMDU5ZGQ0ZjZhYTVfZU5XcDJqSmxRSG96eW5aMEEyVm1yZkdsRmE5c0hQc3lfVG9rZW46U2FLTGIzQTJ1b3FCYVp4YnZab2NXY1R5bmtiXzE2ODc4NjUzOTM6MTY4Nzg2ODk5M19WNA](https://zegocloud.feishu.cn/space/api/box/stream/download/asynccode/?code=NDk2NzRkM2ZlMzQ4ODgzN2M4MzlkMDU5ZGQ0ZjZhYTVfZU5XcDJqSmxRSG96eW5aMEEyVm1yZkdsRmE5c0hQc3lfVG9rZW46U2FLTGIzQTJ1b3FCYVp4YnZab2NXY1R5bmtiXzE2ODc4NjUzOTM6MTY4Nzg2ODk5M19WNA)

reference:

[https://github.com/fullstorydev/grpcurl](https://github.com/fullstorydev/grpcurl)