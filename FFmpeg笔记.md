# FFmpeg笔记

## FFmpeg常用命令

#### 查看媒体文件信息

``` shell
ffmpeg -i input.mp4
```

#### 提取音频或视频

``` shell
ffmpeg -i input.mp4 -q:a 0 -map a output.mp3
ffmpeg -i input.mp4 -an -c:v copy output.mp4
```

## FFmpeg API使用流程

``` c

//注册编解码器（高版本FFmpeg已经删除了该API）
av_register_all();

//打开文件，获取流信息
AVFormatContext *formatContext = avformat_alloc_context();
if (avformat_open_input(&formatContext, "input.mp4", NULL, NULL) != 0) {
    return -1;
}
if (avformat_find_stream_info(formatContext, NULL) < 0) {
    return -1;
}

//查找对应的音视频流
int index = -1;
for (int i = 0; i < formatContext->nb_streams; i++) {
    if (formatContext->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {
        index = i;
        break;
    }
}

//根据需要创建编码器或解码器
AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_H264);
AVCodec *codec = avcodec_find_decoder(codecParameters->codec_id);

//准备帧和包
AVFrame *frame = av_frame_alloc();
AVPacket *packet = av_packet_alloc();

//解码的话，需要从流中读取数据
av_read_frame(formatContext, packet);
//编码的话，需要往frame中填充参数和YUV数据
......

//数据发送给解码器
avcodec_send_packet(codecContext, packet)；
//数据发送给编码器
avcodec_send_frame(codecContent, frame);

//从解码器获取解码后的数据
avcodec_receive_frame(codecContext, frame) 
//从编码器获取编码后的数据
avcodec_receive_packet(codecContent, packet);

//编解码后，拿到数据可能还要做一些操作
......

//清理资源
av_frame_free(&frame);
av_packet_free(&packet);
avcodec_free_context(&codecContext);
avformat_close_input(&formatContext);

```