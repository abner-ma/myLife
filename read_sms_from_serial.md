# read_sms_from_serial

## 编写脚本通过串口获得短信的代码
使用脚本通过串口获得CDMA芯片的短信，难点在于解析PDU协议，具体的协议规则可从网上查询
下面是我写的两个版本，正常操作可用，偶尔会出现问题，未查明原因。
### lua版本
为了能在支持USB口的路由器上读取串口数据，所以编写了cdma.lua脚本，lua比较轻量，路由系统中自带解析器。
但由于lua功能有限，不能把unicode转换成中文，所以使用test.html来获得中文内容。
cdma.lua可以自动解析所有的短信，并解析出发送人号码，短信内容以unicode代码显示，以下是cdma.lua代码
```
#!/usr/bin/lua

function read_from_serial( dev_name,answer )
    local rserial=io.open(dev_name,"r")
	while true do
		get_str=rserial:read("*l")
		print(get_str)
		local spos,epos = string.find(get_str,answer)
		if spos and epos then
            return true
		end
		spos,epos = string.find(get_str,"ERROR")
		if spos and epos then
            return false
		end
	end
end

function write_to_serial( dev_name,cmd )
    local wserial=io.open(dev_name,"w")
	wserial:write(cmd)
	wserial:flush()
end

function execute_cmd_and_return(dev_name,cmd,ret_str)
	write_to_serial(dev_name,cmd)
	if read_from_serial(dev_name,ret_str) then
		return true
	end
	return false
end

function decode_sms(str_msg)
    local str_msg_len = string.len(str_msg)
    local phone_len_code = string.sub(str_msg,1,2)
    local pos = 1
    local phone_len = tonumber(phone_len_code, 16)
    print("phone num length:"..phone_len)
	pos = pos + 4
	local phone_num_code = string.sub(str_msg,pos,pos+phone_len*2)
    local phone_num="",temp
    for i=0,phone_len,1 do
            temp=i*2+1
            ch = string.char(tonumber(string.sub(phone_num_code,temp,temp+1),16))
			phone_num=phone_num..ch
	end
    print("phone num is:"..phone_num)

    pos = pos + phone_len*2 + 28

    local min_len_code = string.sub(str_msg,pos,pos+1)
    local min_len = tonumber(min_len_code,16)

    pos = pos + min_len*2 + 2
	
	local sms_len_code = string.sub(str_msg,pos,pos+1)
	local sms_len = tonumber(sms_len_code,16)
	
	print("sms length is:"..sms_len)
	
	pos = pos + 2
	local sms_txt = string.sub(str_msg,pos,pos+sms_len*2-1)
	sms_len = sms_len/2
	local str = ""
	for i=0,sms_len-1,1 do
            temp=4*i+1
            ch = string.sub(sms_txt,temp,temp+3)
            str=str.."\\u"..ch
	end
	print("sms txt is:"..str)
end

function display_one_sms(dev_name,num)
	write_to_serial(dev_name,"AT+CMGR="..num.."\r")
	local rserial=io.open(dev_name,"r")
	while true do
		get_str=rserial:read("*l")
		if string.len(get_str) > 22 then
			print(get_str)
			decode_sms(get_str)
            return true
		end
		local spos,epos = string.find(get_str,"ERROR")
		if spos and epos then
            return false
		end
	end
	return true
end

function decode_all_sms(dev_name)
    for i = 0,30,1 do
	    if not display_one_sms(dev_name,i) then
		    break
		end
	end
end

function display_all_sms(dev_name)
	write_to_serial(dev_name,"AT+CMGL\r")
	if read_from_serial(dev_name,"OK") then
		return true
	end
	return false
end

function delete_all_sms(dev_name)
	for i=1,30,1 do
		write_to_serial(dev_name,"AT+CMGD="..i.."\r")
		if not read_from_serial(dev_name,"OK") then
			break
		end
	end
	return true
end

function main()
	if execute_cmd_and_return("/dev/ttyUSB0","AT+CPIN?\r","OK") then
		if execute_cmd_and_return("/dev/ttyUSB0","AT+CMGF=0\r","OK") then
			display_all_sms("/dev/ttyUSB0")
			decode_all_sms("/dev/ttyUSB0")
			--delete_all_sms("/dev/ttyUSB0")
		end
	else
		print("error")
	end
end

main()
```
下面是test.html的代码
```
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="utf-8">
<meta content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" name="viewport" />
<title>在线unicode转中文,中文转unicode</title>
</head>
<body>

<form id="JSONVYasuo" method="post" action="http://www.bejson.com/." name="JSONVYasuo">
<input type="hidden" id="reformat" value="1" />
<input type="hidden" id="compress" value="0" />
<div>
<textarea id="json_input" name="json_input" class="json_input" rows="10" cols="80" spellcheck="false" placeholder="Enter JSON to validate"></textarea>
</div>
<div class="validateButtons clear">
<div class="left">
<input type="button" value="Unicode转中文" onclick="u2h()" />
<input type="button" value="中文转Unicode" onclick="h2u()" />
<input type="button" value="中文符号转英文符号" title="如果您从他人技术博客copy代码时,可能会因为json中重要符号被替换成中文字符而导致校验失败,这时就可以使用本功能替换" onclick="cnChar2EnChar()" />
</div>
</div>
</form>
<script>
			/**
			1 压缩
			2 转义
			3 压缩转义
			*/
			function yasuo(ii){
				 var txtA = document.getElementById("json_input");
				 var text = txtA.value;
					if(ii==1||ii==3){
						 text = text.split("\n").join(" ");
						var t = [];
						var inString = false;
						for (var i = 0, len = text.length; i < len; i++) {
							var c = text.charAt(i);
							if (inString && c === inString) {
								// TODO: \\"
								if (text.charAt(i - 1) !== '\\') {
									inString = false;
								}
							} else if (!inString && (c === '"' || c === "'")) {
								inString = c;
							} else if (!inString && (c === ' ' || c === "\t")) {
								c = '';
							}
							t.push(c);
						}
						text= t.join('');
					}
					if(ii==2||ii==3){
						 text = text.replace(/\\/g,"\\\\").replace(/\"/g,"\\\"");
					}
					if(ii==4){
					 text = text.replace(/\\\\/g,"\\").replace(/\\\"/g,'\"');
					}
					 txtA.value = text;
			}
			
			String.prototype.trim=function()
		{
		     return this.replace(/(^\s*)|(\s*$)/g, '');
		}
			var GB2312UnicodeConverter={
		  ToUnicode:function(str){
		    var txt= escape(str).toLocaleLowerCase().replace(/%u/gi,'\\u');
			//var txt= escape(str).replace(/([%3F]+)/gi,'\\u');
			return txt.replace(/%7b/gi,'{').replace(/%7d/gi,'}').replace(/%3a/gi,':').replace(/%2c/gi,',').replace(/%27/gi,'\'').replace(/%22/gi,'"').replace(/%5b/gi,'[').replace(/%5d/gi,']').replace(/%3D/gi,'=').replace(/%20/gi,' ').replace(/%3E/gi,'>').replace(/%3C/gi,'<').replace(/%3F/gi,'?').replace(/%5c/gi,'\\');//
		  }
		  ,ToGB2312:function(str){
		    return unescape(str.replace(/\\u/gi,'%u'));
		  }
		};
		
		function u2h(){
			 var txtA = document.getElementById("json_input");
			 var text = txtA.value;
			 text = text.trim();
			// text = text.replace(/\u/g,"");
			 txtA.value = GB2312UnicodeConverter.ToGB2312(text);	 
		}
		
		function h2u(){
			var txtA = document.getElementById("json_input");
			 var text = txtA.value;
			 text = text.trim();
			// text = text.replace(/\u/g,"");
			 txtA.value = GB2312UnicodeConverter.ToUnicode(text);
		}
		
		function cnChar2EnChar(){
		 var txtA = document.getElementById("json_input");
		  var str = txtA.value;
		str = str.replace(/\’|\‘/g,"'").replace(/\“|\”/g,"\"");
		str = str.replace(/\【/g,"[").replace(/\】/g,"]").replace(/\｛/g,"{").replace(/\｝/g,"}");
		str = str.replace(/，/g,",").replace(/：/g,":");
		 txtA.value = str;
		}
		</script>
</div>
</body></html>
```

