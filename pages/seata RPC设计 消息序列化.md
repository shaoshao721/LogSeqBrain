tags:: seata，rpc，消息序列化

- 载入序列化类
	- ```
	  // 载入序列化器
	  Serializer serializer = SerializerServiceLoader.load(SerializerType.getByCode(rpcMessage.getCodec()));
	  ```
- 默认的序列化类为seataSerializer，实现Serializer接口的序列化
- ```
  public interface Serializer {
      <T> byte[] serialize(T t);
  
      <T> T deserialize(byte[] bytes);
  }
  ```
- SeataSerializer类如何实现序列化和反序列化的
- ```
  public <T> byte[] serialize(T t) {
          if (t == null || !(t instanceof AbstractMessage)) {
              throw new IllegalArgumentException("AbstractMessage isn't available.");
          }
          // 转为抽象消息
          AbstractMessage abstractMessage = (AbstractMessage)t;
          //获得类型码
          short typecode = abstractMessage.getTypeCode();
          //根据类型码能活的到编码器或解码器
          MessageSeataCodec messageCodec = MessageCodecFactory.getMessageCodec(typecode);
          //获得1kb大小的bugger
          ByteBuf out = Unpooled.buffer(1024);
          //将消息编码到buffer里
          messageCodec.encode(t, out);
          byte[] body = new byte[out.readableBytes()];
          // 从buffer中读取字节数组
          out.readBytes(body);
  
          //分配一个字节bugger
          ByteBuffer byteBuffer = ByteBuffer.allocate(2 + body.length);
          // 放入类型码和body
          byteBuffer.putShort(typecode);
          byteBuffer.put(body);
  
          byteBuffer.flip();
          byte[] content = new byte[byteBuffer.limit()];
          // 得到序列化后的字节数组
          byteBuffer.get(content);
          return content;
      }
  ```
- 反序列化
	- ```
	  public <T> T deserialize(byte[] bytes) {
	          if (bytes == null || bytes.length == 0) {
	              throw new IllegalArgumentException("Nothing to decode.");
	          }
	          // 正确的字节长度应该大于2
	          if (bytes.length < 2) {
	              throw new IllegalArgumentException("The byte[] isn't available for decode.");
	          }
	          ByteBuffer byteBuffer = ByteBuffer.wrap(bytes);
	          //获取字节码和body
	          short typecode = byteBuffer.getShort();
	          byte[] body = new byte[byteBuffer.remaining()];
	          byteBuffer.get(body);
	          ByteBuffer in = ByteBuffer.wrap(body);
	          //根据字节码创建消息
	          AbstractMessage abstractMessage = MessageCodecFactory.getMessage(typecode);
	          MessageSeataCodec messageCodec = MessageCodecFactory.getMessageCodec(typecode);
	          //解码为消息对象
	          messageCodec.decode(abstractMessage, in);
	          return (T)abstractMessage;
	      }
	  ```
- 通过类型码可以得到具体的消息类型的编码和解码类，通过这个来进行实际的操作。
- 通过类型码获取具体的消息类型的代码解析
	- ```
	      public static MessageSeataCodec getMessageCodec(short typeCode) {
	          MessageSeataCodec msgCodec = null;
	          switch (typeCode) {
	              // 合并消息
	              case MessageType.TYPE_SEATA_MERGE:
	                  msgCodec = new MergedWarpMessageCodec();
	                  break;
	              // 合并结果消息
	              case MessageType.TYPE_SEATA_MERGE_RESULT:
	                  msgCodec = new MergeResultMessageCodec();
	                  break;
	              // TM注册消息
	              case MessageType.TYPE_REG_CLT:
	                  msgCodec = new RegisterTMRequestCodec();
	                  break;
	              // TM注册结果消息
	              case MessageType.TYPE_REG_CLT_RESULT:
	                  msgCodec = new RegisterTMResponseCodec();
	                  break;
	              // RM注册消息
	              case MessageType.TYPE_REG_RM:
	                  msgCodec = new RegisterRMRequestCodec();
	                  break;
	              // RM注册结果消息
	              case MessageType.TYPE_REG_RM_RESULT:
	                  msgCodec = new RegisterRMResponseCodec();
	                  break;
	              // 分支事务二阶段提交消息
	              case MessageType.TYPE_BRANCH_COMMIT:
	                  msgCodec = new BranchCommitRequestCodec();
	                  break;
	              // 分支事务二阶段回滚消息
	              case MessageType.TYPE_BRANCH_ROLLBACK:
	                  msgCodec = new BranchRollbackRequestCodec();
	                  break;
	              // 全局事务状态上报消息
	              case MessageType.TYPE_GLOBAL_REPORT:
	                  msgCodec = new GlobalReportRequestCodec();
	                  break;
	              // 批量结果消息
	              case MessageType.TYPE_BATCH_RESULT_MSG:
	                  msgCodec = new BatchResultMessageCodec();
	                  break;
	              default:
	                  break;
	          }
	          if (msgCodec != null) {
	              return msgCodec;
	          }
	          try {
	              // 合并消息类型的消息处理类放在这里
	              // 当客户端或资源管理器向事务协调器发送消息的时候，如果发现有多个消息时发给同一个TC的，就可以合并成一个消息，在TC收到之后会再拆分成多个消息，提高RPC性能
	              msgCodec = getMergeRequestMessageSeataCodec(typeCode);
	          } catch (Exception exx) {
	          }
	          if (msgCodec != null) {
	              return msgCodec;
	          }
	          msgCodec = getMergeResponseMessageSeataCodec(typeCode);
	          return msgCodec;
	      }
	  ```
