## Resembler

```C++

// 当前进党设置了closed标识位且待重组的字节为0
bool Reassembler::is_closed() const { return closed_ && bytes_pending() == 0; }   

void Reassembler::insert(uint64_t first_index, string data, bool is_last_substring, Writer &output)
{
    if (is_last_substring) {
        closed_ = true;
    }
待处理的超出可容纳范围 || Data have been transferred || data为空 || 可用capacity为0
    if (first_index >= unassembled_index_ + output.available_capacity() || /* Out of bound */
        first_index + data.length() - 1 < unassembled_index_ ||            /* Data have been transferred */
        data.empty() || output.available_capacity() == 0) {
        if (is_closed()) {
            output.close();
        }
        return;
    }

    const uint64_t cap = output.available_capacity();
    // new_index actually distinguish where the current data start, the start index
    uint64_t new_index = first_index;
    
    // Data needs to fit the capability limitation	
    if (first_index <= unassembled_index_) {
        // 有重复
        new_index = unassembled_index_;
        // overlapped_length = 70 - 60;
        const uint64_t overlapped_length = unassembled_index_ - first_index;
        // substr(10, 20 - 10); data.size = [60, 80] => [0, 20]
        data = data.substr(overlapped_length, min(data.size() - overlapped_length, cap));
    } else {
        // 没有重复
        // get 全部数据or可容纳的部分
        data = data.substr(0, min(data.size(), cap));
        if (first_index + data.size() - 1 > unassembled_index_ + cap - 1) {
            // 
            data = data.substr(0, unassembled_index_ + cap - first_index);
        }
    }
    
    // 主要是处理 unassembled_substrings_ 和
    // 获取 >=new_index的最小索引
    auto rear_iter = unassembled_substrings_.lower_bound(new_index);
    while (rear_iter != unassembled_substrings_.end()) {
        auto &[rear_index, rear_data] = *rear_iter;
        // 和rear_index没有重叠
        if (new_index + data.size() - 1 < rear_index) {
            break;
        } // No overlap conflict
        
        // 否则就是有重叠
        uint64_t rear_overlapped_length = 0;
        if (new_index + data.size() - 1 < rear_index + rear_data.size() - 1) {
            // the last index of current data less than counterpart of the rear_data
            // denote that current: [. [  ]. ], then cut from [new_index, new_index + overlap_len]，这种case新的或者旧的应该包含重叠部分，这里采用的是旧的包含重叠部分，新的不包含重叠部分
            rear_overlapped_length = new_index + data.size() - rear_index;
        } else {
            // else current: [. [  ] ]. 这种case旧的substring就不应该需要了，新的替换
            rear_overlapped_length = rear_data.size();
        }
        // Prepare for next rear early, because the data may be erased afterwards.
        const uint64_t next_rear = rear_index + rear_data.size() - 1; // last index of rear_data
        if (rear_overlapped_length == rear_data.size()) {
          // 走了上面的else，就会走这里
            unassembled_bytes_ -= rear_data.size();
            unassembled_substrings_.erase(rear_index); // 抹除旧的
        } else {
          	// 走了上面的if，就会走这里
            // We don't combine current data and rear data.
            // Erase the overlapped part in current data is more efficient.
            data.erase(data.end() - static_cast<int64_t>(rear_overlapped_length), data.end());
        }
        // 寻找下一个 >= next_rear的substring
        rear_iter = unassembled_substrings_.lower_bound(next_rear);
    }
    
     // 同样地主要是处理 unassembled_substrings_ 和最大索引
     // if the current substring behind the unassembled_index_
     if (first_index > unassembled_index_) {
        // 获取>new_index的最小值
        auto front_iter = unassembled_substrings_.upper_bound(new_index);
        if (front_iter != unassembled_substrings_.begin()) {
            // 递减front_iter
            front_iter--;
            const auto &[front_index, front_data] = *front_iter;
            // if first_index, ]
            if (front_index + front_data.size() - 1 >= first_index) {
                uint64_t overlapped_length = 0;
                // if  ] last_index
                if (front_index + front_data.size() <= first_index + data.size()) {
                   // 一次性把之前的全部delete了，所以不需要遍历
                    overlapped_length = front_index + front_data.size() - first_index;
                } else {
                    // if  last_index  ]
                    overlapped_length = data.size();
                }
                if (overlapped_length == front_data.size()) {
                    unassembled_bytes_ -= front_data.size();
                    unassembled_substrings_.erase(front_index);
                } else {
                    data.erase(data.begin(), data.begin() + static_cast<int64_t>(overlapped_length));
                    // Don't forget to update the inserted location
                    new_index = first_index + overlapped_length;
                }
            }
        }
    }
    
    
    // If the processed data is empty, no need to insert it.
    if (data.size() > 0) {
        unassembled_bytes_ += data.size();
        unassembled_substrings_.insert(make_pair(new_index, std::move(data)));
    }

	 // 从unassembled_strings中取出合理的（这里的合理指有序取，所以顺序遍历从unassembled_strings中取和上一层未组装的地方）插入发送缓冲区
   for (auto iter = unassembled_substrings_.begin(); iter != unassembled_substrings_.end();) {
        auto &[sub_index, sub_data] = *iter;
        if (sub_index == unassembled_index_) {
          	// 找到了
            const uint64_t prev_bytes_pushed = output.bytes_pushed();
            output.push(sub_data);
            const uint64_t bytes_pushed = output.bytes_pushed();
            // 但是可能装不下
            // which case can go this if condition path ? when the Writer has no available space to store data.
            if (bytes_pushed != prev_bytes_pushed + sub_data.size()) {
                // Cannot push all data, we need to reserve the un-pushed part.
                const uint64_t pushed_length = bytes_pushed - prev_bytes_pushed;
               // 已经装进去的 
                unassembled_index_ += pushed_length;
               // 未装进去的
                unassembled_bytes_ -= pushed_length;
               // 把没有装进去的放回缓存区（unassembled_substrings_）
                unassembled_substrings_.insert(make_pair(unassembled_index_, sub_data.substr(pushed_length)));
                // Don't forget to remove the previous incompletely transferred data
                unassembled_substrings_.erase(sub_index);
                break;
            }
            unassembled_index_ += sub_data.size();
            unassembled_bytes_ -= sub_data.size();
            unassembled_substrings_.erase(sub_index);
            iter = unassembled_substrings_.find(unassembled_index_);
        } else {
            break; // No need to do more. Data has been discontinuous.
        }
    }
    
    if (is_closed()) {
        // if it is the last substring and bytes_pending === 0, then close
        output.close();
    }
```