### python版本
下面是一个python版本，适合在树莓派上运行
```
# -*- coding: utf-8 -*
import serial
import time
import string

import os.path
import os

class MYSendMail(object):

    To = "njuptgggzs@163.com"
    Cc = "969023674@qq.com"
    #Cc = "18115622337@189.cn 2008sunyunlong@163.com"
    title_str = ""
    text_msg = ""

    def __init__(self):
        self.title_str="短信邮件"
        self.text_msg="邮件正文"

    def sendMail(self,txt):
        # 构造MIMEText对象做为邮件显示内容并附加到根容器
        cmd = "echo \""+txt+"\"|mutt -s "+self.title_str+" "+self.To + " -c "+self.Cc
#        print cmd
        os.system(cmd)

class MYcdmactr(object):
    ser = serial.Serial()
    def __init__(self):
# 打开串口
        self.ser = serial.Serial("/dev/ttyAMA0", 9600)
    def setPDU(self):
        cdma_at = "AT+CMGF=0\r"
        self.ser.write(cdma_at)
        self.ser.flushInput()
        while True:
            count = self.ser.inWaiting()
            if count != 0:
                recv = self.ser.read(count)
                if string.find(recv, "OK\r\n")!=-1:
                    break
                elif string.find(recv, "ERROR")!=-1:
                    break
                self.ser.flushInput()
                time.sleep(0.1)
    def decode_sms(self,ser_msg):
        ser_msg_len = len(ser_msg)
        ret_str = ""

        pos = 0
        phone_len_code = ser_msg[pos:pos+2]
        phone_len = int(phone_len_code,16)
        ret_str = "phone num length:%d" % phone_len

        pos = pos + 4
        phone_num_code = ser_msg[pos:pos+phone_len*2]

        phone_num = ""
        for i in range(0,phone_len):
            temp=i*2
            ch = chr(int(phone_num_code[temp:temp+2],16))
            phone_num=phone_num+ch
        ret_str = ret_str + "phone num is:"+phone_num

        pos = pos + phone_len*2 + 28

        min_len_code = ser_msg[pos:pos+2]
        min_len = int(min_len_code,16)

        pos = pos + min_len*2 + 2

        sms_len_code = ser_msg[pos:pos+2]
        sms_len = int(sms_len_code,16)
        ret_str = ret_str + "sms length is : %d" % sms_len

        pos = pos + 2
        sms_txt = ser_msg[pos:pos+sms_len*2]

        sms_len = sms_len/2

        str=""

        for i in range(0,sms_len):
            temp=4*i
            ch = "\u"+sms_txt[temp:temp+4]
            str=str+ch

        ret_str = ret_str + "sms txt is:"
        ret_str = ret_str + str.decode('raw_unicode_escape')
        return ret_str

    def display_sms_num(self,num):
        num_str = "%d" % num
        cdma_at = "AT+CMGR="+num_str+"\r"
        print cdma_at
        self.ser.write(cdma_at)
        self.ser.flushInput()
        sms_str=''
        while True:
            count = self.ser.inWaiting()
            if count != 0:
                recv = self.ser.read(count)
                print recv
                self.ser.flushInput()
                sms_str = sms_str + recv
                if string.find(sms_str, "OK\r\n")!=-1:
                    break
                elif string.find(recv, "ERROR")!=-1:
                    break
                time.sleep(0.1)
        sms_list = sms_str.splitlines()
        ret_str = "num of sms : " + num_str+"\n"
        if len(sms_list) == 6:
            ret_str = ret_str + self.decode_sms(sms_list[3])
        return ret_str

    def display_all_sms(self):
        ret_str = ""
        for num in range(0,30):
            ret_str = ret_str + self.display_sms_num(num) + "\n"
        return ret_str

    def delete_sms_num(self,num):
        num_str = "%d" % num
        cdma_at = "AT+CMGD="+num_str+"\r"
        self.ser.write(cdma_at)
        self.ser.flushInput()
        while True:
            count = self.ser.inWaiting()
            if count != 0:
                recv = self.ser.read(count)
                if string.find(recv, "OK\r\n")!=-1:
                    break
                elif string.find(recv, "ERROR")!=-1:
                    break
                self.ser.flushInput()
                time.sleep(0.1)

    def delete_all_sms(self):
        for num in range(0,30):
            self.delete_sms_num(num)

    def run(self):
        cdma_str=""
        while True:
        # 获得接收缓冲区字符
            count = self.ser.inWaiting()
            if count != 0:
            # 读取内容并回显
                recv = self.ser.read(count)
                cdma_str = cdma_str + recv
            #ser.write(recv)
        # 清空接收缓冲区
                self.ser.flushInput()
        # 必要的软件延时
                time.sleep(5)
            if string.find(cdma_str,"SMS READY\r")!=-1:
                break

    def destory(self):
        if self.ser != None:
            self.ser.close()

if __name__ == '__main__':
    test = MYcdmactr()
    test.setPDU()
    test.run()
    print "after run"
    ret_str = test.display_all_sms()
    mail_obj=MYSendMail()
    mail_obj.sendMail(ret_str.encode("utf-8"))
    test.delete_all_sms()
    test.destory()
```