- 这种msgCodec实现里MessageSeataCodec接口。主要有获取消息类型和编码和解码这仨方法。
-
- 资源管理器注册消息的编码和解码
	- 编码
		- ```
		  protected <T> void doEncode(T t, ByteBuf out) {
		          // 父类的编码方式
		          super.doEncode(t, out);
		          // 转成RM注册请求对象
		          RegisterRMRequest registerRMRequest = (RegisterRMRequest)t;
		          // 得到资源ID
		          String resourceIds = registerRMRequest.getResourceIds();
		          if (resourceIds != null) {
		              // 资源id转成字节数组
		              byte[] bs = resourceIds.getBytes(UTF8);
		              // 写字节数据长度
		              out.writeInt(bs.length);
		              // 长度大于0，就写字节数组
		              if (bs.length > 0) {
		                  out.writeBytes(bs);
		              }
		          } else {
		              // 如果没有资源ID，就将字节数据长度设置成0
		              out.writeInt(0);
		          }
		      }
		  ```
	- request的定义
		- ```
		  blic class RegisterRMRequest extends AbstractIdentifyRequest implements Serializable {
		      // 构造函数，将应用ID和事务服务组设置成空
		      public RegisterRMRequest() {
		          this(null, null);
		      }
		      // 构造函数，设置
		      public RegisterRMRequest(String applicationId, String transactionServiceGroup) {
		          super(applicationId, transactionServiceGroup);
		      }
		  
		      // 资源ID，可能有多个
		      private String resourceIds;
		  
		      public String getResourceIds() {
		          return resourceIds;
		      }
		      
		      public void setResourceIds(String resourceIds) {
		          this.resourceIds = resourceIds;
		      }
		  
		      @Override
		      public short getTypeCode() {
		          return MessageType.TYPE_REG_RM;
		      }
		      }
		  ```
		- 父类中定义的需求编码/解码的成员变量
			- ```
			  public abstract class AbstractIdentifyRequest extends AbstractMessage {
			  
			      // 版本号
			      protected String version = Version.getCurrent();
			  
			      // 应用id
			      protected String applicationId;
			  
			      // 事务服务组
			      protected String transactionServiceGroup;
			  
			      // 附加数据，用来扩展
			      protected String extraData;
			  }
			  ```
			- 他对应的解码器是：AbstractIdentifyRequestCodec。对这四个成员变量的编码都和上述资源码一个逻辑，都是string类型。写入string的字节的长度，在写入字节数组。
	- 解码
		- 因为在super.doEncode(t, out);，所以是先对这四个成员变量加密了的。所以解码里要依次对这四个成员变量解密，在对资源ID进行解码
		- ```
		  public <T> void decode(T t, ByteBuffer in) {
		          RegisterRMRequest registerRMRequest = (RegisterRMRequest)t;
		  
		          if (in.remaining() < 2) {
		              return;
		          }
		          short len = in.getShort();
		          if (len > 0) {
		              if (in.remaining() < len) {
		                  return;
		              }
		              byte[] bs = new byte[len];
		              in.get(bs);
		              registerRMRequest.setVersion(new String(bs, UTF8));
		          } else {
		              return;
		          }
		          if (in.remaining() < 2) {
		              return;
		          }
		          len = in.getShort();
		  
		          if (len > 0) {
		              if (in.remaining() < len) {
		                  return;
		              }
		              byte[] bs = new byte[len];
		              in.get(bs);
		              registerRMRequest.setApplicationId(new String(bs, UTF8));
		          }
		  
		          if (in.remaining() < 2) {
		              return;
		          }
		          len = in.getShort();
		  
		          if (in.remaining() < len) {
		              return;
		          }
		          byte[] bs = new byte[len];
		          in.get(bs);
		          registerRMRequest.setTransactionServiceGroup(new String(bs, UTF8));
		  
		          if (in.remaining() < 2) {
		              return;
		          }
		          len = in.getShort();
		  
		          if (len > 0) {
		              if (in.remaining() < len) {
		                  return;
		              }
		              bs = new byte[len];
		              in.get(bs);
		              registerRMRequest.setExtraData(new String(bs, UTF8));
		          }
		  
		          int iLen;
		          if (in.remaining() < 4) {
		              return;
		          }
		          iLen = in.getInt();
		  
		          if (iLen > 0) {
		              if (in.remaining() < iLen) {
		                  return;
		              }
		              bs = new byte[iLen];
		              in.get(bs);
		              registerRMRequest.setResourceIds(new String(bs, UTF8));
		          }
		      }
		  ```
		- 大体的逻辑就是，先读取版本号的字节数组长度，如果长度为0，就把这个成员变量的值设置为空，如果大于0，要判断下刻度的字节数是否足够，不足够就返回，如果足够，读取对应长度的字符来转成对应的字符串，设置成成员变脸的值。