## TCPReceiver

这一节最好合上一节来看

```c++
#include "tcp_receiver.hh"

using namespace std;

void TCPReceiver::receive(TCPSenderMessage message, Reassembler &reassembler, Writer &inbound_stream)
{
  	// 没有设置SYN，此时不应该接受数据
    if (!set_syn_) {
      	// 且当前报道文也不是SYN的
        if (!message.SYN) {
            return; // drop all data if SYN isn't received
        }
        // 当前报道文是SYN的，设置ISN：随机的32位数字
        isn_ = message.seqno; // FIN occupied one seqno
        set_syn_ = true; // 设置SYN标志位
    }

    // inbound_stream.bytes_pushed()即unassembled_index, + 1即为下一个期待接入的序号（first_index），需要基于ISN转为abs_seqno => stram_index
    // stream_index = abs_seqno - 1 + SYN
    const uint64_t checkpoint = inbound_stream.bytes_pushed() + 1;
    const uint64_t abs_seqno = message.seqno.unwrap(isn_, checkpoint);
    // unwrap function starts from isn_, which occupies one seqno.
    // We calculate one index more so we need to minus it.
    // But if SYN exists in this message, compensation is needed.
    const uint64_t stream_index = abs_seqno - 1 + message.SYN;
    // 调用上一节实现的insert方法
    reassembler.insert(stream_index, message.payload.release(), message.FIN, inbound_stream);
}

// receive调用结束了之后调用
TCPReceiverMessage TCPReceiver::send(const Writer &inbound_stream) const
{
    TCPReceiverMessage recv_msg {};

    // ws为接受方的缓存里的available_capacity，
    const uint16_t window_size
        = inbound_stream.available_capacity() > UINT16_MAX ? UINT16_MAX : inbound_stream.available_capacity();
    if (!set_syn_) {
        return {std::optional<Wrap32> {}, window_size};
    }
    // add one ISN(SYN) length，ackno为下一个期待的序列号
    uint64_t abs_ackno_offset = inbound_stream.bytes_pushed() + 1;
    if (inbound_stream.is_closed()) {
        abs_ackno_offset++; // add one FIN
    }
    recv_msg.ackno = isn_ + abs_ackno_offset;
    recv_msg.window_size = window_size;

    return recv_msg;
}
```





