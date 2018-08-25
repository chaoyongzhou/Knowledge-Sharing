
# 来源

高效低延迟UDP传输项目Aeron： [https://github.com/real-logic/Aeron](https://github.com/real-logic/Aeron)
 
## Design Overview


Currently out of date and needs rewritten
So how does Aeron implement its Protocol? This diagram shows a the communication for a channel with a single producer and a single consumer. Channels support multicast through multiple consumers.

![图 1](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/architecture.png)

0 - Aeron uses an underlying unreliable media protocol, which could be anything, eg: UDP or Infiniband.
1 - Aeron has a Media Driver which implements support for these protocols. It can operate within the same process as the client code or outside. Media drivers can be implemented using Java, C or even FPGAs and sit within either user or kernel space.
2 - Communication between the Client API/Stack and the Media Driver is implemented as a messaging protocol on a series of ring buffers. Each channel requires two ring buffers, one for sending messages and one for receiving messages. This also includes a control channel for administrative/internal communication. These use memory mapped file based shared memory.
3 - The Client API/Stack sits in process. After configuration, the client stack is abstract of the Media Driver. This presents an API to the client code to send or receive messages.
4 - Multiple client threads can be reading from or write to the channel at given moment in time.


## unicast

![图 2](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/%E5%8D%95%E6%92%AD%E4%BA%A4%E4%BA%92%E6%B5%81%E7%A8%8B.png)


## multicast

![图 3](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/%E7%BB%84%E6%92%AD%E4%BA%A4%E4%BA%92%E6%B5%81%E7%A8%8B.png)

aeron over udp:

* unicast mode

* multicast mode

* multi-destination mode

## receiver offer
aeron\_driver\_receiver\_proxy\_offer： 将一条指令放入receiver的（指令）队列中

	void aeron_driver_receiver_proxy_offer(aeron_driver_receiver_proxy_t *receiver_proxy, void *cmd)
	{
	   while (aeron_spsc_concurrent_array_queue_offer(receiver_proxy->command_queue, cmd) != AERON_OFFER_SUCCESS)
	   {
	       aeron_counter_ordered_increment(receiver_proxy->fail_counter, 1);
	       sched_yield();
	   }
	}

指令有：

1, delete cmd:  

	放入指令队列的封装接口：aeron_driver_receiver_proxy_on_delete_create_publication_image_cmd
	
	指令执行接口： aeron_command_on_delete_cmd

2, add endpoint:

	放入指令队列的封装接口： aeron_driver_receiver_proxy_on_add_endpoint
	指令执行接口： aeron_driver_receiver_on_add_endpoint

3, remove endpoint:

	放入指令队列的封装接口： aeron_driver_receiver_proxy_on_remove_endpoint
	指令执行接口： aeron_driver_receiver_on_remove_endpoint

4, add subscription:
	
	放入指令队列的封装接口： aeron_driver_receiver_proxy_on_add_subscription
	指令执行接口： aeron_driver_receiver_on_add_subscription

5, remove subscription:

	放入指令队列的封装接口： aeron_driver_receiver_proxy_on_remove_subscription
	指令执行接口： aeron_driver_receiver_on_remove_subscription

6, add publication image:
	
	放入指令队列的封装接口： aeron_driver_receiver_proxy_on_add_publication_image
	指令执行接口： aeron_driver_receiver_on_add_publication_image

7, remove publication image:
	
	放入指令队列的封装接口： aeron_driver_receiver_proxy_on_remove_publication_image
	指令执行接口： aeron_driver_receiver_on_remove_publication_image

8, remove cooldown:
	
	放入指令队列的封装接口： aeron_driver_receiver_proxy_on_remove_cooldown
	指令执行接口： aeron_driver_receiver_on_remove_cooldown


注： 这些指令是由conductor发给receiver的


## sender offer
aeron\_driver\_sender\_proxy\_offer： 将一条指令放入sender的（指令）队列中


	void aeron_driver_sender_proxy_offer(aeron_driver_sender_proxy_t *sender_proxy, void *cmd)
	{
	   while (aeron_spsc_concurrent_array_queue_offer(sender_proxy->command_queue, cmd) != AERON_OFFER_SUCCESS)
	   {
	       aeron_counter_ordered_increment(sender_proxy->fail_counter, 1);
	       sched_yield();
	   }
	}

指令有：

1, add endpoint:  

	放入指令队列的封装接口： aeron_driver_sender_proxy_on_add_endpoint
	指令执行接口： aeron_driver_sender_on_add_endpoint

2, remove endpoint:  

	放入指令队列的封装接口： aeron_driver_sender_proxy_on_remove_endpoint
	指令执行接口： aeron_driver_sender_on_remove_endpoint

3, add publication:  
   
	放入指令队列的封装接口： aeron_driver_sender_proxy_on_add_publication
	指令执行接口： aeron_driver_sender_on_add_publication

4, remove publication:  
   
	放入指令队列的封装接口： aeron_driver_sender_proxy_on_remove_publication
	指令执行接口： aeron_driver_sender_on_remove_publication

5, add destination:  
   
	放入指令队列的封装接口： aeron_driver_sender_proxy_on_add_destination
	指令执行接口： aeron_driver_sender_on_add_destination

6, remove destination:  
   
	放入指令队列的封装接口： aeron_driver_sender_proxy_on_remove_destination
	指令执行接口： aeron_driver_sender_on_remove_destination

注： 这些指令是由conductor发给sender的


endpoint和destination的关系： endpoint由一组destination构成，也就是说，endpoint维护着一个destination数组。
参见：aeron\_driver\_sender\_on\_add\_destination


RTTM尚未支持。

## conductor offer
aeron\_driver\_conductor\_proxy\_offer： 将一条指令放入conductor的（指令）队列中


	void aeron_driver_conductor_proxy_offer(aeron_driver_conductor_proxy_t *conductor_proxy, void *cmd)
	{
	    while (aeron_mpsc_concurrent_array_queue_offer(conductor_proxy->command_queue, cmd) != AERON_OFFER_SUCCESS)
	    {
	        aeron_counter_ordered_increment(conductor_proxy->fail_counter, 1);
	        sched_yield();
	    }
	}

客户端在指令放入mpsc ring buffer共享内存中（conductor->to\_driver\_commands），conductor从共享内存中取出指令并执行。

指令有：

1, add publication:  

    指令执行接口：aeron_driver_conductor_on_add_network_publication

2, add exclusive publication:

    指令执行接口： aeron_driver_conductor_on_add_network_publication

3, remove publication:

    指令执行接口：aeron_driver_conductor_on_remove_publication

4, add subscription:

    指令执行接口：aeron_driver_conductor_on_add_network_subscription

5, remove subscription:

    指令执行接口：aeron_driver_conductor_on_remove_subscription

6, client keepalive:

    指令执行接口：aeron_driver_conductor_on_client_keepalive

7, add destination:

    指令执行接口：aeron_driver_conductoor_on_add_destination

8, remove destination:

    指令执行接口：aeron_driver_conductoor_on_remove_destination

9, add counter:

    指令执行接口：aeron_driver_conductor_on_add_counter

10, remove counter:
        
    指令执行接口：aeron_driver_conductor_on_remove_counter

11, client close:

    指令执行接口：aeron_driver_conductor_on_client_close

receiver\_proxy和sender\_proxy面向conductor，即
receiver通过receiver\_proxy与conductor交互
sender通过sender\_proxy与conductor交互
（conductor也有conductor proxy）



	int aeron_static_window_congestion_control_strategy_supplier(
	   aeron_congestion_control_strategy_t **strategy,
	   const char *channel,
	   int32_t stream_id,
	   int32_t session_id,
	   int64_t registration_id,
	   int32_t term_length,
	   int32_t sender_mtu_length,
	   aeron_driver_context_t *context,
	   aeron_counters_manager_t *counters_manager)
	{
	   aeron_congestion_control_strategy_t *_strategy;
	
	   if (aeron_alloc((void **)&_strategy, sizeof(aeron_congestion_control_strategy_t)) < 0 ||
	       aeron_alloc((void **)&_strategy->state, sizeof(aeron_static_window_congestion_control_strategy_state_t)) < 0)
	   {
	       return -1;
	   }
	
	   _strategy->should_measure_rtt = aeron_static_window_congestion_control_strategy_should_measure_rtt;
	   _strategy->on_rttm = aeron_static_window_congestion_control_strategy_on_rttm;
	   _strategy->on_track_rebuild = aeron_static_window_congestion_control_strategy_on_track_rebuild;
	   _strategy->initial_window_length = aeron_static_window_congestion_control_strategy_initial_window_length;
	   _strategy->fini = aeron_static_window_congestion_control_strategy_fini;
	
	   aeron_static_window_congestion_control_strategy_state_t *state = _strategy->state;
	   const int32_t initial_window_length = (int32_t)context->initial_window_length;
	   const int32_t max_window_for_term = term_length / 2;
	
	   state->window_length = max_window_for_term < initial_window_length ? max_window_for_term : initial_window_length;
	
	   *strategy = _strategy;
	   return 0;
	}
	
	bool aeron_static_window_congestion_control_strategy_should_measure_rtt(void *state, int64_t now_ns)
	{
	   return false;
	}
	
	void aeron_static_window_congestion_control_strategy_on_rttm(
	   void *state, int64_t now_ns, int64_t rtt_ns, struct sockaddr_storage *source_address)
	{
	}
	
sender的收只发生在control channel上，根据指令进行相应动作。
sender收到控制指令后的处理入口：aeron\_send\_channel\_endpoint\_dispatch

sender收到NAK指令的处理：
aeron\_send\_channel\_endpoint\_on\_nak  => aeron\_network\_publication\_on\_nak

	void aeron_network_publication_on_nak(
	   aeron_network_publication_t *publication, int32_t term_id, int32_t term_offset, int32_t length)
	{
	   aeron_retransmit_handler_on_nak(
	       &publication->retransmit_handler,
	       term_id,
	       term_offset,
	       (size_t)length,
	       (size_t)(publication->term_length_mask + 1),
	       publication->nano_clock(),
	       aeron_network_publication_resend,
	       publication);
	}
	
	int aeron_retransmit_handler_on_nak(
	   aeron_retransmit_handler_t *handler,
	   int32_t term_id,
	   int32_t term_offset,
	   size_t length,
	   size_t term_length,
	   int64_t now_ns,
	   aeron_retransmit_handler_resend_func_t resend,
	   void *resend_clientd)
	{
	   int result = 0;
	
	   if (!aeron_retransmit_handler_is_invalid(handler, term_offset, term_length))
	   {
	       const int64_t key = aeron_int64_to_ptr_hash_map_compound_key(term_id, term_offset);
	
	       if (NULL == aeron_int64_to_ptr_hash_map_get(&handler->active_retransmits_map, key) && // 仅当缺失的（term_id, term_offset）不在表中时，才会重传
	           handler->active_retransmits_map.size < AERON_RETRANSMIT_HANDLER_MAX_RETRANSMITS)
	       {
	           aeron_retransmit_action_t *action = aeron_retransmit_handler_assign_action(handler);
	
	           if (NULL == action)
	           {
	               aeron_set_err(EINVAL, "%s", "could not assign retransmit action");
	               return -1;
	           }
	
	           const size_t term_length_left = term_length - term_offset;
	
	           action->term_id = term_id;
	           action->term_offset = term_offset;
	           action->length = length < term_length_left ? length : term_length_left;
	
	           result = resend(resend_clientd, term_id, term_offset, action->length);
	           action->state = AERON_RETRANSMIT_ACTION_STATE_LINGERING;
	           action->expire_ns = now_ns + handler->linger_timeout_ns;
	
	           if (aeron_int64_to_ptr_hash_map_put(&handler->active_retransmits_map, key, action) < 0) // 入表
	           {
	               int errcode = errno;
	
	               aeron_set_err(errcode, "could not put retransmit handler map: %s", strerror(errcode));
	               return -1;
	           }
	       }
	   }
	
	   return result;
	}

可见，仅当缺失的（term\_id, term\_offset）不在表中时，才会重传并入表，也就是说，如果缺失部分最近重传过，则此时不会再次重传，起到抑制NAK的作用。

那么，什么时候出表呢？答案在接口aeron\_retransmit\_handler\_process\_timeouts中，即仅当重传过了一段时间（超时）后，才会出表。

如果receiver缺失部分依然没有收到，再次向sender发送NAK，那么sender可以让缺失部分再次入表。重复上述动作。

可见，aeron是通过超时的机制，来抑制NAK（去重），降低相同数据重发频度。

哈希表： aeron\_send\_channel\_endpoint\_t::publication\_dispatch_map  { (stream\_id | session\_id, publication) }

也即，一对（stream id， session id）对应一个publication


sender收到STATUS（SM）指令的处理：
aeron\_send\_channel\_endpoint\_on\_status\_message


	void aeron_send_channel_endpoint_on_status_message(
	   aeron_send_channel_endpoint_t *endpoint, uint8_t *buffer, size_t length, struct sockaddr_storage *addr)
	{
	   aeron_status_message_header_t *sm_header = (aeron_status_message_header_t *)buffer;
	   int64_t key_value =
	       aeron_int64_to_ptr_hash_map_compound_key(sm_header->stream_id, sm_header->session_id);
	   aeron_network_publication_t *publication =
	       aeron_int64_to_ptr_hash_map_get(&endpoint->publication_dispatch_map, key_value);
	
	  ...................
	
	   if (NULL != publication)
	   {
	       if (sm_header->frame_header.flags & AERON_STATUS_MESSAGE_HEADER_SEND_SETUP_FLAG) // 如果STATUS携带Setup标志，则设置publication的标志should_send_setup_frame
	       {
	           aeron_network_publication_trigger_send_setup_frame(publication);
	       }
	       else
	       {
	           aeron_network_publication_on_status_message(publication, buffer, length, addr);
	       }
	   }
	}
	
	inline void aeron_network_publication_trigger_send_setup_frame(aeron_network_publication_t *publication)
	{
	   bool is_end_of_stream;
	   AERON_GET_VOLATILE(is_end_of_stream, publication->is_end_of_stream);
	
	   if (!is_end_of_stream)
	   {
	       publication->should_send_setup_frame = true;
	   }
	}
	
如果STATUS携带了Setup标志，则设立publication的标志should\_send\_setup\_frame为true，该标志在接口aeron\_network\_publication\_send中将触发向receiver发送SETUP指令。

	aeron_driver_sender_do_work -> aeron_driver_sender_do_send -> aeron_network_publication_send

如果STATUS没有携带Setup标志，除了调整一下send limit position，貌似也没做啥流控（flow control）: aeron\_max\_flow\_control\_strategy\_on\_sm

sender收到RTTM指令的处理：

aeron\_send\_channel\_endpoint\_on\_rttm => aeron\_network\_publication\_on\_rttm

如果RTTM携带Reply标，则向receiver回一个RTTM（标志清零）。

注意：RTTM在data channel上传输。

注：receiver收到RTTM指令的处理流程类似

publication image：仅在receiver侧，一对（stream id， session id）对应一个image

	int aeron_data_packet_dispatcher_on_setup(
	   aeron_data_packet_dispatcher_t *dispatcher,
	   aeron_receive_channel_endpoint_t *endpoint,
	   aeron_setup_header_t *header,
	   uint8_t *buffer,
	   size_t length,
	   struct sockaddr_storage *addr)
	{
	   aeron_int64_to_ptr_hash_map_t *session_map =
	       aeron_int64_to_ptr_hash_map_get(&dispatcher->session_by_stream_id_map, header->stream_id);
	
	   if (NULL != session_map)
	   {
	       aeron_publication_image_t *image = aeron_int64_to_ptr_hash_map_get(session_map, header->session_id);
	        .........
	   }
	
	   return 0;
	}


receiver收到控制指令后的处理入口：aeron\_receive\_channel\_endpoint\_dispatch

receiver收到SETUP指令的处理：

	aeron_receive_channel_endpoint_on_setup => aeron_data_packet_dispatcher_on_setup => aeron_driver_conductor_proxy_on_create_publication_image_cmd => aeron_driver_conductor_proxy_offer

也就是向conductor发送create publication image指令。



（注意：此处receiver并没有立即向sender回STATUS，后面应该需要一个更大的流程才会发出STATUS）


receiver收到DATA或PAD的处理：

	aeron_receive_channel_endpoint_on_data => aeron_data_packet_dispatcher_on_data


	int aeron_data_packet_dispatcher_on_data(
	   aeron_data_packet_dispatcher_t *dispatcher,
	   aeron_receive_channel_endpoint_t *endpoint,
	   aeron_data_header_t *header,
	   uint8_t *buffer,
	   size_t length,
	   struct sockaddr_storage *addr)
	{
	   aeron_int64_to_ptr_hash_map_t *session_map =
	       aeron_int64_to_ptr_hash_map_get(&dispatcher->session_by_stream_id_map, header->stream_id); // 根据stream id获取session map
	
	   if (NULL != session_map)
	   {
	       aeron_publication_image_t *image = aeron_int64_to_ptr_hash_map_get(session_map, header->session_id); // 根据session id获取 image
	
	       if (NULL != image)
	       {
	           return aeron_publication_image_insert_packet(image, header->term_id, header->term_offset, buffer, length); // 将收到的term数据插入image
	       }
	       else if (NULL == aeron_int64_to_ptr_hash_map_get(
	           &dispatcher->ignored_sessions_map,
	           aeron_int64_to_ptr_hash_map_compound_key(header->session_id, header->stream_id)) &&
	           (((aeron_frame_header_t *)buffer)->flags & AERON_DATA_HEADER_EOS_FLAG) == 0) // 未结束流
	       {
	           return aeron_data_packet_dispatcher_elicit_setup_from_source( // receiver诱导sender发出SETUP
	               dispatcher, endpoint, addr, header->stream_id, header->session_id);
	       }
	   }
	
	   return 0;
	}
	
	int aeron_data_packet_dispatcher_elicit_setup_from_source(
	   aeron_data_packet_dispatcher_t *dispatcher,
	   aeron_receive_channel_endpoint_t *endpoint,
	   struct sockaddr_storage *addr,
	   int32_t stream_id,
	   int32_t session_id)
	{
	   struct sockaddr_storage *control_addr =
	       endpoint->conductor_fields.udp_channel->multicast ? &endpoint->conductor_fields.udp_channel->remote_control : addr;
	
	   if (aeron_int64_to_ptr_hash_map_put(
	       &dispatcher->ignored_sessions_map,
	       aeron_int64_to_ptr_hash_map_compound_key(session_id, stream_id),
	       &dispatcher->tokens.pending_setup_frame) < 0)
	   {
	       int errcode = errno;
	
	       aeron_set_err(errcode, "could not aeron_data_packet_dispatcher_remove_publication_image: %s", strerror(errcode));
	       return -1;
	   }
	
	   if (aeron_receive_channel_endpoint_send_sm( // 向sender发送STATUS，并携带Setup标
	       endpoint, control_addr, stream_id, session_id, 0, 0, 0, AERON_STATUS_MESSAGE_HEADER_SEND_SETUP_FLAG) < 0)
	   {
	       return -1;
	   }
	
	   return aeron_driver_receiver_add_pending_setup(dispatcher->receiver, endpoint, session_id, stream_id, NULL); // 先创建一个setup entry，等待sender的SETUP指令到来
	}
	
首先，receiver根据（stream id, session id）查询到image。

如果image存在，则将term数据（term id, term offset, buffer, length）插入到image。

如果image不存在，且未携带EOS标（End Of Stream），即receiver收到一个新流的DATA或PAD，则向

sender发送STATUS（携带Setup标），诱导sender发出SETUP。注意，此时receiver收到的DATA或PAD无意义，被忽略掉。

回顾前面的问题，receiver在收到SETUP后，请求Conductor创建publication image，但是并没有向sender发送STATUS，进而触发sender开始向receiver发送DATA。 那么，receiver什么时候发送STATUS呢？

回答这个问题，需要同时考察receiver和conductor。

首先receiver中STATUS的发出点（仅此一点）:

	aeron_driver_receiver_do_work => aeron_publication_image_send_pending_status_message


	int aeron_driver_receiver_do_work(void *clientd)
	{
	   ............
	   for (size_t i = 0, length = receiver->images.length; i < length; i++)
	   {
	       aeron_publication_image_t *image = receiver->images.array[i].image;
	
	       int send_sm_result = aeron_publication_image_send_pending_status_message(image);
	       if (send_sm_result < 0)
	       {
	           AERON_DRIVER_RECEIVER_ERROR(receiver, "receiver send SM: %s", aeron_errmsg());
	       }
	        ............
	   }
	    ............
	}
	
	int aeron_publication_image_send_pending_status_message(aeron_publication_image_t *image)
	{
	   int work_count = 0;
	
	   if (NULL != image->endpoint && AERON_PUBLICATION_IMAGE_STATUS_ACTIVE == image->conductor_fields.status)
	   {
	       int64_t change_number;
	       AERON_GET_VOLATILE(change_number, image->end_sm_change);
	
	       if (change_number != image->last_sm_change_number)      // 即 end_sm_change != last_sm_change_number
	       {
	           const int64_t sm_position = image->next_sm_position;
	           const int32_t receiver_window_length = image->next_sm_receiver_window_length;
	
	           aeron_acquire(); /* loadFence */
	
	           if (change_number == image->begin_sm_change) // 即 end_sm_change == image->begin_sm_change
	           {
	               const int32_t term_id =
	                   aeron_logbuffer_compute_term_id_from_position(
	                       sm_position, image->position_bits_to_shift, image->initial_term_id);
	               const int32_t term_offset = (int32_t)(sm_position & image->term_length_mask);
	
	               int send_sm_result = aeron_receive_channel_endpoint_send_sm( // 向sender发送STATUS，注意标志清零，即触发sender开始发送DATA
	                   image->endpoint,
	                   &image->control_address,
	                   image->stream_id,
	                   image->session_id,
	                   term_id,
	                   term_offset,
	                   receiver_window_length,
	                   0); // 标志清零
	
	               aeron_counter_ordered_increment(image->status_messages_sent_counter, 1);
	
	              image->last_sm_change_number = change_number; // 更新 last_sm_change_number = end_sm_change
	               work_count = send_sm_result < 0 ? send_sm_result : 1;
	           }
	       }
	   }
	
	   return work_count;
	}

receiver在每次循环（loop）中，遍历image表，检查是否需要发送STATUS（从aeron的角度来看，即是否有pending的STATUS）。

检查的标准就是image中关于SM的三个计数器：begin\_sm\_change, end\_sm\_change,  last_sm_change_number

如果 begin\_sm\_change和end\_sm\_change相等，且与last\_sm\_change\_number不等，则发送STATUS，并将last\_sm\_change\_number更新至与end\_sm\_change同样值；否则，不必发送。
看起来，这像是个追击问题。

再看这三个计数器在image创建时的初始化：

	int aeron_publication_image_create(...)
	{
	........
	    _image->last_sm_change_number = -1;
	   _image->begin_sm_change = -1;
	........
	   _image->end_sm_change = -1;
	........
	}

比较简单直白，全部初始化为-1。也就是说，image创建之初，这三个计数器值相同，receiver即使获得image，也不会触发STATUS的发送。

再看conductor侧是如何维护这三个计数器的：

	aeron_driver_conductor_do_work => aeron_publication_image_track_rebuild => aeron_publication_image_schedule_status_message


	int aeron_driver_conductor_do_work(void *clientd)
	{
	   aeron_driver_conductor_t *conductor = (aeron_driver_conductor_t *)clientd;
	
	    ................................
	
	   for (size_t i = 0, length = conductor->publication_images.length; i < length; i++)
	   {
	       aeron_publication_image_track_rebuild(
	           conductor->publication_images.array[i].image, now_ns, conductor->context->status_message_timeout_ns);
	   }
	
	   return work_count;
	}
	
	void aeron_publication_image_track_rebuild(
	   aeron_publication_image_t *image, int64_t now_ns, int64_t status_message_timeout)
	{
	  ................................
	
	   bool should_force_send_sm = false;
	   const int32_t window_length = image->congestion_control->on_track_rebuild(
	       image->congestion_control->state,
	       &should_force_send_sm,
	       now_ns,
	       min_sub_pos,
	       image->next_sm_position,
	       hwm_position,
	       rebuild_position,
	       new_rebuild_position,
	       loss_found);
	
	   const int32_t threshold = window_length / 4;
	
	   if (should_force_send_sm ||
	       (now_ns > (image->last_status_mesage_timestamp + status_message_timeout)) ||
	       (min_sub_pos > (image->next_sm_position + threshold)))
	   {
	       aeron_publication_image_schedule_status_message(image, now_ns, min_sub_pos, window_length);
	       aeron_publication_image_clean_buffer_to(image, min_sub_pos - (image->term_length_mask + 1));
	   }
	}

conductor在每次循环（loop）中，遍历publication image表，重建（rebuild）image，即看看是否需要调整一下发送数据的起始位置、发送窗口大小等信息。

关键的拥塞控制（congestion control）算法，输出两个信息：是否需要强制发送STATUS（should\_force\_send\_sm），窗口大小（window\_length）。

然后再结合传进来的SM超时参数（status\_message\_timeout）来判定是否需要发送STATUS：

（a） 如果拥塞控制要求强制发送STATUS（should\_force\_send\_sm），则发送。

（b） 如果距离上次发送STATUS已经过了一个时间周期（status\_message\_timeout），则发送。

（c） 如果最小发送位置（订阅位置的最小值）已经越过image所记录的下次发送起始位置，并且超过1/4窗口大小，则发送。

注意，conductor和receiver在两个不同的线程中，操作同一个image。因此，在conductor的视角，发送STATUS是指对receiver的一种“期待”，期望receiver发出STATUS，所以称之为schedule status message。

来看conductor是如何表达这种愿望的：


	inline void aeron_publication_image_schedule_status_message(
	   aeron_publication_image_t *image, int64_t now_ns, int64_t sm_position, int32_t window_length)
	{
	   const int64_t change_number = image->begin_sm_change + 1;  // 取当前begin_sm_change计数器的下一跳（下一个值）
	
	   AERON_PUT_ORDERED(image->begin_sm_change, change_number);   // begin_sm_change更新到下一跳
	
	   image->next_sm_position = sm_position;
	   image->next_sm_receiver_window_length = window_length;
	   image->last_status_mesage_timestamp = now_ns;
	
	   AERON_PUT_ORDERED(image->end_sm_change, change_number); // end_sm_change更新到下一跳
	}

考虑到多线程的影响，这儿动用内存屏障，确保数据从内存取，而非从CPU Cache中取（确保数据单一来源）。
可以看到，仅当aeron\_publication\_image\_schedule\_status\_message执行完毕，begin\_sm\_change, end\_sm\_change才会更新到同一个下一跳，而此时，last\_sm\_change\_number保持不动。

再回头看receiver的aeron\_publication\_image\_send\_pending\_status\_message，三个计数器满足条件，触发STATUS的发送。

总结一下大致流程：

1， receiver收到SETUP，要求conductor创建image。

2， conductor创建image，将image挂入自身的publication image表中，同时将image告之receiver，receiver将image挂入image表中。

3， conductor根据拥塞控制算法、STATUS超时设置、订阅最小位置，schedule一次SM发送。

4， receiver检测到有pending SM（conductor schedule的结果）时，向sender发送STATUS（标志清零）。

5， sender收到STATUS后，开始向receiver发送数据。

顺便看看image的拥塞控制算法的设定。image的拥塞控制算法初始化发生在image被创建时（aeron\_driver\_conductor\_on\_create\_publication\_image）。

	void aeron_driver_conductor_on_create_publication_image(void *clientd, void *item)
	{
	   aeron_driver_conductor_t *conductor = (aeron_driver_conductor_t *)clientd;
	   aeron_command_create_publication_image_t *command = (aeron_command_create_publication_image_t *)item;
	   aeron_receive_channel_endpoint_t *endpoint = command->endpoint;
	
	................................
	
	   aeron_congestion_control_strategy_t *congestion_control = NULL;
	   if (conductor->context->congestion_control_supplier_func(  // 从congestion control supplier产生一个congestion control
	       &congestion_control,
	       channel_str,
	       command->stream_id,
	       command->session_id,
	       registration_id,
	       command->term_length,
	       command->mtu_length,
	       conductor->context,
	       &conductor->counters_manager) < 0)
	   {
	       return;
	   }
	
	................................
	   aeron_publication_image_t *image = NULL;
	   if (aeron_publication_image_create(
	       &image,
	       endpoint,
	       conductor->context,
	       registration_id,
	       command->session_id,
	       command->stream_id,
	       command->initial_term_id,
	       command->active_term_id,
	       command->term_offset,
	       &rcv_hwm_position,
	       &rcv_pos_position,
	       congestion_control,  // 创建image时，挂到image中作为拥塞控制算法
	       &command->control_address,
	       &command->src_address,
	       command->term_length,
	       command->mtu_length,
	       &conductor->loss_reporter,
	       is_reliable,
	       &conductor->system_counters) < 0)
	   {
	       return;
	   }
	................................
	}
	
可见，image的拥塞控制算法，是从一个congestion control supplier中产生的，而后者来自于context，context来自于配置参数，缺省地，来自于aeron\_static\_window\_congestion\_control\_strategy\_supplier


	int aeron_driver_context_init(aeron_driver_context_t **context)
	{
	   aeron_driver_context_t *_context = NULL;
	
	................................
	   if ((_context->congestion_control_supplier_func =
	       aeron_congestion_control_strategy_supplier_load("aeron_static_window_congestion_control_strategy_supplier")) == NULL)
	   {
	       return -1;
	   }
	................................
	}
	
	int aeron_static_window_congestion_control_strategy_supplier(
	   aeron_congestion_control_strategy_t **strategy,
	   const char *channel,
	   int32_t stream_id,
	   int32_t session_id,
	   int64_t registration_id,
	   int32_t term_length,
	   int32_t sender_mtu_length,
	   aeron_driver_context_t *context,
	   aeron_counters_manager_t *counters_manager)
	{
	   aeron_congestion_control_strategy_t *_strategy;
	
	   if (aeron_alloc((void **)&_strategy, sizeof(aeron_congestion_control_strategy_t)) < 0 ||
	       aeron_alloc((void **)&_strategy->state, sizeof(aeron_static_window_congestion_control_strategy_state_t)) < 0)
	   {
	       return -1;
	   }
	
	   _strategy->should_measure_rtt = aeron_static_window_congestion_control_strategy_should_measure_rtt;
	   _strategy->on_rttm = aeron_static_window_congestion_control_strategy_on_rttm;
	   _strategy->on_track_rebuild = aeron_static_window_congestion_control_strategy_on_track_rebuild; // 这就是conductor在rebuild image时用到的拥塞控制算法
	   _strategy->initial_window_length = aeron_static_window_congestion_control_strategy_initial_window_length;
	   _strategy->fini = aeron_static_window_congestion_control_strategy_fini;
	
	   aeron_static_window_congestion_control_strategy_state_t *state = _strategy->state;
	   const int32_t initial_window_length = (int32_t)context->initial_window_length;
	   const int32_t max_window_for_term = term_length / 2;
	
	   state->window_length = max_window_for_term < initial_window_length ? max_window_for_term : initial_window_length;
	
	   *strategy = _strategy;
	   return 0;
	}
	
	int32_t aeron_static_window_congestion_control_strategy_on_track_rebuild(
	   void *state,
	   bool *should_force_sm,
	   int64_t now_ns,
	   int64_t new_consumption_position,
	   int64_t last_sm_position,
	   int64_t hwm_position,
	   int64_t starting_rebuild_position,
	   int64_t ending_rebuild_position,
	   bool loss_occurred)
	{
	   *should_force_sm = false;
	   return ((aeron_static_window_congestion_control_strategy_state_t *)state)->window_length;
	}
	
缺省的拥塞控制算法很简陋，简言之就是：不强制发送STATUS、窗口大小使用配置值。



Publication Image Log Buffer的结构：

![图 4](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/publication%20log%20buffer%E7%BB%93%E6%9E%84.png)


Log Buffer主要由三部分组成：3个partition，1个Meta Data、1个Default Frame Header。

Log Buffer Partition存的是数据部，由若干定长的Term组成。

Log Buffer Meta Data用来指示每个Partition存的是哪个term的哪部分数据，关键点是其中的背压状态（is_connected）和数组（term_tail_counters）。

Log Buffer Meta Data中的is_connected用来表示subscriber是否连接，在client端称之为背压backpressure  status标志，这个标志来自于
aeron\_network\_publication\_on\_time\_event：


	void aeron_network_publication_on_time_event(
	    aeron_driver_conductor_t *conductor, aeron_network_publication_t *publication, int64_t now_ns, int64_t now_ms)
	{
	    bool has_receivers;
	    AERON_GET_VOLATILE(has_receivers, publication->has_receivers);
	
	    const bool current_connected_status =
	        has_receivers || (publication->spies_simulate_connection && publication->conductor_fields.subscribable.length > 0); // 是否有接受者或者subscriber
	
	    bool is_connected;
	    AERON_GET_VOLATILE(is_connected, publication->is_connected);
	
	    if (current_connected_status != is_connected) // 背压状态发生变化时更新
	    {
	        AERON_PUT_ORDERED(publication->log_meta_data->is_connected, current_connected_status); // 注意：Log Buffer Meta Data中的背压状态标志存在于共享内存，可被外界（进程，线程）访问
	        AERON_PUT_ORDERED(publication->is_connected, current_connected_status); // publication中的背压状态标志存在于当前的进程或线程内存中
	    }
	
	................................
	}
	
再看Log Buffer Meta Data中的数组（term_tail_counters）：

	int64_t term_tail_counters[AERON_LOGBUFFER_PARTITION_COUNT]; // 3个partition

对应关系以数组索引为准，即: term\_tail\_counters[0]与partition[0]对应，term\_tail\_counters[1]与partition[1]对应，term\_tail\_counters[2]与partition[2]对应。

term_tail_counters所存的信息为： （term id, term offset）， 其中，term id在高32位， term offset在低32位。
为方便计，用“|”来表示位的连接，左边为高位部，右边为低位部。于是，

	raw tail = term_id | term_offset

进一步，把Log Buffer看做一个Round Robbin的池，决定一个term插入到哪个partition，可以通过计算该term id与initial term id的距离与3（即AERON\_LOGBUFFER\_PARTITION\_COUNT）取模得到，即

	partition index = （active term id - initial term id） % 3

注意，一对（stream id， session id）对应一个image，当image被创建时， initial term id也被固定下来不再变更，同时initial term id被传递给Log Buffer（被记录下来）不再变更。所以上面计算与initial term id的距离有效。

再来看在一个流中，包的位置（packet position）（也是数据的位置）与Log Buffer中记录的信息（initial term id, term id, term offset）的换算关系。
aeron中约定，term为固定长度（term buffer length）， 于是存在以下关系：

	packet position = （term id - initial term id） * （term buffer length） + （term offset）

进一步，为了减少不必要的乘法运算、提高计算效率，aeron又约定，term的固定长度必须为2的N次方，即

	term buffer length =  1 << （position bits to shift）

于是，

	packet position = （term id - initial term id） | （term offset） = (（term id - initial term id） <<（position bits to shift） ) + （term offset） = （term id - initial term id） |  （term offset）

	注：低位部占（position bits to shift）个比特

反过来，由packet position推导出term id和term offset，按以下方式：

	term offset = （packet position）的低位部（position bits to shift）个比特 =  （packet position） & mask， 这里 mask = （1 << （position bits to shift）） - 1
	term id       =  （packet position）的高位部 + （initial term id） = （（packet position） >> （position bits to shift）） + （initial term id）

根据前述Log Buffer的partition index计算办法， packet position 的高位部模3，即可得到。

	partition index = （（packet position） >> （position bits to shift）） % 3

由此可见， 一个packet唯一地存储在Log Buffer中的一个partition， 一个partition唯一地对应stream中的一个packet。

注意：当一个image被创建时，需要传入term的固定长度（term buffer length），后经计算得到（position bits to shift）并存于image中。


来看看数据入image的过程：receiver收到DATA帧后，插入image。

	aeron_receive_channel_endpoint_dispatch => aeron_receive_channel_endpoint_on_data => aeron_data_packet_dispatcher_on_data => aeron_publication_image_insert_packet


	int aeron_publication_image_insert_packet(
	   aeron_publication_image_t *image, int32_t term_id, int32_t term_offset, const uint8_t *buffer, size_t length)
	{
	   const bool is_heartbeat = aeron_publication_image_is_heartbeat(buffer, length);
	   const int64_t packet_position =
	       aeron_logbuffer_compute_position(term_id, term_offset, image->position_bits_to_shift, image->initial_term_id);
	   const int64_t proposed_position = is_heartbeat ? packet_position : packet_position + (int64_t)length;
	   const int64_t window_position = image->next_sm_position;
	
	   if (!aeron_publication_image_is_flow_control_under_run(image, window_position, packet_position) && // 是否在流控之下
	       !aeron_publication_image_is_flow_control_over_run(image, window_position, proposed_position))    // 是否在流控之上
	   {
	       if (is_heartbeat)
	       {
	           if (!image->is_end_of_stream && aeron_publication_image_is_end_of_stream(buffer, length))
	           {
	               AERON_PUT_ORDERED(image->is_end_of_stream, true);
	               AERON_PUT_ORDERED(image->log_meta_data->end_of_stream_position, packet_position);
	           }
	
	           aeron_counter_ordered_increment(image->heartbeats_received_counter, 1);
	       }
	       else
	       {
	           const size_t index = aeron_logbuffer_index_by_position(packet_position, image->position_bits_to_shift); // 获取partition index
	           uint8_t *term_buffer = image->mapped_raw_log.term_buffers[index].addr; // 获取partition地址
	
	           aeron_term_rebuilder_insert(term_buffer + term_offset, buffer, length);      // 将收到的数据拷贝到partition
	       }
	
	       AERON_PUT_ORDERED(image->last_packet_timestamp_ns, image->nano_clock()); // 更新image最近写入时间戳
	       aeron_counter_propose_max_ordered(image->rcv_hwm_position.value_addr, proposed_position); // 如果可能，更新image记录的position最大值
	   }
	
	   return (int)length;
	}
	
	inline void aeron_term_rebuilder_insert(uint8_t *dest, const uint8_t *src, size_t length)
	{
	   aeron_data_header_t *hdr_dest = (aeron_data_header_t *)dest;
	   aeron_data_header_as_longs_t *dest_hdr_as_longs = (aeron_data_header_as_longs_t *)dest;
	   aeron_data_header_as_longs_t *src_hdr_as_longs = (aeron_data_header_as_longs_t *)src;
	
	   if (0 == hdr_dest->frame_header.frame_length) // 如果目标位置的帧为空，则允许拷贝
	   {
	       memcpy(dest + AERON_DATA_HEADER_LENGTH, src + AERON_DATA_HEADER_LENGTH, length - AERON_DATA_HEADER_LENGTH); // 先拷贝数据部
	
	        // 再拷贝头部
	       dest_hdr_as_longs->hdr[3] = src_hdr_as_longs->hdr[3];
	       dest_hdr_as_longs->hdr[2] = src_hdr_as_longs->hdr[2];
	       dest_hdr_as_longs->hdr[1] = src_hdr_as_longs->hdr[1];
	
	       AERON_PUT_ORDERED(dest_hdr_as_longs->hdr[0], src_hdr_as_longs->hdr[0]);
	   }
	}

数据入image的过程并不复杂，首先根据流控判定是否允许插入image，如果允许，定位到image中的Log Buffer中的partition，然后将收到的数据（整个DATA Frame，包含头部和数据部）插入到partition相应的偏移位置（term offset），
最后，更新image中记录的最近写入时间戳和最大position值。

我们先看插入partition的具体做法，再看流控。

数据插入接口（aeron\_term\_rebuilder\_insert）首先检查目标位置的帧是否为空。如果不为空，则拒绝插入；如果未空，则开始拷贝插入。

插入时，先拷贝DATA Frame的数据部，然后按照64比特整数形式，从后向前拷贝4个整数， 共32个字节，正是DATA Frame Header的大小。并且在拷贝第0个字节时，动用了内存屏障。

> Q： 源和目标数据都是连续内存存放，为什么不一次性内存拷贝，而是对数据部和头部分别拷贝？
>
> A： 推测有两个原因：一是要动用内存屏障；二是考虑到未来字节序的调整。根据 [FAQ 4](https://github.com/real-logic/aeron/wiki)

> Q: The protocol specification says that Little Endian is used for byte order of fields. Network order is always Big Endian. Why doesn't Aeron use Big Endian?
A: The primary latency sensitive systems in use today are Little Endian. Adding unnecessary byte reversal would be wasteful. Adding conditional checks for byte ordering to frames would be wasteful. If the need arises in the future, highly doubtful, to be conditionally Big Endian, then the Version field would be used to hold a Big Endian field bit flag.

aeron是按照小端来收发数据的，而通常网络字节序是大端，也就是为了节省字节序调整带来的时间开销，aeron使用了小端的网络字节序。如果目标机是大端的CPU，那么aeron就需要在收到Frame后，调整Frame Header中各field的字节序。此处是一个待修改点。

流控的两个接口如下：

	inline bool aeron_publication_image_is_flow_control_under_run(
	   aeron_publication_image_t *image, int64_t window_position, int64_t packet_position)
	{
	   const bool is_flow_control_under_run = packet_position < window_position;
	
	   if (is_flow_control_under_run)
	   {
	       aeron_counter_ordered_increment(image->flow_control_under_runs_counter, 1);
	   }
	
	   return is_flow_control_under_run;
	}
	
	inline bool aeron_publication_image_is_flow_control_over_run(
	   aeron_publication_image_t *image, int64_t window_position, int64_t proposed_position)
	{
	   const bool is_flow_control_over_run = proposed_position > (window_position + image->next_sm_receiver_window_length);
	
	   if (is_flow_control_over_run)
	   {
	       aeron_counter_ordered_increment(image->flow_control_over_runs_counter, 1);
	   }
	
	   return is_flow_control_over_run;
	}
	
实现比较简单，就是检查packet position是否落在区间[ window position, window position + window length]。如果落在左边，则判定为流控之下；如果落在右边，则判定为流控之上；如果落在区间内，则流控通过。

注意，这里window position就是由conductor调整的sm position，参见接口aeron\_publication\_image\_schedule\_status\_message

留意到，数据插入到image，事实上是插入到image所管理的Log Buffer中，

	aeron_publication_image_insert_packet
	 uint8_t *term_buffer = image->mapped_raw_log.term_buffers[index].addr; // 获取partition地址

这里，mapped_raw_log就是Log Buffer，  term_buffers是其管理的3个partition。来看image创建时是如何创建term_buffers的。

	aeron_publication_image_create
	    if (context->map_raw_log_func( // 建立 Log Buffer
	        &_image->mapped_raw_log,
	        path,
	        context->term_buffer_sparse_file,
	        (uint64_t)term_buffer_length,
	        context->file_page_size) < 0)
	    {
	        aeron_free(_image->log_file_name);
	        aeron_free(_image);
	        aeron_set_err(aeron_errcode(), "error mapping network raw log %s: %s", path, aeron_errmsg());
	        return -1;
	    }
	    _image->map_raw_log_close_func = context->map_raw_log_close_func;
	
	    strncpy(_image->log_file_name, path, path_length);
	    _image->log_file_name[path_length] = '\0';
	    _image->log_file_name_length = (size_t)path_length;
	    _image->log_meta_data = (aeron_logbuffer_metadata_t *)(_image->mapped_raw_log.log_meta_data.addr);   // 建立到log meta data的短路（shortcut）
	
	    _image->log_meta_data->initial_term_id = initial_term_id;
	    _image->log_meta_data->mtu_length = sender_mtu_length;
	    _image->log_meta_data->term_length = term_buffer_length;
	    _image->log_meta_data->page_size = (int32_t)context->file_page_size;
	    _image->log_meta_data->correlation_id = correlation_id;
	    _image->log_meta_data->is_connected = 0;
	    _image->log_meta_data->end_of_stream_position = INT64_MAX;
	    aeron_logbuffer_fill_default_header( // 初始化缺省的data frame header
	        _image->mapped_raw_log.log_meta_data.addr, session_id, stream_id, initial_term_id);

map\_raw\_log\_func和map\_raw\_log\_close\_func是建立和关闭Log Buffer的接口，来自于context，初始化为：

    _context->map_raw_log_func = aeron_map_raw_log;
    _context->map_raw_log_close_func = aeron_map_raw_log_close;

其本质就是通过mmap建立磁盘文件到内存的映射，一个文件存的就是一个Log Buffer，其内存结构请查阅前面的图解。

该文件的长度为 term_length * 3 + 508字节，其中term_length来自SETUP：

Setup Frame


	0 1 2 3
	0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
	+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
	|R| Frame Length (= header length) |
	+---------------+---------------+-------------------------------+
	| Version | Flags | Type (=0x05) |
	+---------------+---------------+-------------------------------+
	|R| Term Offset |
	+---------------------------------------------------------------+
	| Session ID |
	+---------------------------------------------------------------+
	| Stream ID |
	+---------------------------------------------------------------+
	| Initial Term ID |
	+---------------------------------------------------------------+
	| Active Term ID |
	+---------------------------------------------------------------+
	| Term Length |
	+---------------------------------------------------------------+
	| MTU |
	+---------------------------------------------------------------+
	| TTL |
	+---------------------------------------------------------------+

sender侧，发送的SETUP携带的Term Length可以追踪确认该值在aeron\_driver\_context\_init中初始化，缺省为16MB，可以通过配置环境变量AERON\_TERM\_BUFFER\_LENGTH改变初值。


	_pub->term_length_mask = (int32_t)term_buffer_length - 1;
	
	term_buffer_length = conductor->context->term_buffer_length
	
	_context->term_buffer_length = 16 * 1024 * 1024;
	
	_context->term_buffer_length =
	        aeron_config_parse_uint64(
	            getenv(AERON_TERM_BUFFER_LENGTH_ENV_VAR),
	            _context->term_buffer_length,
	            1024,
	            INT32_MAX);

数据流图

通过Log Buffer交换数据（data），通常在/dev/shm目录下的共享内存：publisher将数据写入sender端的Log Buffer，subscriber从receiver端的Log Buffer读出数据。

指令流图：

client和driver之间通过cnc共享内存发送指令（command）和广播反馈建立双向通信。

	Aeron clients communicate with media driver via the command and control (C'n'C) file which is memory mapped. The media driver can run in or out of process as required. An out of process driver is isolated and can be shared by many clients. Each media driver can have many clients which can concurrently interact. Clients send commands to the driver and the driver will broadcast back events. A bi-directional heart-beating protocol is used between the driver and clients to track each others presence.

ref: [Client-Concurrency-Model](https://github.com/real-logic/Aeron/wiki/Client-Concurrency-Model)


cnc文件的创建：

	int aeron_driver_create_cnc_file(aeron_driver_t *driver)
	{
	    char buffer[AERON_MAX_PATH];
	    size_t cnc_file_length = aeron_cnc_length(driver->context);
	
	    driver->context->cnc_map.addr = NULL;
	    driver->context->cnc_map.length = cnc_file_length;
	
	    snprintf(buffer, sizeof(buffer) - 1, "%s/%s", driver->context->aeron_dir, AERON_CNC_FILE); // 比如：/dev/shm/aeron-root/cnc.dat
	
	    if (aeron_map_new_file(&driver->context->cnc_map, buffer, true) < 0)
	    {
	        aeron_set_err(aeron_errcode(), "could not map CnC file: %s", aeron_errmsg());
	        return -1;
	    }
	
	    aeron_driver_fill_cnc_metadata(driver->context);
	    return 0;
	}
	
aeron\_map\_new\_file通过mmap将共享内存文件cnc，比如/dev/shm/aeron\-root/cnc.dat，映射到内存，内存地址和内存大小记录在cnc\_map中。

指令通道和反馈通道的内存建立：

	void aeron_driver_fill_cnc_metadata(aeron_driver_context_t *context)
	{
	    aeron_cnc_metadata_t *metadata = (aeron_cnc_metadata_t *)context->cnc_map.addr;
	    metadata->to_driver_buffer_length = (int32_t)context->to_driver_buffer_length;
	    metadata->to_clients_buffer_length = (int32_t)context->to_clients_buffer_length;
	    metadata->counter_metadata_buffer_length = (int32_t)context->counters_metadata_buffer_length;
	    metadata->counter_values_buffer_length = (int32_t)context->counters_values_buffer_length;
	    metadata->error_log_buffer_length = (int32_t)context->error_buffer_length;
	    metadata->client_liveness_timeout = (int64_t)context->client_liveness_timeout_ns;
	    metadata->start_timestamp = context->epoch_clock();
	    metadata->pid = getpid();
	
	    context->to_driver_buffer = aeron_cnc_to_driver_buffer(metadata); // client向driver发送指令的通道 （mpsc）
	    context->to_clients_buffer = aeron_cnc_to_clients_buffer(metadata); // driver向client发送反馈的通道（broadcast）
	    context->counters_values_buffer = aeron_cnc_counters_values_buffer(metadata);
	    context->counters_metadata_buffer = aeron_cnc_counters_metadata_buffer(metadata);
	    context->error_buffer = aeron_cnc_error_log_buffer(metadata);
	}
	
	inline uint8_t *aeron_cnc_to_driver_buffer(aeron_cnc_metadata_t *metadata)
	{
	    return (uint8_t *)metadata + AERON_CNC_VERSION_AND_META_DATA_LENGTH;
	}
	
	inline uint8_t *aeron_cnc_to_clients_buffer(aeron_cnc_metadata_t *metadata)
	{
	    return (uint8_t *)metadata + AERON_CNC_VERSION_AND_META_DATA_LENGTH +
	        metadata->to_driver_buffer_length;
	}
	
	#define AERON_CNC_VERSION_AND_META_DATA_LENGTH (AERON_ALIGN(sizeof(aeron_cnc_metadata_t), AERON_CACHE_LINE_LENGTH * 2))
	
	#define AERON_CACHE_LINE_LENGTH (64)
	
	_context->to_driver_buffer_length = 1024 * 1024 + AERON_RB_TRAILER_LENGTH;
	_context->to_clients_buffer_length = 1024 * 1024 + AERON_BROADCAST_BUFFER_TRAILER_LENGTH;
	
	#define AERON_RB_TRAILER_LENGTH (sizeof(aeron_rb_descriptor_t))
	
	#define AERON_BROADCAST_BUFFER_TRAILER_LENGTH (sizeof(aeron_broadcast_descriptor_t))
	
即cnc\_map起始地址起128字节偏移量的地方，是client到driver的指令通道地址，长度为to\_driver\_buffer\_length，这段内存是一个mpsc（多生产者单消费者）的ring buffer （rb），内存缺省大小为1MB；

紧随其后的，是driver到client的反馈通道地址，长度为to\_clients\_buffer\_length，这段内存是一个spmc（单生产者多消费者）（或称broadcast，广播）的ring buffer（rb），内存缺省大小为1MB。


	int aeron_driver_conductor_init(aeron_driver_conductor_t *conductor, aeron_driver_context_t *context)
	{
	    if (aeron_mpsc_rb_init(
	        &conductor->to_driver_commands, context->to_driver_buffer, context->to_driver_buffer_length) < 0)
	    {
	        return -1;
	    }
	
	    if (aeron_broadcast_transmitter_init(
	        &conductor->to_clients, context->to_clients_buffer, context->to_clients_buffer_length) < 0)
	    {
	        return -1;
	    }
	................................
	}
	
	int aeron_mpsc_rb_init(volatile aeron_mpsc_rb_t *ring_buffer, void *buffer, size_t length)
	{
	    const size_t capacity = length - AERON_RB_TRAILER_LENGTH;
	    int result = -1;
	
	    if (AERON_RB_IS_CAPACITY_VALID(capacity))
	    {
	        ring_buffer->buffer = buffer;
	        ring_buffer->capacity = capacity;
	        ring_buffer->descriptor = (aeron_rb_descriptor_t *) (ring_buffer->buffer + ring_buffer->capacity);
	        ring_buffer->max_message_length = AERON_RB_MAX_MESSAGE_LENGTH(ring_buffer->capacity);  
	        result = 0;
	    }
	    else
	    {
	        aeron_set_err(EINVAL, "%s:%d: %s", __FILE__, __LINE__, strerror(EINVAL));
	    }
	
	    return result;
	}
	
	int aeron_broadcast_transmitter_init(volatile aeron_broadcast_transmitter_t *transmitter, void *buffer, size_t length)
	{
	    const size_t capacity = length - AERON_BROADCAST_BUFFER_TRAILER_LENGTH;
	    int result = -1;
	
	    if (AERON_BROADCAST_IS_CAPACITY_VALID(capacity))
	    {
	        transmitter->buffer = buffer;
	        transmitter->capacity = capacity;
	        transmitter->descriptor = (aeron_broadcast_descriptor_t *) (transmitter->buffer + transmitter->capacity);
	        transmitter->max_message_length = AERON_BROADCAST_MAX_MESSAGE_LENGTH(transmitter->capacity);
	        result = 0;
	    }
	    else
	    {
	        aeron_set_err(EINVAL, "%s:%d: %s", __FILE__, __LINE__, strerror(EINVAL));
	    }
	
	    return result;
	}
	
cnc metadata | log buffer to driver (1M) | rb tailer (rb descriptor) | log buffer to clients (1M) | broadcast tailer（broadcast descriptor）

rb log buffer: rb record | rb record  |  ... 

rb record: rb record header | rb record msg

rb record header: msg type id | msg length

broadcast log buffer: broadcast record | broadcast record  |  ... 

broadcast record: broadcast record header | broadcast record msg

broadcast record header: msg length | msg type id

![图 6 CNC文件图解](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/publication%20log%20buffer%E7%BB%93%E6%9E%842.png)


（现在我们知道，与aeron driver交互就是基于driver提供的指令通道和反馈通道）

create publication => create image => create log buffers


subscription流程：

	1， subscriber （client）向areon media driver发送AERON_COMMAND_ADD_SUBSCRIPTION指令（事实上是发送给areon conductor）
	
	2， conductor处理该指令，首先创建endpoint（ 携带subscriber endpoint的信息，特别是fd）。
	       对于单播（unicast），该endpoint是一个udp server，需要绑定到单播端口监听；
	       对于组播（multicast），该endpoint需要绑定到组播端口监听
	
	3，conductor向所有client广播AERON_RESPONSE_ON_SUBSCRIPTION_READY
	
	4，然后，conductor找到image，建立endpoint和image的联系
	
	5，conductor向所有client广播AERON_RESPONSE_ON_AVAILABLE_IMAGE

	aeron_driver_conductor_on_command => 	aeron_driver_conductor_on_add_network_subscription => 	aeron_driver_conductor_get_or_add_receive_channel_endpoint =>  	aeron_receive_channel_endpoint_create


	int aeron_driver_conductor_on_add_network_subscription(
	    aeron_driver_conductor_t *conductor,
	    aeron_subscription_command_t *command)
	{
	    aeron_client_t *client = NULL;
	    aeron_udp_channel_t *udp_channel = NULL;
	    aeron_receive_channel_endpoint_t *endpoint = NULL;
	    const char *uri = (const char *)command + sizeof(aeron_subscription_command_t);
	    int ensure_capacity_result = 0;
	
	    if (aeron_udp_channel_parse(uri, (size_t)command->channel_length, &udp_channel) < 0)
	    {
	        return -1;
	    }
	
	    if ((client = aeron_driver_conductor_get_or_add_client(conductor, command->correlated.client_id)) == NULL)
	    {
	        return -1;
	    }
	
	    if ((endpoint = aeron_driver_conductor_get_or_add_receive_channel_endpoint(conductor, udp_channel)) == NULL)  // 获取已有的或创建新的endpoint
	    {
	        return -1;
	    }
	
	    if (AERON_RECEIVE_CHANNEL_ENDPOINT_STATUS_CLOSING == endpoint->conductor_fields.status)
	    {
	        aeron_set_err(EINVAL, "%s", "receive_channel_endpoint found in CLOSING state");
	        return -1;
	    }
	
	    if (aeron_receive_channel_endpoint_incref_to_stream(endpoint, command->stream_id) < 0)
	    {
	        return -1;
	    }
	
	    AERON_ARRAY_ENSURE_CAPACITY(ensure_capacity_result, conductor->network_subscriptions, aeron_subscription_link_t);
	    if (ensure_capacity_result >= 0)
	    {
	        aeron_subscription_link_t *link = &conductor->network_subscriptions.array[conductor->network_subscriptions.length++];
	
	        link->endpoint = endpoint;
	        link->spy_channel = NULL;
	        link->stream_id = command->stream_id;
	        link->client_id = command->correlated.client_id;
	        link->registration_id = command->correlated.correlation_id;
	        link->subscribable_list.length = 0;
	        link->subscribable_list.capacity = 0;
	        link->subscribable_list.array = NULL;
	
	        aeron_driver_conductor_on_subscription_ready( // conductor向所有client广播AERON_RESPONSE_ON_SUBSCRIPTION_READY
	            conductor, command->correlated.correlation_id, (int32_t)endpoint->channel_status.counter_id);
	
	        for (size_t i = 0, length = conductor->publication_images.length; i < length; i++)
	        {
	            aeron_publication_image_t *image = conductor->publication_images.array[i].image;
	
	            if (endpoint == image->endpoint && command->stream_id == image->stream_id &&
	                aeron_publication_image_is_accepting_subscriptions(image)) // 根据（endpoint, stream_id）找到匹配的image。注意，在没有publisher时，不会创建image，从而把不会广播下面的反馈
	            {
	                char source_identity[AERON_MAX_PATH];
	                aeron_format_source_identity(source_identity, sizeof(source_identity), &image->source_address);
	
	                if (aeron_driver_conductor_link_subscribable( // conductor向所有client广播AERON_RESPONSE_ON_AVAILABLE_IMAGE
	                    conductor,
	                    link,
	                    &image->conductor_fields.subscribable,
	                    image->conductor_fields.managed_resource.registration_id,
	                    image->session_id,
	                    image->stream_id,
	                    aeron_counter_get(image->rcv_pos_position.value_addr),
	                    endpoint->conductor_fields.udp_channel->original_uri,
	                    source_identity,
	                    image->log_file_name,
	                    image->log_file_name_length) < 0)
	                {
	                    return -1;
	                }
	            }
	        }
	
	        if (endpoint->conductor_fields.udp_channel != udp_channel)
	        {
	            aeron_udp_channel_delete(udp_channel);
	        }
	
	        return 0;
	    }
	
	    return -1;
	}
	
	int aeron_receive_channel_endpoint_create(
	    aeron_receive_channel_endpoint_t **endpoint,
	    aeron_udp_channel_t *channel,
	    aeron_counter_t *status_indicator,
	    aeron_system_counters_t *system_counters,
	    aeron_driver_context_t *context)
	{
	    aeron_receive_channel_endpoint_t *_endpoint = NULL;
	
	    if (aeron_alloc((void **)&_endpoint, sizeof(aeron_receive_channel_endpoint_t)) < 0)
	    {
	        int errcode = errno;
	
	        aeron_set_err(errcode, "could not allocate receive_channel_endpoint: %s", strerror(errcode));
	        return -1;
	    }
	
	    if (aeron_data_packet_dispatcher_init(
	        &_endpoint->dispatcher, context->conductor_proxy, context->receiver_proxy->receiver) < 0)
	    {
	        return -1;
	    }
	
	    if (aeron_int64_to_ptr_hash_map_init(
	        &_endpoint->stream_id_to_refcnt_map, 16, AERON_INT64_TO_PTR_HASH_MAP_DEFAULT_LOAD_FACTOR) < 0)
	    {
	        int errcode = errno;
	
	        aeron_set_err(errcode, "could not init stream_id_to_refcnt_map: %s", strerror(errcode));
	        return -1;
	    }
	
	    _endpoint->conductor_fields.udp_channel = channel;
	    _endpoint->conductor_fields.managed_resource.clientd = _endpoint;
	    _endpoint->conductor_fields.managed_resource.registration_id = -1;
	    _endpoint->conductor_fields.status = AERON_RECEIVE_CHANNEL_ENDPOINT_STATUS_ACTIVE;
	    _endpoint->transport.fd = -1;
	    _endpoint->channel_status.counter_id = -1;
	
	    if (aeron_udp_channel_transport_init( // 创建UDP套接字，绑定单播或组播端口
	        &_endpoint->transport, // UDP套接字文件描述符fd存于此
	        &channel->remote_data,
	        &channel->local_data,
	        channel->interface_index,
	        (0 != channel->multicast_ttl) ? channel->multicast_ttl : context->multicast_ttl,
	        context->socket_rcvbuf,
	        context->socket_sndbuf) < 0)
	    {
	        aeron_receive_channel_endpoint_delete(NULL, _endpoint);
	        return -1;
	    }
	
	    if (aeron_udp_channel_transport_get_so_rcvbuf(&_endpoint->transport, &_endpoint->so_rcvbuf) < 0)
	    {
	        aeron_receive_channel_endpoint_delete(NULL, _endpoint);
	        return -1;
	    }
	
	    _endpoint->transport.dispatch_clientd = _endpoint;
	    _endpoint->has_receiver_released = false;
	
	    _endpoint->channel_status.counter_id = status_indicator->counter_id;
	    _endpoint->channel_status.value_addr = status_indicator->value_addr;
	
	    _endpoint->receiver_id = context->receiver_id;
	    _endpoint->receiver_proxy = context->receiver_proxy;
	
	    _endpoint->short_sends_counter = aeron_system_counter_addr(system_counters, AERON_SYSTEM_COUNTER_SHORT_SENDS);
	    _endpoint->possible_ttl_asymmetry_counter =
	        aeron_system_counter_addr(system_counters, AERON_SYSTEM_COUNTER_POSSIBLE_TTL_ASYMMETRY);
	
	    *endpoint = _endpoint;
	    return 0;
	}

注：一个endpoint可以传输多个流（stream），每个流对应一个image。

来看数据流示意图：


单机：

![图 7 单机](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/%E5%8D%95%E6%9C%BA%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A4%BA%E6%84%8F%E5%9B%BE.png)


双机：

![图 8 双机](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/%E5%8F%8C%E6%9C%BA%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A4%BA%E6%84%8F%E5%9B%BE.png)



multicast:

### scenario 1:  同主机上，先启动subscriber，再启动publisher

#### s1. 启动media driver:  export AERON_DIR=/dev/shm/aeron-cyzhou-001 && ./aeronmd

#### s2. 启动subscriber: ./BasicSubscriber -p /dev/shm/aeron-cyzhou-001 -c aeron:udp?endpoint=224.10.9.7:4050

#### s3. 查看内存目录： tree /dev/shm/aeron-cyzhou-001

![图 9](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/scenario%201-3.png)

#### s4. 启动publisher: ./BasicPublisher -p /dev/shm/aeron-cyzhou-001 -c aeron:udp?endpoint=224.10.9.7:4050

#### s5. 查看内存目录：tree /dev/shm/aeron-cyzhou-001

![图 10](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/scenario%201-5.png)


可见，publication和image的log buffer被创建。

#### s6. 终止aeronmd，一段时间后，subscriber和publisher自动终止，报告心跳超时。

### scenario 2: 同主机上，先启动publisher，再启动subscriber

#### s1. 启动media driver:  export AERON_DIR=/dev/shm/aeron-cyzhou-002 && ./aeronmd

#### s2. 启动publisher: ./BasicPublisher -p /dev/shm/aeron-cyzhou-002 -c aeron:udp?endpoint=224.10.9.7:4050

#### s3. 查看内存目录：tree /dev/shm/aeron-cyzhou-002

![图 11](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/scenario%202-3.png)

publication的log buffer被创建。

#### s4. 启动subscriber: ./BasicSubscriber -p /dev/shm/aeron-cyzhou-002 -c aeron:udp?endpoint=224.10.9.7:4050

#### s5. 查看内存目录： tree /dev/shm/aeron-cyzhou-002

![图 12](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/scenario%202-4.png)

image的log buffer被创建。

### scenario 3: 局域网内两台主机上，分别启动publisher和subscriber

#### s1. 主机A上，启动media driver:  export AERON_DIR=/dev/shm/aeron-cyzhou-003 && ./aeronmd

#### s2. 启动publisher: ./BasicPublisher -p /dev/shm/aeron-cyzhou-003 -c aeron:udp?endpoint=224.10.9.7:4050

#### s3. 查看内存目录：tree /dev/shm/aeron-cyzhou-003

![图 13](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/scenario%203-3.png)

#### s4. 主机B上，启动media driver:  export AERON_DIR=/dev/shm/aeron-cyzhou-003 && ./aeronmd

#### s5. 启动subscriber: ./BasicSubscriber -p /dev/shm/aeron-cyzhou-003 -c aeron:udp?endpoint=224.10.9.7:4050

#### s6. 查看内存目录： tree /dev/shm/aeron-cyzhou-003

![图 14](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/scenario%203-6.png)


unicast:
### scenario 4:主机A（192.168.4.61）启动publisher, B（192.168.4.181）启动subscriber

#### s1. 主机A上，启动media driver:  export AERON_DIR=/dev/shm/aeron-cyzhou-005 && ./aeronmd

#### s2. 主机A上，启动publisher: ./BasicPublisher -p /dev/shm/aeron-cyzhou-005 -c aeron:udp?endpoint=192.168.4.181:4050
（注意：这儿的endpoint是subscriber主机地址，即pub的目标机地址）

#### s3. 主机A上，查看内存目录：tree /dev/shm/aeron-cyzhou-005

![图 15](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/scenario%204-3.png)

#### s4. 主机B上，启动media driver:  export AERON_DIR=/dev/shm/aeron-cyzhou-005 && ./aeronmd

#### s5. 主机B上，启动subscriber: ./BasicSubscriber -p /dev/shm/aeron-cyzhou-005 -c aeron:udp?endpoint=192.168.4.181:4050

#### s6. 查看内存目录： tree /dev/shm/aeron-cyzhou-005

![图 16](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/scenario%204-6.png)


总结：

	（1）一对publisher和subscriber，对应一对log buffer，分别在publications目录下和images目录下。
	（2）subscriber的image logbuffer创建，仅当publisher的publication log buffer被创建后。

shell命令集：

组播：

	export AERON_DIR=/dev/shm/aeron-cyzhou-003 && ./aeronmd
	./BasicPublisher -p /dev/shm/aeron-cyzhou-003 -c aeron:udp?endpoint=224.10.9.7:4050
	./BasicSubscriber -p /dev/shm/aeron-cyzhou-003 -c aeron:udp?endpoint=224.10.9.7:4050

单播：

	export AERON_DIR=/dev/shm/aeron-cyzhou-005 && ./aeronmd
	./BasicPublisher -p /dev/shm/aeron-cyzhou-005 -c aeron:udp?endpoint=192.168.4.181:4050
	./BasicSubscriber -p /dev/shm/aeron-cyzhou-005 -c aeron:udp?endpoint=192.168.4.181:4050

注：aeron约定，组播地址必须是奇数，作为data endpoint；紧随其后的组播地址作为control endpoint。
比如，组播地址224.10.9.7用作数据传输通道，默认地，aeron将组播地址224.10.9.8用作控制传输通道。


# Archive原理：

### Scenario 1: Archive Client和Archive Media Driver同机部署

![图 17](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/archive%E5%8E%9F%E7%90%86-%E5%90%8C%E6%9C%BA%E9%83%A8%E7%BD%B2.png)


Archive Media Driver启动时：

	1，  创建CNC
	2，  创建Subscription 8010/10（【注】格式为：channel port/stream id），用来处理来自客户端的指令，如Connect Request，Start Recording，Start Replay等。（注意：在Publication 8010/10创建前，Image 8010/10不会真的创建）
	3，  创建Exclusive Publication 8030/30，用来向8030/30上可能的所有subscriber发送录播进度（Record Progress）。

Archive Client启动时：

	1， 通过CNC，向Archive Media Driver发送指令：
	    1.1，发送ADD SUBSCRIPTION 8020/20，触发Archive Media Driver发起Subscription 8020/20创建流程（注意：在Publication 8020/20创建前，Image 8020/20不会真的创建） ，Archive Client利用该通道接收指令反馈。
	    1.2，发送ADD EXCLUSIVE PUBLICATION 8010/10，触发Archive Media Driver发起Publication 8010/10创建流程，从而创建Exclusive Publication 8010/10和Image 8010/10。
	2，发送Archive指令：
	    2.1，在Publication 8010/10通道上，发送（offer）指令Connect Request。 Archive Media Driver在Subscription 8010/10上收到该指令后，发起Publication 8020/20创建流程，从而创建Exclusive Publication 8020/20和Image 8020/20。Archive Media Driver利用该通道向Archive Client发送指令反馈。
	    2.2，在Publication 8010/10通道上，发送（offer）指令Start Recording。Archive Media Driver在Subscription 8010/10上收到该指令后，发起Record流程。
	    2.3，在Publication 8010/10通道上，发送（offer）指令Start Replay。Archive Media Driver在Subscription 8010/10上收到该指令后，发起Replay流程。

可能的数据传输：

	1， Archive Client通过CNC，向Archive Media Driver发送指令ADD PUBLICATION 40123/10，创建Publication 40123/10。
	2， Archive Client在Publication 40123/10通道上，发送（offer）数据。
	3， Archive Media Driver将Publication 40123/10通道中的数据，发送给所有Subscriber，如果有的话。

注意：Archive Client向Archive Media Driver发送指令或者数据，都是指通过操作共享内存，直接写入；接收指令反馈，则是操作共享内存，直接读出。

### Scenario 2: Archive Client和Archive Media Driver双机部署

![图 18](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/archive%E5%8F%8C%E6%9C%BA%E9%83%A8%E7%BD%B2.png)


Archive Client和Archive Media Driver双机部署时，需要一个Media Driver完成传输中转。如上图所示，Archive Client和Media Driver部署在一台设备上，Archive Media Driver部署在另一台设备上。

Archive Media Driver启动时（注：与同机部署一样）：

	1，  创建CNC
	2，  创建Subscription 8010/10（【注】格式为：channel port/stream id），用来处理来自客户端的指令，如Connect Request，Start Recording，Start Replay等。（注意：在另一台设备上的Publication 8010/10创建前，Image 8010/10不会真的创建）
	3，  创建Exclusive Publication 8030/30，用来向8030/30上可能的所有subscriber发送录播进度（Record Progress）。

Media Driver启动时：
	1， 创建CNC

Archive Client启动时：

	1， 通过CNC，向Media Driver发送指令：
	    1.1，发送ADD SUBSCRIPTION 8020/20，触发Media Driver发起Subscription 8020/20创建流程（注意：在另一台设备上的Publication 8020/20创建前，Image 8020/20不会真的创建） ，Archive Client利用该通道接收指令反馈。
	    1.2，发送ADD EXCLUSIVE PUBLICATION 8010/10，触发Media Driver发起Publication 8010/10创建流程，从而在本设备上创建Exclusive Publication 8010/10和在另一台设备上创建Image 8010/10。
	2，发送Archive指令：
	    2.1，在Publication 8010/10通道上，发送（offer）指令Connect Request。 Archive Media Driver在Subscription 8010/10上收到该指令后，发起Publication 8020/20创建流程，从而在本设备上创建Exclusive Publication 8020/20和在另一台设备上创建Image 8020/20。Archive Media Driver利用该通道向Archive Client发送指令反馈。
	    2.2，在Publication 8010/10通道上，发送（offer）指令Start Recording。Archive Media Driver在Subscription 8010/10上收到该指令后，发起Record流程。
	    2.3，在Publication 8010/10通道上，发送（offer）指令Start Replay。Archive Media Driver在Subscription 8010/10上收到该指令后，发起Replay流程。

注：Archive指令的基本流程为：

    s1. Archive Client通过操作共享内存的方式，将指令写入Media Driver的Exclusive Publication 8010/10；
    s2. Media Driver将指令pub到Archive Media Driver；
    s3. Archive Media Driver内部的8010/10 subscriber处理指令，并将反馈通过8020/20 publish给Media Driver；
    s4. Media Driver收到反馈；
    s5.  Archive Client从Media Driver的image 8020/20中取出反馈。

注：双机部署务必在channel中指明UDP监听者的IP地址。

注：双机部署同样适用于单机部署、不同的aeron目录（即aeron.dir）设置，此时channel中依然可以使用localhost。
