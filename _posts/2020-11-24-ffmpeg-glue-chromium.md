---
layout: post
author: angtylook
title: Chromium中的FFMpeg glue
---

使用ffmpeg打开音视频文件，解封状，即做Demuxer时，需要用到函数`avformat_open_input`，该函数有两种使用方式。
第一种是传入文件名，由该API创建`AVFormatContext`，如下：
```cpp
int main() {
	AVFormatContext* format_context = nullptr;
	if (avformat_open_input(&format_context, argv[1], NULL, NULL) != 0) {
		return -1;
	}
	return 0;
}
```
另一种是提供自定义的字节流IO，即提供`AVIOContext`结构，下面的例子来源chromium的media模块。
先看一下创建AVIOContext的`avio_alloc_context`函数声明。
```c
AVIOContext *avio_alloc_context(
                  unsigned char *buffer,
                  int buffer_size,
                  int write_flag,
                  void *opaque,
                  int (*read_packet)(void *opaque, uint8_t *buf, int buf_size),
                  int (*write_packet)(void *opaque, uint8_t *buf, int buf_size),
                  int64_t (*seek)(void *opaque, int64_t offset, int whence));`
```
可以看到需要提供`read`，`write`，`seek`这三个操作，`opaque`是自定义的数据。其中`write`不是必须的，一般解封装也不需要写入。在chromium里面，定义了`FFmpegURLProtocol`接口来抽象各种输入。
```cpp
class MEDIA_EXPORT FFmpegURLProtocol {
 public:
  // Read the given amount of bytes into data, returns the number of bytes read
  // if successful, kReadError otherwise.
  virtual int Read(int size, uint8_t* data) = 0;

  // Returns true and the current file position for this file, false if the
  // file position could not be retrieved.
  virtual bool GetPosition(int64_t* position_out) = 0;

  // Returns true if the file position could be set, false otherwise.
  virtual bool SetPosition(int64_t position) = 0;

  // Returns true and the file size, false if the file size could not be
  // retrieved.
  virtual bool GetSize(int64_t* size_out) = 0;

  // Returns false if this protocol supports random seeking.
  virtual bool IsStreaming() = 0;
};
```
基于此再实现`avio_alloc_context`需要的`read`、`seek`函数，见下：
```cpp
// Internal buffer size used by AVIO for reading.
// TODO(dalecurtis): Experiment with this buffer size and measure impact on
// performance.  Currently we want to use 32kb to preserve existing behavior
// with the previous URLProtocol based approach.
enum { kBufferSize = 32 * 1024 };

static int AVIOReadOperation(void* opaque, uint8_t* buf, int buf_size) {
  return reinterpret_cast<FFmpegURLProtocol*>(opaque)->Read(buf_size, buf);
}

static int64_t AVIOSeekOperation(void* opaque, int64_t offset, int whence) {
  FFmpegURLProtocol* protocol = reinterpret_cast<FFmpegURLProtocol*>(opaque);
  int64_t new_offset = AVERROR(EIO);
  switch (whence) {
    case SEEK_SET:
      if (protocol->SetPosition(offset))
        protocol->GetPosition(&new_offset);
      break;

    case SEEK_CUR:
      int64_t pos;
      if (!protocol->GetPosition(&pos))
        break;
      if (protocol->SetPosition(pos + offset))
        protocol->GetPosition(&new_offset);
      break;

    case SEEK_END:
      int64_t size;
      if (!protocol->GetSize(&size))
        break;
      if (protocol->SetPosition(size + offset))
        protocol->GetPosition(&new_offset);
      break;

    case AVSEEK_SIZE:
      protocol->GetSize(&new_offset);
      break;

    default:
      NOTREACHED();
  }
  return new_offset;
}
```
使用时，先用`avformat_alloc_context`创建`AVFormatContext`，再用`avio_alloc_context`创建`AVIOContext`，设置使用`AVFMT_FLAG_CUSTOM_IO`的参数并赋值`AVIOContext`实例给`AVFormatContext`。之后调用`avformat_open_input`时，提供自定义的`AVFormatConext`参数并且不提供第二个文件名参数。

```cpp
FFmpegGlue::FFmpegGlue(FFmpegURLProtocol* protocol) {
  // Initialize an AVIOContext using our custom read and seek operations.  Don't
  // keep pointers to the buffer since FFmpeg may reallocate it on the fly.  It
  // will be cleaned up
  format_context_ = avformat_alloc_context();
  avio_context_.reset(avio_alloc_context(
      static_cast<unsigned char*>(av_malloc(kBufferSize)), kBufferSize, 0,
      protocol, &AVIOReadOperation, nullptr, &AVIOSeekOperation));

  // Ensure FFmpeg only tries to seek on resources we know to be seekable.
  avio_context_->seekable =
      protocol->IsStreaming() ? 0 : AVIO_SEEKABLE_NORMAL;

  // Ensure writing is disabled.
  avio_context_->write_flag = 0;

  // Tell the format context about our custom IO context.  avformat_open_input()
  // will set the AVFMT_FLAG_CUSTOM_IO flag for us, but do so here to ensure an
  // early error state doesn't cause FFmpeg to free our resources in error.
  format_context_->flags |= AVFMT_FLAG_CUSTOM_IO;

  // Enable fast, but inaccurate seeks for MP3.
  format_context_->flags |= AVFMT_FLAG_FAST_SEEK;

  // Ensures we can read out various metadata bits like vp8 alpha.
  format_context_->flags |= AVFMT_FLAG_KEEP_SIDE_DATA;

  // Ensures format parsing errors will bail out. From an audit on 11/2017, all
  // instances were real failures. Solves bugs like http://crbug.com/710791.
  format_context_->error_recognition |= AV_EF_EXPLODE;

  format_context_->pb = avio_context_.get();
}

bool FFmpegGlue::OpenContext(bool is_local_file) {
  DCHECK(!open_called_) << "OpenContext() shouldn't be called twice.";

  // If avformat_open_input() is called we have to take a slightly different
  // destruction path to avoid double frees.
  open_called_ = true;

  // By passing nullptr for the filename (second parameter) we are telling
  // FFmpeg to use the AVIO context we setup from the AVFormatContext structure.
  const int ret =
      avformat_open_input(&format_context_, nullptr, nullptr, nullptr);
  // 下面省略....
}
```