- 绝对序列号是为了不让别人轻易猜到，但是为什么用32位？
- 为什么`const uint64_t stream_index = abs_seqno - 1 + message.SYN;`



## TCPSender

```c++
#include "tcp_sender.hh"
#include "tcp_config.hh"

#include <iostream>
#include <random>

using namespace std;

/* TCPSender constructor (uses a random ISN if none given) */
TCPSender::TCPSender(uint64_t initial_RTO_ms, optional<Wrap32> fixed_isn)
    : isn_(fixed_isn.value_or(Wrap32 {random_device()()})), initial_RTO_ms_(initial_RTO_ms)
{}

uint64_t TCPSender::sequence_numbers_in_flight() const { return outstanding_seqno_; }

uint64_t TCPSender::consecutive_retransmissions() const { return consecutive_retransmission_times_; }

optional<TCPSenderMessage> TCPSender::maybe_send()
{
    if (!segments_out_.empty() && set_syn_) {
        TCPSenderMessage segment = std::move(segments_out_.front());
        segments_out_.pop();
        return segment;
    }

    return nullopt;
}

void TCPSender::push(Reader &outbound_stream)
{
  	// ws默认从receiver的返回结果中取，娶不到默认为1
    const uint64_t curr_window_size = window_size_ ? window_size_ : 1;
    // 填充
    while (curr_window_size > outstanding_seqno_) {
        TCPSenderMessage msg;

        if (!set_syn_) {
            msg.SYN = true;
            set_syn_ = true;
        }

        msg.seqno = get_next_seqno();
        const uint64_t payload_size
            = min(TCPConfig::MAX_PAYLOAD_SIZE, curr_window_size - outstanding_seqno_ - msg.SYN);
        std::string payload = std::string(outbound_stream.peek()).substr(0, payload_size);
        outbound_stream.pop(payload_size);

        if (!set_fin_ && outbound_stream.is_finished()
            && payload.size() + outstanding_seqno_ + msg.SYN < curr_window_size) {
            msg.FIN = true;
            set_fin_ = true;
        }

        msg.payload = Buffer(std::move(payload));

        // no data, stop sending
        if (msg.sequence_length() == 0) {
            break;
        }

        // no outstanding segments, restart timer
        if (outstanding_seg_.empty()) {
            RTO_timeout_ = initial_RTO_ms_;
            timer_ = 0;
        }

        // 还没发送，放入未确认缓存
        segments_out_.push(msg);
				
      	// 当然，未确认的序列号也要相应增加
        outstanding_seqno_ += msg.sequence_length();
        // { next_abs_seqno_: msg }
      	outstanding_seg_.insert(std::make_pair(next_abs_seqno_, msg));
        // 增加
      	next_abs_seqno_ += msg.sequence_length();

       // 如果是FIN报道文，直接结束
        if (msg.FIN) {
            break;
        }
    }
}

TCPSenderMessage TCPSender::send_empty_message() const
{
    TCPSenderMessage segment;
    segment.seqno = get_next_seqno();

    return segment;
}

// receive => push => send
void TCPSender::receive(const TCPReceiverMessage &msg)
{
    if (!msg.ackno.has_value()) {
        ;
    } else {
        const uint64_t recv_abs_seqno = msg.ackno.value().unwrap(isn_, next_abs_seqno_);
        if (recv_abs_seqno > next_abs_seqno_) {
            // Impossible, we couldn't transmit future data
            return;
        }

        for (auto iter = outstanding_seg_.begin(); iter != outstanding_seg_.end();) {
            const auto &[abs_seqno, segment] = *iter;
          	// 如果当前的未发送的片段的序列号小于已经确认的，说明是已经确认了
            if (abs_seqno + segment.sequence_length() <= recv_abs_seqno) {
                outstanding_seqno_ -= segment.sequence_length();
                iter = outstanding_seg_.erase(iter);
                // reset RTO and if outstanding data is not empty, start timer，为什么要重新计时？
                RTO_timeout_ = initial_RTO_ms_;
                if (!outstanding_seg_.empty()) {
                    timer_ = 0;
                }
            } else {
                break;
            }
        }
        consecutive_retransmission_times_ = 0;
    }
    window_size_ = msg.window_size;
}

void TCPSender::tick(const size_t ms_since_last_tick)
{
    timer_ += ms_since_last_tick;
    auto iter = outstanding_seg_.begin();
    if (timer_ >= RTO_timeout_ && iter != outstanding_seg_.end()) {
        const auto &[abs_seqno, segment] = *iter;
        if (window_size_ > 0) {
            RTO_timeout_ *= 2;
        }
        timer_ = 0;
        consecutive_retransmission_times_++;
        segments_out_.push(segment);
    }
}
```





