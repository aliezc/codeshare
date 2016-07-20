# 【Javascript】Node.js请求地址规则判断

```javascript
var assert = require('assert');
var http = require('http');

/*
 * 匹配路由规则
 * @param {http.IncomingMessage} 请求
 * @param {http.ServerResponse} 响应
 * @param {Array} 路由规则列表
 * @param {function} 匹配成功时执行的回调
 */
function matchRules(req, res, rules, cb){
	assert(req instanceof http.IncomingMessage);
	assert(res instanceof http.ServerResponse);
	assert(rules instanceof Array);
	
	/*
	规则格式：
	[{
		matcher: /^\/$/,		// 正则表达式或字符串
		handle: function(){}	// 函数，接收req和res两个参数
	}]
	*/
	
	// 真实请求地址，过滤参数和hash
	var url = require('url').parse(req.url).pathname;
	
	// 缺省响应方法
	var defResponse = function(req, res){
		res.statusCode = 404;
		res.end();
	};
	
	var op = function(rule){
		// 调用执行函数
		var func = 'function' == typeof rule.handle ? rule.handle : defResponse;
		process.nextTick(function(){
			func.call(null, req, res);
		});
		
		// 执行回调
		if(typeof cb == 'function') cb.call(null, rule);
	}
	
	for(var i = 0; i < rules.length; i++){
		// 判断规则类型
		if(rules[i].matcher instanceof RegExp){
			// 正则类型
			if(rules[i].matcher.test(url)){
				// 匹配成功
				
				op.call(null, rules[i]);
				return;
			}
		}else if(typeof rules[i].matcher == 'string'){
			// 字符串
			if(url == rules[i].matcher){
				// 匹配成功
				
				op.call(null, rules[i]);
				return;
			}
		}
	}
	
	// 没有匹配结果
	res.statusCode = 404;
	res.end();
	
	// 执行回调
	if(typeof cb == 'function') cb.call(null, null);
}
```