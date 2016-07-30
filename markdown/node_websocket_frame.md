# WebSocket数据帧编码解码分片

### 握手加密字符串

```
258EAFA5-E914-47DA-95CA-C5AB0DC85B11
```

### 数据帧结构

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Payload Data continued ...                |
 +---------------------------------------------------------------+
```

#### 数据说明

```
FIN      1bit 表示信息的最后一帧，flag，也就是标记符
RSV 1-3  1bit each 以后备用的 默认都为 0
Opcode   4bit 帧类型
Mask     1bit 掩码，是否加密数据，默认必须置为1
Payload  7bit 数据的长度
Masking-key      1 or 4 bit 掩码
Payload data     (x + y) bytes 数据
Extension data   x bytes  扩展数据
Application data y bytes  程序数据
```

### Opcode

```
 |Opcode  | Meaning                             | Reference |
-+--------+-------------------------------------+-----------|
 | 0      | Continuation Frame                  | RFC 6455  |
-+--------+-------------------------------------+-----------|
 | 1      | Text Frame                          | RFC 6455  |
-+--------+-------------------------------------+-----------|
 | 2      | Binary Frame                        | RFC 6455  |
-+--------+-------------------------------------+-----------|
 | 8      | Connection Close Frame              | RFC 6455  |
-+--------+-------------------------------------+-----------|
 | 9      | Ping Frame                          | RFC 6455  |
-+--------+-------------------------------------+-----------|
 | 10     | Pong Frame                          | RFC 6455  |
-+--------+-------------------------------------+-----------|
```

## 代码

```javascript
var WSapi = {
	// 解码数据类型
	decodeFrame : function(e){
		// 检查数类型
		assert(e instanceof Buffer);
		
		var i = 0, s = [];
		var frame = {
			FIN: e[i] >> 7,				// 结束标志
			opcode: e[i++] & 0x0f,		// 数据类型
			mask: e[i] >> 7,			// 是否掩码加密
			length: e[i++] & 0x7f		// 长度
		};
		
		// 处理特殊长度
		if(frame.length == 126){
			// 如果长度是126，使用后2byte作为长度
			frame.length = e[i++] << 8 + e[i++]
		}else if(frame.length ==127){
			// 如果长度是127，使用后8byte作为长度，跳过4byte作为长整型
			i += 4, frame.length = (e[i++] << 24) + (e[i++] << 16) + (e[i++] << 8) + e[i++];
		}
		
		if(frame.mask){
			// 如果使用掩码加密，先获取掩码信息
			var mask = [e[i++], e[i++], e[i++], e[i++]];
			frame.maskKey = mask;
			
			// 所有数据轮流和掩码发生性关系
			for(var j = 0; j < frame.length; j++){
				s.push(e[i + j] ^ mask[j % 4]);
			}
			
			s = new Buffer(s);
		}else{
			// 如果不加密，则直接获取数据
			s = e.slice(i, frame.length);
		}
		
		// 如传输的是文本，则处理一下
		if(frame.opcode == 1) s = s.toString();
		
		frame.data = s;
		
		return frame;
	},
	
	// 编码数据
	encodeFrame : function(data){
		assert(data instanceof Buffer || 'string' == typeof data);
		
		// 储存分片结果的数组，大于125字节的都分片传输
		var result = [], peace_length = Math.ceil(data.length / 125);
		
		// opcode
		var opcode = data instanceof Buffer && 2 || 1;
		
		for(var i = 0; i < peace_length; i++){
			var tmp_buf = data.slice(i * 125, 125);
			var temp = [];
			
			// 填充第一个字节
			if(peace_length == 1){
				temp.push((1 << 7) + opcode);
			}else{
				if(i == 0){
					// 第一个数据包，FIN = 0，opcode = opcode
					temp.push(opcode);
				}else if(i == peace_length - 1){
					// 最后一个数据包，FIN = 1，opcode = 0
					temp.push(1 << 7);
				}else{
					// 中间的数据包，FIN = 0，opcode = 0
					temp.push(0);
				}
			}
			
			// 填充长度字节，由于分片长度总是小于125，所以只要填充第二个字节
			// 永远不使用掩码
			temp.push(tmp_buf.length);
			
			result.push(Buffer.concat([new Buffer(temp), (tmp_buf instanceof Buffer && tmp_buf || new Buffer(tmp_buf))]));
		}
		
		return result;
	}
}
```