- 分支事务注册消息的编码和解码
	- 获取全局事务ID，分支事务类型，资源ID，事务全局锁数据，应用数据。进行编码。如果是字符串类型，那就按之前的方式。不同的是分支事务类型，他的转换方式如下:
		- ```
		  BranchType branchType = branchRegisterRequest.getBranchType();
		  // 2. Branch Type
		          out.writeByte(branchType.ordinal());
		  ```
	- 分支事务注册响应消息中只有一个branchId，分支事务ID，为long类型
		- ```
		      @Override
		      public <T> void encode(T t, ByteBuf out) {
		          super.encode(t, out);
		  
		          BranchRegisterResponse branchRegisterResponse = (BranchRegisterResponse)t;
		          out.writeLong(branchRegisterResponse.getBranchId());
		      }
		  
		      @Override
		      public <T> void decode(T t, ByteBuffer in) {
		          super.decode(t, in);
		  
		          BranchRegisterResponse branchRegisterResponse = (BranchRegisterResponse)t;
		          branchRegisterResponse.setBranchId(in.getLong());
		      }
		  ```
- 合并消息的编码解码
	- 合并消息时为了提升性能，把多个事务消息合并成一个。
		- MergedWarpMessageCodec 合并请求消息的编码解码器
		- ```
		  public <T> void encode(T t, ByteBuf out) {
		          MergedWarpMessage mergedWarpMessage = (MergedWarpMessage)t;
		          // 得到消息列表
		          List<AbstractMessage> msgs = mergedWarpMessage.msgs;
		          List<Integer> msgIds = mergedWarpMessage.msgIds;
		          // 分配1kb大小的buffer
		          final ByteBuf buffer = Unpooled.buffer(1024);
		          // 总长度先写成0，计算到总长度之后再进行修改
		          buffer.writeInt(0); // write placeholder for content length
		          // 写消息数量
		          buffer.writeShort((short)msgs.size());
		          for (final AbstractMessage msg : msgs) {
		              // 遍历所有消息。分配1kb大小的buffer
		              final ByteBuf subBuffer = Unpooled.buffer(1024);
		              // 得到消息类型码，得到消息编码器，进行编码，写消息类型码和字节数据
		              short typeCode = msg.getTypeCode();
		              MessageSeataCodec messageCodec = MessageCodecFactory.getMessageCodec(typeCode);
		              messageCodec.encode(msg, subBuffer);
		              buffer.writeShort(msg.getTypeCode());
		              buffer.writeBytes(subBuffer);
		          }
		          // 将msgId都写进去
		          for (final Integer msgId : msgIds) {
		              buffer.writeInt(msgId);
		          }
		          // 得到待合并所有消息的总长度
		          final int length = buffer.readableBytes();
		          final byte[] content = new byte[length];
		          // 将总长度写进去，总长度不包括写总长度本身的4byte
		          buffer.setInt(0, length - 4);  // minus the placeholder length itself
		          buffer.readBytes(content);
		  
		          if (msgs.size() > 20) {
		              if (LOGGER.isDebugEnabled()) {
		                  LOGGER.debug("msg in one packet:" + msgs.size() + ",buffer size:" + content.length);
		              }
		          }
		          out.writeBytes(content);
		      }
		  ```
		- [总长度][消息数量][消息1类型码][消息1字节数组][消息2类型码][消息2字节数组].....
		- 若合并的消息大小大于20，打印日志输出，检测是否出现消息积压
		- 如果超过100个消息合并为1个消息，需要提前规划服务端扩容
	- decode解码
		- 先读出合并消息的总长度，根据总长度读出整个字节数据，放在buffer里
		- 从buffer里读出消息数量，循环解码这个合并消息里包含的每个子消息
			- 读出消息的类型码
			- 根据类型码，创建对应的消息对象
			- 根据类型码，创建编码器对象
			- 调用解码器的decode方法解码出子消息
			- 将子消息放到消息列表里
		- merge主要是为了提高吞吐量
	-