tick可以理解为心脏跳动，每隔一段时间自动调用，理想情况下这个时间间隔是一样的（但实际是这样吗？），而基于tick方法可以设置一个定时器，比如说，当开始发送的时候，定时器标志位为true，并且timer = 0，每次调用tick方法都会累计timer的值，如果segment发送并在规定时间内返回，则timer reset为0，同时处理掉缓存区那些已经确认的segment；与此同时，由于tick定期调用，如果发现timer超时了，则重新发送最旧的没有收到对应ack的segment；

> 这里有个细节问题：当收到有效的ackno的时候，始终reset timer，很明显不会触发重新发送，这个时候有没有可能实际上之前发送的某个segment超时了呢？然而没触发超时。
>
> 这和超时时间的设置有很大关系，假设当前发送的sgement丢失了，然后Sender调用receive，此时由于没有正确的ackno并不会reset 计时器，继而send下一个segment，并收到响应（这个时候前一个timer的时间就达到了>2RTT），在还没调用本次receive方法之前，就触发了超时重传，
>
> seqno和ackseqno究竟是什么
>
> tcp是以segment为单位进行发送和接收的，会丢失segment
>
> 当发生segment丢失，接收方做什么？
>
> 不会调用receive方法，如果下一个响应到来，对该segment调用receive方法，



## TCP Connection

TCPConnection的一个重要功能是决定TCP 连接什么时候完全结束。当TCP连接完全结束时，停止对任何接收到的segment回复ackno，并且active()方法返回false 。

TCP连接有两种关闭的方法：

不干净的关闭：TCPConnection发送或接收到一个首部字段中的RST标志位被设置的segment 。这种情况下，inbound和outbound的ByteStream都处于error state，并且active()方法可以马上返回false 。

干净的关闭：在没有error的情况下关闭（active()=false）。这种情况可以尽可能地保证两个字节流都完全可靠地交付到接收对等方。

由于两将军问题，不可能保证对等方都能完全干净关闭连接，但是可以非常接近。

从一个对等设备的角度来看，对其与“远程”对等设备的连接进行干净关闭有四个先决条件，条件1保证了输入流被读取干净了，条件2和3保证了输出流被对等方读取干净了。条件4也是关于输入流的，保证了输入流的正常关闭。

输入流被完全确认（StreamReassembler为空）并且结束（收到了FIN）

输出流被应用层结束（调用ByteStream的end_input()方法），并且被完全发送出去（ByteStream为空），首部字段包括FIN的segment也被发送出去。

输出流被对等方完全确认（对方的StreamReassembler为空，实际上要求本地的_outstanding_segments为空）

本地TCPConnection需要让对等方满足条件3。有两种方式：

选择A：在两个流都已经结束后 linger 一段时间：

本地TCPConnection确认了整个输入流，但是难以确认对等端是否知道自己确认了整个输入流，即对等端是否收到ack（因为对等端不会对本地发送的ack进行ack ）。如果不能确保对等端收到ack，也就不能确保对等端的_outstanding_segments为空，那么对等端就有可能不停地重传无法得到本地确认的segment，输入流永远无法关闭。

我们可以让本地的TCPConnection等待一段时间，如果对等端没有重传任何东西，那么就可以相信对等端收到了ack。

具体地，当一个连接满足条件1到条件3，并且在收到对等端最后一个segment后，等待了至少 初始重传超时时间（_cfg.rt_timeout）的十倍，才能断开。

这意味着TCPConnection需要保持alive一段时间，保持对本地端口号的独占声明并可能发送 acks 以响应传入的segment，即使在 TCPSender 和 TCPReceiver 完全完成其工作并且两个流都已经结束了。

