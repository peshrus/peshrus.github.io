---
layout: post
title:  "A fix of SMS UI for MTS 4G Wi-Fi router 874FT"
date:   2019-11-06 18:13:00 +0300
categories: mts huawei 874FT fix
---
# Long Story Short

On the 2nd of October, I bought an [MTS 4G Wi-Fi router 874FT produced by Huawei](https://spb.mts.ru/personal/mobilnaya-svyaz/mobilniy-internet/modemy-i-routery/). 
It worked well and displayed SMS in a way below:

![MTS 4G Wi-Fi router 874FT SMS UI](/assets/2019-11-06-mts-huawei-4g-wi-fi-router-fix/sms-ui.jpg)

When I changed my plan on the 28th of October to a new one I stopped to receive SMS messages and could 
not realize why. That is how I found the bug.

# The Bug Localization
The page on the screenshot above had the following URL `http://192.168.1.1/index.html#sms` and it 
had the following error in the browser console:
```
Uncaught ReferenceError: result is not defined
    at eval (eval at <anonymous> (jquery-1.8.2.min.js:2), <anonymous>:278:27)
    at Object.error (service.js:380)
    at k (jquery-1.8.2.min.js:2)
    at Object.fireWith [as rejectWith] (jquery-1.8.2.min.js:2)
    at y (jquery-1.8.2.min.js:2)
    at XMLHttpRequest.d (jquery-1.8.2.min.js:2)
```

`service.js`:
```javascript
/**
 * 获取短消息数据
 * @method getSMSMessages
 */
function getSMSMessages(param, callback) {
    $.ajax({
        type: 'GET',
        url: "/goform/goform_get_cmd_process",
        data: param,
        cache: false,
        dataType: "json",
        success: function(data) {
            callback(true, data.messages || []);
        },
        error: function() {
            callback(false, "error_info");
        }
    });
}
```

The line `380` was `callback(false, "error_info")` and the real `callback` was:
```javascript
    // 获取消息
    function sms_getMessage(param) {
        sms_getSmsCapability()
       /* if (sms_capacity.sms_nv_rev_total == 0) {       
     hideLoading()                               
     return                                      
 }                                               */
        getSMSMessages(param, function(flag, data) {
            if (flag) {
                list_all_message = []
                $.each(data, function(index, item) {
                    item.content = decodeMessage(escapeMessage(item.content));
                    item.date = transTime('20' + item.date);
                    list_all_message.push(item)
                })
                sms_renderMessage()
            } else {
                hideLoading()
                showAlert(result.data)
            }
        })
    }
```

The problem was in the line `showAlert(result.data)`:
```
ReferenceError: result is not defined
    at eval (eval at <anonymous> (eval at <anonymous> (http://192.168.1.1/js/jquery-1.8.2.min.js:2:14070)), <anonymous>:1:1)
    at eval (eval at <anonymous> (http://192.168.1.1/js/jquery-1.8.2.min.js:2:14070), <anonymous>:278:17)
    at Object.error (http://192.168.1.1/js/service.js:380:13)
    at k (http://192.168.1.1/js/jquery-1.8.2.min.js:2:16920)
    at Object.fireWith [as rejectWith] (http://192.168.1.1/js/jquery-1.8.2.min.js:2:17707)
    at y (http://192.168.1.1/js/jquery-1.8.2.min.js:2:80829)
    at XMLHttpRequest.d (http://192.168.1.1/js/jquery-1.8.2.min.js:2:86374)
```

However, it was just another issue showing the original one. 

To detect the issue I needed to modify `service.js` in my browser:
```javascript
/**
 * 获取短消息数据
 * @method getSMSMessages
 */
function getSMSMessages(param, callback) {
    $.ajax({
        type: 'GET',
        url: "/goform/goform_get_cmd_process",
        data: param,
        cache: false,
        dataType: "json",
        success: function(data) {
            callback(true, data.messages || []);
        },
        error: function(jqXHR, textStatus, errorThrown) {
            callback(false, "error_info");
        }
    });
}
```

It showed me the following message:
```
errorThrown: SyntaxError: Unexpected token  in JSON at position 2149 at JSON.parse (<anonymous>)
    at parseJSON (http://192.168.1.1/js/jquery-1.8.2.min.js:2:13523)
    at cD (http://192.168.1.1/js/jquery-1.8.2.min.js:2:5883)
    at y (http://192.168.1.1/js/jquery-1.8.2.min.js:2:80684)
    at XMLHttpRequest.d (http://192.168.1.1/js/jquery-1.8.2.min.js:2:86374)
```

# The Root Cause & The Fix

`service.js` requested the following URL 
`http://192.168.1.1/goform/goform_get_cmd_process?cmd=sms_data_total&data_per_page=100&mem_store=1&order_by=order%20by%20id%20desc&page=0&tags=12` 
which returned something like that:
```json
{
  "messages": [
    {
      "id": "20",
      "number": "MTC",
      "content": "<ENCODED_CONTENT>",
      "tag": "0",
      "date": "19,10,03,21,50,55,+12",
      "draft_group_id": ""
    }
  ]
}
```

In my case, after the plan change, I had the following SMS among others:

![A wrong number](/assets/2019-11-06-mts-huawei-4g-wi-fi-router-fix/wrong-number.jpg)

The number was the culprit. To fix it and to delete that message I needed to fix the JS function in 
the following way:
```javascript
/**
 * 获取短消息数据
 * @method getSMSMessages
 */
function getSMSMessages(param, callback) {
    $.ajax({
        type: 'GET',
        url: "/goform/goform_get_cmd_process",
        data: param,
        cache: false,
        dataType: "text",
        success: function(text) {
            let data = $.parseJSON(text.replace(/"MTS[^"]*"/, '"MTC"'));
            callback(true, data.messages || []);
        },
        error: function() {
            callback(false, "error_info");
        }
    });
}
```

I changed `dataType` to `text` and did the parsing to JSON manually: `let data = $.parseJSON(text.replace(/"MTS[^"]*"/, '"MTC"'));`.

It allowed me to delete that buggy message and to see all the new messages.
