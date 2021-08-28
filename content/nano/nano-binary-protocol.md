---
title: "Nano Binary Protocol"
date: 2021-08-26T10:40:25+08:00
weight: 2
draft: false
---

### message.Encode
```GO
func Encode(m *Message) ([]byte, error) {  
	if invalidType(m.Type) {  
		return nil, ErrWrongMessageType  
	}  
	buf := make([]byte, 0)  
	// m.Type 最小时0x00 最大是0x03,左移一位，flag为0,2,4,6  
	flag := byte(m.Type) << 1  

	// 如果路由压缩 flag|1 为1,3,5,7  
	code, compressed := routes[m.Route]  
	if compressed {  
		flag |= msgRouteCompressMask  
	}  
	buf = append(buf, flag)  

	if m.Type == Request || m.Type == Response {  
		n := m.ID  
		// variant length encode  
		for {  
			//b是n/128余数，然后n按位右移7,相当于n、128。  
			//如果n为0，将b添加到buf，  
			//如果n不为0，b+128 添加到buf，并继续执行相同的逻辑  
			b := byte(n % 128)  
			n >>= 7  
			if n != 0 {  
				buf = append(buf, b+128)  
			} else {  
				buf = append(buf, b)  
				break  
			}  
		} 
	}  
	// 如果有route  
	// 如果路由压缩：code 按位右移8，然后与0xff做位与操作，最后code只保留低八位，高八位补零。0xff为11111111  
	if routable(m.Type) {  
		if compressed {  
			buf = append(buf, byte((code>>8)&0xFF))  
			buf = append(buf, byte(code&0xFF))  
		} else {  
			buf = append(buf, byte(len(m.Route)))  
			buf = append(buf, []byte(m.Route)...)  
		} 
	}
	
	buf = append(buf, m.Data...)  
	return buf, nil  
}
```
