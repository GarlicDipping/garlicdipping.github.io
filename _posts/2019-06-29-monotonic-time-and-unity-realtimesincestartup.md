---
layout: post
title: 모바일 환경의 Monotonic Time 분석
date:   2019-06-23 20:00:00
author: GarlicDipping
tags:
- Programming, iOS, Android, Linux, OS
categories: Programming
published: false
---

# 서론

유니티의 Time.realtimeSinceStartup은 이런저런 용도로 쓸 곳이 많다. timeScale의 영향을 받지 않기도 하고, 게임이 백그라운드로 내려간 상황에서도 시간을 카운트하므로 실시간 서버를 사용할 수 없는 환경에서 시간을 시뮬레이션하는 용도로도 이용 가능하다.  

하지만 2017.4.26f1 버전 기준으로 안드로이드에서는 치명적인 버그가 있다. 앱을 백그라운드로 내리는 부분은 괜찮은데, 앱을 띄운 상태에서 화면을 껐다가 다시 켜면 realtimeSinceStartup이 제대로 카운트되지 않는 문제다.  

도당체 무슨 로직으로 이 놈이 돌아가나 알 수만 있으면 좋겠는데, 유니티는 오픈 소스 프로젝트가 아니니 Time 클래스를 뜯어보고 싶어도 뜯어 볼 수가 없다...  

없으면 만드는 법, 앱이 백그라운드로 내려가더라도 정상적인 Elapsed Time을 받아올 수 있도록 네이티브 코드를 짜기 위해 Time에 대해 이런저런 조사를 하며 알게 된 부분들을 공유한다.

# Monotonic Time

