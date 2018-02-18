# 我的股市相关学习

## 每日获取股票信息
个人为了获得每日股票的信息，写了一个脚本，自动从网页上获取股票的信息。
代码文件名为haoStock.lua，放在openwrt的路由器上，通过crond定时在15点10分执行，把当日的股票情况发邮件到指定的邮箱了。
运行下面代码需要有lua环境并支持luasocket

```
#!/usr/bin/lua

--下面用来分割字符串
function split( str,reps )
    local resultStrList = {}
    string.gsub(str,'[^'..reps..']+',function ( w )
        table.insert(resultStrList,w)
    end)
    return resultStrList
end

function get_date()
    local date = os.date("%Y%m%d")
	return date
end

--检查当日是否是开市的日子
function verify_date()
    local http = require("socket.http")
    local url = "http://qt.gtimg.cn/q=sh000001"
	local resp = http.request(url)
	local mytab = split(resp,'~')
	local stock_date =string.sub(mytab[31],1,8)
	if stock_date == get_date() then
	    return stock_date 
	else
	    return nil
	end
end

--产生最终文件
function gen_file(file_name,code_file)
    local http = require("socket.http")
	local pre_url = "http://qt.gtimg.cn/q="
    local file_w = io.open(file_name,"w")
	local file_r = io.open(code_file,"r")
	local lines = {}
	io.input(file_r)
	io.output(file_w)
	io.write("name,turnover rate,previous trading day close,opening,price,StockCode\n")
    for line in io.lines() do   --通过迭代器访问每一个数据
        lines[#lines + 1] = line
    end
    for _,l in ipairs(lines) do
	    local url = pre_url..l
	    local resp = http.request(url)
        --print(resp)
		if resp then
    	    local mytab = split(resp,'~')
	    	if mytab[2] then
		        io.write(mytab[2],",")
		        io.write(mytab[39],",")
	            io.write(mytab[5],",")
	            io.write(mytab[6],",")
	            io.write(mytab[4],",")
		        io.write(l,"\n")
		    end
	    end
    end
	io.close(file_r)
	io.close(file_w)
end

--发送邮件及附件
function sendmail(file_name)
    local smtp = require("socket.smtp")
    local mime = require("mime")
    local ltn12 = require("ltn12")
	local rcpt = {
    "<xxx@xxx>"			--此处为收件人
    }
    local source = smtp.message{
        headers = {
            from = "asd<dasfa@xxx>", 	--填上发件人
            to = "you<xxx@xxx>",	--填上收件人
            subject = "这是"..get_date().."股票情况"
        },
        body = {
            [1] = { 
            body = mime.eol(0, [[
                name代表股票名,
				price代表统计时的股价,
				previous trading day close代表前一交易日收盘价,
				opening代表今日开盘价,
				turnover rate代表换手率,
				StockCode代表股票代码
                ]])
            },
            [2] = { 
                headers = {
                    ["content-type"] = 'application/octet-stream; name='..file_name,
                    ["content-disposition"] = 'attachment; filename='..file_name,
                    ["content-description"] = '股票数据',
                    ["content-transfer-encoding"] = "BASE64"
                },
                body = ltn12.source.chain(
                    ltn12.source.file(io.open(file_name, "rb")),
                    ltn12.filter.chain(
                        mime.encode("base64"),
                        mime.wrap()
                    )
                )
            },
        }
    }

    r, e = smtp.send{
        from = "<asda@163.com>",	--发件人邮箱
        rcpt = rcpt,
        source = source,
	    server = "smtp.163.com",	--smtp服务器地址
        user = "usermail",	--此处填写发送邮箱的用户名
        password = "password"	--此处填邮箱的密码
    }
    if not r then
        print(e)
--    else
--        print("send ok!")
    end
	return 0
end

--获取http://quote.eastmoney.com/stocklist.html
function get_stockcodehtml(html_file)
    local http = require("socket.http")
	local ltn12 = require("ltn12")
	local file = io.open(html_file,"w")
	--io.output(file)
	local res, code, response_headers = http.request{
        url = "http://quote.eastmoney.com/stocklist.html",
        sink = ltn12.sink.file(file)
        --proxy = "Mozilla/4.0 (compatible; MSIE 5.5; Windows NT)"
    }
	if code == 200 then
        return true
	else
	    return false
	end
end

--解析出所有股票代码
function get_stockcode(html_file,code_file)
    local file_r = io.open(html_file,"r")
	local file_w = io.open(code_file,"w")
	local stockcode = "s[hz]%d%d%d%d%d%d"
	local lines = {}
	io.input(file_r)
	io.output(file_w)
    for line in io.lines() do   --通过迭代器访问每一个数据
        lines[#lines + 1] = line
    end
    table.sort(lines)  --排序，Lua标准库的table库提供的函数。
    for _,l in ipairs(lines) do
	    local spos,epos = string.find(l,stockcode)
	    if spos and epos then
            io.write(string.sub(l,spos,epos),"\n")
		end
    end
	io.close(file_r)
	io.close(file_w)
end

--更新股票代码
function update_stockcode(html_file,code_file)
    if get_stockcodehtml(html_file) then
        get_stockcode(html_file,code_file)
    else
        print("fail")
    end
	os.remove(html_file)
end

function main()
    local stock_date = verify_date()
	if not stock_date then
	    return
	else
	    update_stockcode("/tmp/stock.html","/tmp/stockcode")	--文件位置根据实际系统情况修改
	    local file_name = "/tmp/"..stock_date..".csv"
	    gen_file(file_name,"/tmp/stockcode")
		sendmail(file_name)
		print("sendmail ok.")
		os.remove(file_name)
	end
	return 0
end

main()
```

