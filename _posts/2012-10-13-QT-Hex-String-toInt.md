---
layout: post
title: "QT 16进制字符串转换为数字的问题"
---

QT字符串转换为整型数，当该16进制字符串表示负数，或者最高位为1时，toInt函数就会转换失败，从而结果为0。这个问题在API使用者来说，是BUG；但是QT开发团队好像不这么认为。具体在这个网页: 
[Converting a negative integer to a hexadecimal QString, and then back to an integer again doesn't work](https://bugreports.qt-project.org/browse/QTBUG-1098)

下面是我写的一段测试程序，同时也给出了避开错误的一种方法。

<pre class="prettyprint lang-cpp">
 
 bool ok = false;
 //非负数转换没有问题
 int iValue = tr("FFFC7B5").toInt(&ok, 16);
 logger()->debug(tr("FFFC7B5 toInt [%1] value: %2")
       .arg(ok?"OK":"NK").arg(iValue));
    //负数出错，结果为0
 iValue = tr("FFFFC7B5").toInt(&ok, 16);
 logger()->debug(tr("FFFFC7B5 toInt [%1] value: %2")
         .arg(ok?"OK":"NK").arg(iValue));

 //以下为正确的输出
 QString strHex = tr("FFFFC7B5");
 iValue = strHex.toUInt(&ok, 16);
 logger()->debug(tr("%1 toInt [%2] value: %3")
         .arg(strHex).arg(ok?"OK":"NK").arg(iValue));
 long lValue = strHex.toULong(&ok, 16);
 logger()->debug(tr("%1 toInt [%2] value: %3")
         .arg(strHex).arg(ok?"OK":"NK").arg(lValue));

</pre>

这里使用了[log4qt](http://log4qt.sourceforge.net/)库来输出日志。


