## js函数

#### 数组

1.数组合并

arr1.concat(arr2, arr3)

2.数组排序

arr.sort(function(a, b) {return a - b})

3.数组过滤：

```js
var colors = ["red", "blue", "grey"];
colors = colors.filter(function(item) {
   return item != "red"
});

console.log(colors);    //["blue", "grey"]
```

4.判断某元素在数组中

- arr.indexOf(x)
- arr.includes(x)

5.添加元素

- push() 结尾添加  可一次添加多个
- unshift() 头部添加  可一次添加多个
- splice() 方法向/从数组指定位置添加/删除项目，然后返回被删除的项目。

6.删除元素

- delete arr[1]
- arr.splice(1,1)  方法向/从数组指定位置添加/删除元素，并返回该元素

7.find()

用于找出第一个符合条件的数组成员。它的参数是一个回调函数，所有数组成员依次执行该回调函数，直到找出第一个返回值为`true`的成员，然后返回该成员。如果没有符合条件的成员，则返回`undefined`

8.findIndex()

`findIndex`方法的用法与`find`方法非常类似，返回第一个符合条件的数组成员的位置，如果所有成员都不符合条件，则返回`-1`。

#### 对象

1.删除对象指定属性

delete obj.a

2.







# 样式

### 样式：

style="white-space: pre-wrap;"   识别\n换行符



### js给元素加样式类

 1、覆盖原来的类

**dom.setAttribute("class","newClass")**

2、追加样式类不覆盖原来的样式

**dom.classList.add("newClass")**

同理：

3、移除样式类

**dom.classList.remove("oldClass")** 

4、检查是否含有某个类

**dom.classList.contains("oneClass")**

5、修改元素的style样式

ele.style.color = 'blue'



### 日期

# [JS获取当前日期](https://www.cnblogs.com/aaronthon/p/10640767.html)



var myDate = new Date();
myDate.getYear();        //获取当前年份(2位)
myDate.getFullYear();    //获取完整的年份(4位,1970-????)
myDate.getMonth();       //获取当前月份(0-11,0代表1月)
myDate.getDate();        //获取当前日(1-31)
myDate.getDay();         //获取当前星期X(0-6,0代表星期天)
myDate.getTime();        //获取当前时间(从1970.1.1开始的毫秒数)
myDate.getHours();       //获取当前小时数(0-23)
myDate.getMinutes();     //获取当前分钟数(0-59)
myDate.getSeconds();     //获取当前秒数(0-59)
myDate.getMilliseconds();    //获取当前毫秒数(0-999)
myDate.toLocaleDateString();     //获取当前日期
var mytime=myDate.toLocaleTimeString();     //获取当前时间
myDate.toLocaleString( );        //获取日期与时间

 

[**日期时间**](http://www.cnblogs.com/carekee/admin/file::;)脚本库方法列表

[**Date**](http://www.cnblogs.com/carekee/admin/file::;).prototype.isLeapYear 判断闰年
Date.prototype.Format 日期格式化
Date.prototype.DateAdd 日期计算
Date.prototype.DateDiff 比较日期差
Date.prototype.toString 日期转字符串
Date.prototype.toArray 日期分割为数组
Date.prototype.DatePart 取日期的部分信息
Date.prototype.MaxDayOfDate 取日期所在月的最大[**天数**](http://www.cnblogs.com/carekee/admin/file::;)
Date.prototype.WeekNumOfYear 判断日期所在年的第几周
StringToDate 字符串转日期型
IsValidDate 验证日期有效性
CheckDateTime 完整日期时间检查
daysBetween 日期天数差