选择B：被动关闭

如果在TCPConnection发送FIN之前，TCPConnection的输入流就结束了（收到了FIN），那么这个TCPConnection在两个流结束后不需要 linger 。（因为FIN在发送ack之后，所以FIN的seqno大于之前发送的ack，所以对方对FIN的确认，就相当于确认了之前发送的所有ack


在lab 4中，我们将创建总体模块，称为TCP connection，该模块将TCPSender和TCPReceiver结合起来。

我们的TCP segment可以封装到用户(TCP-In-UDP)或IP(TCP/IP)数据报的有效载荷中。

https://blog.csdn.net/qq_45698833/article/details/120536017

```c++
#include "tcp_connection.hh"

#include <iostream>

using namespace std;

size_t TCPConnection::remaining_outbound_capacity() const { return _sender.stream_in().remaining_capacity(); }

size_t TCPConnection::bytes_in_flight() const { return _sender.bytes_in_flight(); }

size_t TCPConnection::unassembled_bytes() const { return _receiver.unassembled_bytes(); }

size_t TCPConnection::time_since_last_segment_received() const { return _time_since_last_segment_received;}

bool TCPConnection::active() const { return _active; }

// 由操作系统调用，接收从UDP或IP数据报中的解封装的TCPsegment
void TCPConnection::segment_received(const TCPSegment &seg) { 
    // 如果连接断开了，不接收任何segment
    if(!_active){
        return;
    }
    // 接收到一个segment，重置计数
    _time_since_last_segment_received = 0;
    // 被动建立连接的一方可能处于的状态,处于listen状态
    // 没有收到过任何segment，也没有发送过任何segment,
    if(!_receiver.ackno().has_value() && _sender.next_seqno_absolute() == 0){
        // 只接收syn
        if(!seg.header().syn){
            return;
        }
        _receiver.segment_received(seg);
        // 收到对方的syn，就发送SYN与对方建立连接，处于SYN_RECV状态
        // 三次握手的阶段二
        connect();
        return;
    }
    // 主动建立连接的一方可能处于的状态,处于SYN_SENT状态，三次握手的阶段一
    // 发送出去的流没有得到确认，也没有收到过对方的segment。
    if(_sender.next_seqno_absolute() > 0 && _sender.bytes_in_flight() == _sender.next_seqno_absolute() && 
       !_receiver.ackno().has_value()){
        // 如果有效载荷不为0，不符合SYN，直接丢弃
        if(seg.payload().size() ){
            return;
        }
        // 如果ack等于0，则双方同时发起了建立连接
        if(!seg.header().ack){
            if(seg.header().syn){
                _receiver.segment_received(seg);
                // 发送空的segment，以返回ack
                _sender.send_empty_segment();
            }
            return;
        }
        // 如果syn=1，ack=1，rst=1，则关闭连接
        if(seg.header().rst){
            _receiver.stream_out().set_error();
            _sender.stream_in().set_error();
            _active = false;
            return;
        }
    }
    // 如果syn=1，ack=1，rst!=1，或者其他情况
    _receiver.segment_received(seg);
    _sender.ack_received(seg.header().ackno,seg.header().win);
    // 发送确认的报文，进入ESTABLISHED状态，连接建立。处于三次握手的第三阶段
    if (_sender.stream_in().buffer_empty() && seg.length_in_sequence_space())
        _sender.send_empty_segment();
    if (seg.header().rst) {
        _sender.send_empty_segment();
        unclean_shutdown();
        return;
    }
    send_sender_segments();
}

size_t TCPConnection::write(const string &data) {
    if(data.size() == 0){
        return 0;
    }
    // 向TCPSender的ByteStream中写入数据
    size_t write_size = _sender.stream_in().write(data);
    _sender.fill_window();
    // 对TCPSender中的segment设置ackno和windowsize,再发送给对等端
    send_sender_segments();
    return write_size;
}

// 此方法被OS周期性调用
void TCPConnection::tick(const size_t ms_since_last_tick) {
    if(!_active){
        return;
    }
    _time_since_last_segment_received += ms_since_last_tick;
    // 告知TCPSender过去的时间
    _sender.tick(ms_since_last_tick);
    // 如果连续重传的次数超过上限，则强制关闭连接
    if(_sender.consecutive_retransmissions() > TCPConfig::MAX_RETX_ATTEMPTS){
        unclean_shutdown();    
    }
    send_sender_segments();
}

// 结束向TCPConnection中写入，也就是关闭输出流（仍然允许读取输入的数据）
void TCPConnection::end_input_stream() {
    _sender.stream_in().end_input();
    // 发送fin，不能保证这一次能将fin发送出去，因为接收窗口有可能空间不够，ByteStream无法全部发送出去
    _sender.fill_window();
    send_sender_segments();
}

// 主动连接
void TCPConnection::connect() {
    _sender.fill_window();
    send_sender_segments();
}



// 对TCPSender的 _segments_out中的segment设置首部的ackno和windowsize字段，还有ACK标志位
// 再加入到TCPConnection的 _segments_out，真正地将TCPsegment发送出去
void TCPConnection::send_sender_segments(){
    // 此处必须要是引用类型，才能指向_sender中的同一个成员变量,才能对其进行操作
    // std::queue<TCPSegment>&sender_segs_out = _sender.segments_out();

    // 对TCPSender的 _segments_out进行遍历，将所有的segment的头部都加上ackno和windowsize
    // 再发送出去
    while(!_sender.segments_out().empty()){
        TCPSegment seg = _sender.segments_out().front();
        _sender.segments_out().pop();
        // 只有当ackno()的返回值非空时，才需要加上
        if(_receiver.ackno().has_value()){
            seg.header().ack = true;
            seg.header().ackno = _receiver.ackno().value();
            seg.header().win = _receiver.window_size();
        }
        // 将segment真正发送出去
        _segments_out.push(seg);
    }
    // 每次发送segment后，都需要判断是否需要干净关闭连接
    clean_shutdown();
    
}
// 不干净的关闭，直接强制关闭连接
// 将输入输出流设置为错误状态
// 将连接的active置为false，向对等方发送rst
void TCPConnection::unclean_shutdown(){
    _receiver.stream_out().set_error();
    _sender.stream_in().set_error();
    _active = false;
    TCPSegment seg = _sender.segments_out().front();
    _sender.segments_out().pop();
    seg.header().ack = true;
    if(_receiver.ackno().has_value()){
        seg.header().ackno = _receiver.ackno().value();
    }
    seg.header().win = _receiver.window_size();
    seg.header().rst = true;
    _segments_out.push(seg);

}

// 干净关闭连接，判断能否干净地关闭连接，
// 判断是否需要在两个流结束后linger一段时间
void TCPConnection::clean_shutdown(){
    // 自己的接收的StreamReassembler（重组缓存）为空
    if(_receiver.stream_out().input_ended()){
        // 如果sender的输出流还没有结束，即ByteStream不为空，fin还没有发送出去
        if(!_sender.stream_in().eof()){
        // 那么需要在两个流结束后linger一段时间
            _linger_after_streams_finish = false;
        // 如果sender发送了fin，且得到了确认
        }else if(_sender.bytes_in_flight() == 0){
            // 那么只有不需要linger或者linger了指定时间后，才能断开连接
            if(!_linger_after_streams_finish || time_since_last_segment_received() >= 10 * _cfg.rt_timeout){
                _active = false;
            }
        }
    }
 

  TCPConnection::~TCPConnection() {
    try {
        if (active()) {
            cerr << "Warning: Unclean shutdown of TCPConnection\n";
            _sender.send_empty_segment();
            unclean_shutdown();
            // Your code here: need to send a RST segment to the peer
        }
    } catch (const exception &e) {
        std::cerr << "Exception destructing TCP FSM: " << e.what() << std::endl;
    }
}

```

