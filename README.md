# VideoDecode
(转)最简单的基于FFmpeg的解码器-纯净版（不包含libavformat）


本文记录一个更加“纯净”的基于FFmpeg的视频解码器。此前记录过基于FFmpeg的视频播放器实际上就是一个解码器：
《最简单的基于FFMPEG+SDL的视频播放器 ver2 （采用SDL2.0）》
这个播放器调用了FFmpeg中的libavformat和libavcodec两个库完成了视频解码工作。但是这不是一个“纯净”的解码器。该解码器中libavformat完成封装格式的解析，而libavcodec完成解码工作。一个“纯净”的解码器，理论上说只需要使用libavcodec就足够了，并不需要使用libavformat。本文记录的解码器就是这样的一个“纯净”的解码器，它仅仅通过调用libavcodec将H.264/HEVC等格式的压缩视频码流解码成为YUV数据。
流程图
本文记录的纯净版本的基于FFmpeg的解码器的函数调用流程图如下图所示。需要注意的是，此解码器的输入必须是只包含视频编码数据“裸流”（例如H.264、HEVC码流文件），而不能是包含封装格式的媒体数据（例如AVI、MKV、MP4）。

流程图中关键函数的作用如下所列：
avcodec_register_all()：注册所有的编解码器。
avcodec_find_decoder()：查找解码器。
avcodec_alloc_context3()：为AVCodecContext分配内存。
avcodec_open2()：打开解码器。
avcodec_decode_video2()：解码一帧数据。
有两个平时“不太常见”的函数：
av_parser_init()：初始化AVCodecParserContext。
av_parser_parse2()：解析获得一个Packet。

两个存储数据的结构体如下所列：
AVFrame：存储一帧解码后的像素数据
AVPacket：存储一帧（一般情况下）压缩编码数据

AVCodecParser
AVCodecParser用于解析输入的数据流并把它分成一帧一帧的压缩编码数据。比较形象的说法就是把长长的一段连续的数据“切割”成一段段的数据。他的核心函数是av_parser_parse2()。它的定义如下所示。
[cpp] view plaincopy在CODE上查看代码片派生到我的代码片
/** 
 * Parse a packet. 
 * 
 * @param s             parser context. 
 * @param avctx         codec context. 
 * @param poutbuf       set to pointer to parsed buffer or NULL if not yet finished. 
 * @param poutbuf_size  set to size of parsed buffer or zero if not yet finished. 
 * @param buf           input buffer. 
 * @param buf_size      input length, to signal EOF, this should be 0 (so that the last frame can be output). 
 * @param pts           input presentation timestamp. 
 * @param dts           input decoding timestamp. 
 * @param pos           input byte position in stream. 
 * @return the number of bytes of the input bitstream used. 
 * 
 * Example: 
 * @code 
 *   while(in_len){ 
 *       len = av_parser_parse2(myparser, AVCodecContext, &data, &size, 
 *                                        in_data, in_len, 
 *                                        pts, dts, pos); 
 *       in_data += len; 
 *       in_len  -= len; 
 * 
 *       if(size) 
 *          decode_frame(data, size); 
 *   } 
 * @endcode 
 */  
int av_parser_parse2(AVCodecParserContext *s,  
                     AVCodecContext *avctx,  
                     uint8_t **poutbuf, int *poutbuf_size,  
                     const uint8_t *buf, int buf_size,  
                     int64_t pts, int64_t dts,  
                     int64_t pos);  

其中poutbuf指向解析后输出的压缩编码数据帧，buf指向输入的压缩编码数据。如果函数执行完后输出数据为空（poutbuf_size为0），则代表解析还没有完成，还需要再次调用av_parser_parse2()解析一部分数据才可以得到解析后的数据帧。当函数执行完后输出数据不为空的时候，代表解析完成，可以将poutbuf中的这帧数据取出来做后续处理。

对比
简单记录一下这个只使用libavcodec的“纯净版”视频解码器和使用libavcodec+libavformat的视频解码器的不同。
PS：使用libavcodec+libavformat的解码器参考文章《最简单的基于FFMPEG+SDL的视频播放器 ver2 （采用SDL2.0）》
（1） 下列与libavformat相关的函数在“纯净版”视频解码器中都不存在。
av_register_all()：注册所有的编解码器，复用/解复用器等等组件。其中调用了avcodec_register_all()注册所有编解码器相关的组件。
avformat_alloc_context()：创建AVFormatContext结构体。
avformat_open_input()：打开一个输入流（文件或者网络地址）。其中会调用avformat_new_stream()创建AVStream结构体。avformat_new_stream()中会调用avcodec_alloc_context3()创建AVCodecContext结构体。
avformat_find_stream_info()：获取媒体的信息。
av_read_frame()：获取媒体的一帧压缩编码数据。其中调用了av_parser_parse2()。
（2） 新增了如下几个函数。
avcodec_register_all()：只注册编解码器有关的组件。比如说编码器、解码器、比特流滤镜等，但是不注册复用/解复用器这些和编解码器无关的组件。
avcodec_alloc_context3()：创建AVCodecContext结构体。
av_parser_init()：初始化AVCodecParserContext结构体。
av_parser_parse2()：使用AVCodecParser从输入的数据流中分离出一帧一帧的压缩编码数据。
（3） 程序的流程发生了变化。
在“libavcodec+libavformat”的视频解码器中，使用avformat_open_input()和avformat_find_stream_info()就可以解析出输入视频的信息（例如视频的宽、高）并且赋值给相关的结构体。因此我们在初始化的时候就可以通过读取相应的字段获取到这些信息。
在“纯净”的解码器则不能这样，由于没有上述的函数，所以不能在初始化的时候获得视频的参数。“纯净”的解码器中，可以通过avcodec_decode_video2()获得这些信息。因此我们只有在成功解码第一帧之后，才能通过读取相应的字段获取到这些信息。


源代码
[cpp] view plaincopy在CODE上查看代码片派生到我的代码片