## 移动平均线

移动平均线（MA，moving average）是以道琼斯的“平均成本概念”为理论基础，采用统计学中“移动平均”的原理，
将一段时期内的股票价格平均值连成曲线，用来显示股价的历史波动情况，进而反映股价指数未来发展趋势的技术分析方法。
它是道氏理论的形象化表述。

移动平均线的计算方法就是求连续若干天的收盘价的算数平均。天数就是MA的参数。
在技术分析领域中，移动平均线是必不可少的指标工具。移动平均线利用统计学上的“移动平均”原理，
将每天的市场价格进行移动平均计算，求出一个趋势值，用来作为价格走势的研判工具。

计算公式：
MA=(C1+C2+C3+C4+C5+...+Cn)/n,C为收盘价，n为移动平均周期数

5日移动平均价格计算方法为：
MA5=(前四天收盘价+前三天收盘价+前天收盘价+昨天收盘价+今天收盘价)/5

移动平均线依时间长短可分为三种，即短期移动平均线，中期移动平均线，长期移动平均线
短期移动平均线，一般以5日或10日为计算期间
中期移动平均线，大多以20日、60日为计算期间
长期移动平均线，大多以100天和200天为计算期间

简单移动平均线(SMA):又称“算数移动平均线”，是指对特定期间的收盘价进行简单平均化的意思。一般所提及之移动平均线即指简单移动平均线(SMA)

加权移动平均线(WMA):加权移动平均线(Weighted Moving Average 简称WMA)，是一种按时间进行加权运算的移动平均线。时间越近的价格权重越大。
计算方式是基于加权移动平均线日数，将每一个之前日数比重提升。每一价格会乘以一个比重，最新的价格会有量大的比重，其之前的每一日的比重将会递减。
加权移动平均线是移动平均线(MA)的改良。

指数平滑移动平均线(ENA):指数平滑移动平均线EXPMA(Exponential Moving Average),将解决一旦价格已脱离均线差值扩大，而平均线未能立即反应，EXPMA可以减少类似缺点。

---

## 均线
在日K线图中除了标准的价格K线以外，另外还有4条线，分别是日线，黄线，紫线，绿线依次分别表示：5日，10日，20日和60日移动平均线，通过定义这4条线与价格K线的交叉，
就可以形成不同的均线模型。

---

## 均线模型
利用均线平滑的特点，可以发现均线与价格K线会有交叉，各均线之间也有交叉，我们可以通过这些交叉点判断交易信号。

黄金交叉，当10日均线由下往上穿越30日均线，10日均线在上，30日均线在下，其交叉点就是黄金交叉，黄金交叉是多头的表现，出现黄金交叉后，后市会有一定的涨幅空间，
这是进场的最佳时机。

死亡交叉，当30日均线与10日均线交叉时，30日均线由下往上穿越10日平均线，形成30日平均线在上，10日均线在下时，其交叉点称之为“死亡交叉”，
“死亡交叉”预示空头市场来临，股市将下跌，此时是出场的最佳时机。

---

## 局限性
移动平均线是股价定型后产生的图形，反映较慢，只适用于日间交易
移动平均线不能反映股价在当日的变化及成交量的大小，不适用于日内交易
移动平均线是趋势性模型，如果股价未形成趋势，只是频繁波动，模型不适用

---