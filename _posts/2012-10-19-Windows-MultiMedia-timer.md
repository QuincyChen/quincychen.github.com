---
layout: post
title: "Windows下多媒体定时器"
---

Windows下使用多媒体定时器，初始测试，发现很多和自己开始想的不一样。我的使用方式如下:

<pre class="prettyprint lang-cpp">

#include <windows.h>
#include <mmsystem.h>
#pragma comment(lib,"winmm.lib")
//自己实现的一个微型日志系统
#include "Comm/MiniLogger.h"

class WinMMTimer
{
public:
    WinMMTimer():m_iTimerID(0), m_iTimeRes(10)
    {
        TIMECAPS tc;
        if(timeGetDevCaps(&tc, sizeof(TIMECAPS)) == TIMERR_NOERROR)
        {
            m_iTimeRes = min(max(tc.wPeriodMin, 1), tc.wPeriodMax);
            MINI_LOG(logDEBUG) << "Time resolution: " << m_iTimeRes;
            timeBeginPeriod(m_iTimeRes);
        }
    };

    virtual ~WinMMTimer(){
        stop();
        timeEndPeriod(m_iTimeRes);
    };

    bool start(unsigned int iInterval, DriveWorker* pWorker = 0){
        stop();
        if(iInterval < m_iTimeRes)
            iInterval = m_iTimeRes;
        m_iTimerID = timeSetEvent(iInterval, m_iTimeRes, &TimeProc,
                (DWORD)pWorker, TIME_PERIODIC);
        return (m_iTimerID != 0);
    };

    inline void stop(){
        if(m_iTimerID != 0){
            timeKillEvent(m_iTimerID);
            m_iTimerID = 0;
        }
    };

private:
    static void CALLBACK WinMMTimer::TimeProc(UINT /*uiID*/, UINT /*uiMsg*/
            , DWORD dwUser, DWORD /*dw1*/, DWORD /*dw2*/)
    {
   	    SetThreadAffinityMask(GetCurrentThread(), 0x00000001);

        MINI_LOG(logDEBUG) << "Timer on Process " << GetCurrentProcessorNumber();
        Sleep(5);
	};

private:
    unsigned long m_iTimerID;   ///< 计时器ID
    unsigned long m_iTimeRes; ///< 计时器精度
};

</pre>


* 每次定时器到来时，都会执行TimeProc函数，当前的定时时间为1ms；下一个定时到来时，是否会启动不同的线程来运行该函数。经过试验：无论在定时1ms期间， TimeProc函数是否执行完成，第二次执行该函数的线程都是相同的(线程ID相等)。

* 若前一个1ms内执行定时函数未完成，第二个1ms定时到来时，是否会重入执行定时函数。经过在TimeProc函数中使用休眠Sleep，或者循环执行某个操作，都不会造成新的线程被执行，从而不存在该函数重入的问题。原先使用Boost库的boost::mutex来进行互斥就完全没有必要了。


在追求Windows环境下高精度定时器，还参考了以下的文章：

* [Creating a High-Precision, High-Resolution, and Highly Reliable Timer, Utilising Minimal CPU Resources](http://www.codeguru.com/cpp/w-p/system/timers/article.php/c5759/Creating-a-HighPrecision-HighResolution-and-Highly-Reliable-Timer-Utilising-Minimal-CPU-Resources.htm)

* [Implement a Continuously Updating, High-Resolution Time Provider for Windows](http://msdn.microsoft.com/en-us/magazine/cc163996.aspx)