운영 체제에서 시간은 하드웨어의 상황에 영향을 받는다. High Level단에서 시간 관련 함수(C#이라면 DateTime.UTCNow 등)에 액세스할 경우, 가끔씩 미묘하게 시간이 어긋나는 현상이 발생하기도 하는데 이는 DateTime.UTCNow 등의 시간 관련 함수들이 컴퓨터의 Real-Time Clock(RTC)을 사용하기 때문이다.  

이런 종류의 시간 관련 함수들은 NTP(Network Time Protocol)라고 불리는 프로토콜에 의해 네트워크를 통해 싱크되는데, 싱크가 수행되기 전까지 하드웨어 클록의 미묘한 오차로 인해 작게는 몇ms에서 많게는 몇초까지도 차이가 나는 것이다.(= Clock Drifting)  

>[참고자료 : Introduction to NTP](https://www.akadia.com/services/ntp_synchronize.html)

또한 High Level단의 함수들은 유저가 직접 시간대를 조작하는 부분에 있어 바로 영향을 받기에, 만약 유저가 수정한 시간의 영향을 받지 않고 프로그램 내에서 시간을 시뮬레이션하고 싶다면 DateTime.UTCNow 등의 변수를 믿기보다는 다소 다른 방법이 필요하다.  

이런저런 조사를 해 본 결과 클라이언트에서 실제 시간을 시뮬레이션기 위해서는 일반적으로 다음과 같은 방법이 많이 쓰이는 것으로 보인다. (모든 로직 실행에 서버를 통하는 케이스는 논외)

1. 서버에서 시간을 받아온 뒤 클라이언트에 저장한다.(Timezone 문제를 피하기 위해 보통 UTC Time을 이용)
2. 서버에서 시간을 받아온 순간의 하드웨어 Uptime을 같이 저장한다.
3. Elapsed Time을 기준으로 시간을 시뮬레이션한다.

> 하드웨어 Uptime은 하드웨어가 부팅된 이후 지난 시간을 의미한다.

위와 같은 방법을 쓰면 클라이언트에서 유저의 시간 조작 여부와 관계없이 실제 시간을 시뮬레이션할 수 있다. 다만 위에서 언급했듯이 하드웨어의 클럭이나 이런저런 조건에 영향을 받으므로 상황에 따라 적게는 1~2초에서 오차가 점점 쌓이기 시작하면 몇분정도 실제 시간과 차이가 날 확률이 높다. 일정 시간 간격을 두고 꾸준히 서버와 Time Sync를 맞추는 로직이 필요함에 주의하자.

하드웨어의 Uptime과 같이 지속적으로 일정하게 증가하는 시간을 Monotonic Time이라고들 한다. 구글에서 검색할때 유용하게 쓰이는 키워드였다.

## Monotonic Time in Mobile Environment

### Android

안드로이드에서 하드웨어 부트 후 지난 시간을 구하는 건 쉽다. Android API단에서 이미 해당 함수를 제공하기 때문이다.

>[Android SystemClock.elapsedRealtime()](https://developer.android.com/reference/android/os/SystemClock.html#elapsedRealtime())

Sleep 상태에서도 지나간 시간을 시뮬레이션하므로 **진짜** 현재 시간이 언제인지 알아내는데에 유용하다.  

### iOS

그러나 아쉽게도 iOS 환경에서는 하드웨어의 부팅 이후 Uptime을 구하는 API를 High Level단에서 제공하지 않는다. 시스템 콜을 한번 통해야 해당 변수를 가져올 수 있는데, 이를 먼저 살펴본 뒤 안드로이드의 SystemClock 클래스를 한번 뜯어보자.  

iOS는 iOS 10버전 이후와 이전의 구현 방법이 다소 다르다. iOS10 이후부터 clock_gettime 함수가 구현되어 내장되었는데, 그 이전 버전의 경우에는 sysctl을 통해 부트 타임을 가져온 뒤 현재 시간에서 빼는 방법으로 하드웨어 Uptime을 계산해야한다.

#### iOS >= 10 구현

~~~objc

+ (int64_t) ms_uptime {
    struct timespec uptime;
    if(0 != clock_gettime(CLOCK_MONOTONIC_RAW, &uptime)) {
        [NSException raise:@"Clock_Gettime_Error" format:@"Could not execute clock_gettime, errno: \(errno)"];
    }
    int64_t result;
    result = uptime.tv_nsec / 1000000;
    result += uptime.tv_sec * 1000;
    return result;
}

~~~

#### iOS < 10 구현

~~~objc

#import <sys/sysctl.h>

static int64_t ms_boot_timestamp() {
    struct timeval boottime;
    int mib[2] = {CTL_KERN, KERN_BOOTTIME};
    size_t size = sizeof(boottime);
    int rc = sysctl(mib, 2, &boottime, &size, NULL, 0);
    if (rc != 0) {
        return 0;
    }
    return (int64_t)boottime.tv_sec * 1000 + (int64_t)(boottime.tv_usec / 1000);
}

+ (int64_t) ms_uptime_old {
    int64_t before_now;
    int64_t after_now;
    struct timeval now;
    
    after_now = ms_boot_timestamp();
    do {
        before_now = after_now;
        gettimeofday(&now, NULL);
        after_now = ms_boot_timestamp();
    } while (after_now != before_now);
    
    return (int64_t)now.tv_sec * 1000 + (int64_t)(now.tv_usec / 1000) - before_now;
}

~~~

>[출처 : StackOverflow-Getting iOS system uptime that doesn't pause when asleep](https://stackoverflow.com/a/40497811)

### Android (in-depth)

- [안드로이드 elapsedRealtime 구현 코드](https://android.googlesource.com/platform/system/core/+/master/libutils/SystemClock.cpp#51)  

## Monotonic Time in iOS



#참고자료

- [Getting iOS system uptime, that doesn't pause when asleep](https://stackoverflow.com/questions/12488481/getting-ios-system-uptime-that-doesnt-pause-when-asleep/40497811)
- [clock_gettime alternative in Mac OS X](https://stackoverflow.com/questions/5167269/clock-gettime-alternative-in-mac-os-x)