```
发送：TCPConnection::write => send_sender_segments: 遍历Sender的segments，包装其头部，放到一段内存缓存里 => clean_shutdown: 看下是否需要关闭

接收：TCPConnection::segment_received => send_sender_segments 放到一段内存缓存里
```



## IP



```c++
#include "network_interface.hh"

#include "arp_message.hh"
#include "ethernet_frame.hh"

using namespace std;

// ethernet_address: Ethernet (what ARP calls "hardware") address of the interface
// ip_address: IP (what ARP calls "protocol") address of the interface
// cppcheck-suppress uninitMemberVar
NetworkInterface::NetworkInterface(const EthernetAddress &ethernet_address, const Address &ip_address)
    : ethernet_address_(ethernet_address), ip_address_(ip_address)
{
    cerr << "DEBUG: Network interface has Ethernet address " << to_string(ethernet_address_) << " and IP address "
         << ip_address.ip() << "\n";
}

// dgram: the IPv4 datagram to be sent
// next_hop: the IP address of the interface to send it to (typically a router or default gateway, but
// may also be another host if directly connected to the same network as the destination)

// Note: the Address type can be converted to a uint32_t (raw 32-bit IP address) by using the
// Address::ipv4_numeric() method.
void NetworkInterface::send_datagram(const InternetDatagram &dgram, const Address &next_hop)
{
    // 下一跳的地址
    const uint32_t addr_numeric = next_hop.ipv4_numeric();

    /* ARP Table has stored the mapping info, we send the datagram directly */
  	// 如果ARP Table包含下一跳的地址，直接
    if (arp_table_.contains(addr_numeric)) {
        EthernetFrame eth_frame;
      	// 当前物理机以太网地址
        eth_frame.header.src = ethernet_address_;
      	// 目标物理机以太网地址
        eth_frame.header.dst = arp_table_.at(addr_numeric).eth_addr;
        eth_frame.header.type = EthernetHeader::TYPE_IPv4;
      	// 序列化
        eth_frame.payload = serialize(dgram);
      	// 发乳待发送缓冲区
        outbound_frames_.push(eth_frame);
    } else {
      	// 没有则发送arp请求
        /* ARP Table has no such mapping and we haven't send an ARP request for target ip */
        if (arp_requests_lifetime_.find(addr_numeric) == arp_requests_lifetime_.end()) {
            // next hop ipv4 addr is not contained in the arp requests waiting list
            ARPMessage arp_msg;
            arp_msg.opcode = ARPMessage::OPCODE_REQUEST;
            arp_msg.sender_ip_address = ip_address_.ipv4_numeric();
            arp_msg.sender_ethernet_address = ethernet_address_;
            arp_msg.target_ip_address = addr_numeric;
            arp_msg.target_ethernet_address = {/* empty */};

            EthernetFrame arp_eth_frame;
            arp_eth_frame.header.src = ethernet_address_;
            arp_eth_frame.header.dst = ETHERNET_BROADCAST;
            arp_eth_frame.header.type = EthernetHeader::TYPE_ARP;
            arp_eth_frame.payload = serialize(arp_msg);
            outbound_frames_.push(arp_eth_frame);
						
          	// 广播请求
            arp_requests_lifetime_.emplace(std::make_pair(addr_numeric, ARP_REQUEST_DEFAULT_TTL));
        }
        // We need to store the datagram in the list. After we know the eth addr, we can queue
        // the corresponding dgrams.
      	// 广播下一跳的以太网地址的 ARP 请求， 并将 IP 报文放入队列中待 ARP 回复收到后能将其发送出去。
        arp_datagrams_waiting_list_.emplace_back(std::pair {next_hop, dgram});
    }
}

// frame: the incoming Ethernet frame
optional<InternetDatagram> NetworkInterface::recv_frame(const EthernetFrame &frame)
{
  	// 如果当前frame的目标物理机不是当前物理机且也不是广播请求的目标物理机，直接返回
    if (frame.header.dst != ethernet_address_ && frame.header.dst != ETHERNET_BROADCAST) {
        return nullopt;
    }

    /* IP datagrams，是IP数据报，反序列数据并直接返回datagram */
    if (frame.header.type == EthernetHeader::TYPE_IPv4) {
        InternetDatagram datagram;
        if (not parse(datagram, frame.payload)) {
            // printf("[NetworkInterface ERROR]: 'recv_frame' IPV4 parse error\n");
            return nullopt;
        }
        return datagram;
    }

    /* ARP datagrams，广播数据报 */
    if (frame.header.type == EthernetHeader::TYPE_ARP) {
        ARPMessage arp_msg;
        if (not parse(arp_msg, frame.payload)) {
            printf("[NetworkInterface ERROR]: 'recv_frame' ARP parse error\n");
            return nullopt;
        }

        const bool is_arp_request = arp_msg.opcode == ARPMessage::OPCODE_REQUEST
                                    && arp_msg.target_ip_address == ip_address_.ipv4_numeric();
        // 如果是广播请求，则回复即可
      	if (is_arp_request) {
            ARPMessage arp_reply_msg;
            arp_reply_msg.opcode = ARPMessage::OPCODE_REPLY;
            arp_reply_msg.sender_ip_address = ip_address_.ipv4_numeric();
            arp_reply_msg.sender_ethernet_address = ethernet_address_;
            arp_reply_msg.target_ip_address = arp_msg.sender_ip_address;
            arp_reply_msg.target_ethernet_address = arp_msg.sender_ethernet_address;

            EthernetFrame arp_reply_eth_frame;
            arp_reply_eth_frame.header.src = ethernet_address_;
            arp_reply_eth_frame.header.dst = arp_msg.sender_ethernet_address;
            arp_reply_eth_frame.header.type = EthernetHeader::TYPE_ARP;
            arp_reply_eth_frame.payload = serialize(arp_reply_msg);
            outbound_frames_.push(arp_reply_eth_frame);
        }
				
      	// 如果是响应，对应上面send发送的广播数据报
        const bool is_arp_response
            = arp_msg.opcode == ARPMessage::OPCODE_REPLY && arp_msg.target_ethernet_address == ethernet_address_;

        // we can get arp info from either ARP request or ARP reply
        if (is_arp_request || is_arp_response) {
          	// 更新arp_table
            arp_table_.emplace(std::make_pair(arp_msg.sender_ip_address,
                                              arp_t {arp_msg.sender_ethernet_address, ARP_DEFAULT_TTL}));
            // delete arp datagrams waiting list，并且发送该存储的请求
            for (auto iter = arp_datagrams_waiting_list_.begin(); iter != arp_datagrams_waiting_list_.end();) {
                const auto &[ipv4_addr, datagram] = *iter;
                if (ipv4_addr.ipv4_numeric() == arp_msg.sender_ip_address) {
                    send_datagram(datagram, ipv4_addr);
                    iter = arp_datagrams_waiting_list_.erase(iter);
                } else {
                    iter++;
                }
            }
            arp_requests_lifetime_.erase(arp_msg.sender_ip_address);
        }
    }

    return nullopt;
}

// ms_since_last_tick: the number of milliseconds since the last call to this method
void NetworkInterface::tick(const size_t ms_since_last_tick)
{
    /* delete expired ARP items in ARP Table */
    // FIXME: Don't use 'iter++' if we have erase current iter's data!
    for (auto iter = arp_table_.begin(); iter != arp_table_.end(); /* nop */) {
        auto &[ipv4_addr_numeric, arp] = *iter;
        if (arp.ttl <= ms_since_last_tick) {
            iter = arp_table_.erase(iter);
        } else {
            arp.ttl -= ms_since_last_tick;
            iter++;
        }
    }

    /* delete expired ARP requests */
    for (auto &[ipv4_addr, arp_ttl] : arp_requests_lifetime_) {
        /* resent ARP request if this request has expired，使得任何已经过期的 IP 地址到 Ethernet 地址的映射失效 */ 
        if (arp_ttl <= ms_since_last_tick) {
            ARPMessage arp_msg;
            arp_msg.opcode = ARPMessage::OPCODE_REQUEST;
            arp_msg.sender_ip_address = ip_address_.ipv4_numeric();
            arp_msg.sender_ethernet_address = ethernet_address_;
            arp_msg.target_ip_address = ipv4_addr;
            arp_msg.target_ethernet_address = {/* empty */};

            EthernetFrame arp_eth_frame;
            arp_eth_frame.header.src = ethernet_address_;
            arp_eth_frame.header.dst = ETHERNET_BROADCAST;
            arp_eth_frame.header.type = EthernetHeader::TYPE_ARP;
            arp_eth_frame.payload = serialize(arp_msg);
            outbound_frames_.push(arp_eth_frame);

            /* reset ARP ttl for this component */
            arp_ttl = ARP_REQUEST_DEFAULT_TTL;
        } else {
            arp_ttl -= ms_since_last_tick;
        }
    }
}

optional<EthernetFrame> NetworkInterface::maybe_send()
{
    if (!outbound_frames_.empty()) {
        EthernetFrame eth_frame = std::move(outbound_frames_.front());
        outbound_frames_.pop();
        return eth_frame;
    }

    return nullopt;
}
```



## IP Router





## Together





