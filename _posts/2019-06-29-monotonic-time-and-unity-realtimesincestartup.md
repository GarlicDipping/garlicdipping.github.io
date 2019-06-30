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

그러나 아쉽게도 iOS 환경에서는 하드웨어의 부팅 이후 Uptime을 구하는 API를 High Level단에서 제공하지 않는다. 커널 함수를 통해야 해당 변수를 가져올 수 있는데, 우선 이를 먼저 살펴본 뒤 안드로이드의 SystemClock 클래스를 한번 뜯어보자.  

iOS는 iOS 10버전 이후와 이전의 구현 방법이 다소 다르다. iOS10 이후부터 clock_gettime 함수가 커널에 구현되어 내장되었는데, 그 이전 버전의 경우에는 sysctl을 통해 부트 타임을 가져온 뒤 gettimeofday() 함수를 이용해 현재 시간에서 빼는 방법으로 하드웨어 Uptime을 계산해야한다. gettimeofday()는 obsolescent 함수로 지정되었으므로 사용을 피하는 게 좋겠지만 iOS 9.0에서도 Monotonic Time을 활용하고자 한다면 지금 시점으로는 유일한 방법으로 보인다. 한 1~2년만 지나도 iOS9.0 지원 자체를 할 필요가 없어질 테니 뭐...  

어쨌든, 이하는 스택오버플로우에서 찾은 코드이다. 출처는 하단에 있다.  

#### iOS >= 10 구현

~~~objc

+ (int64_t) ms_uptime {
    struct timespec uptime;
    if(0 != clock_gettime(CLOCK_MONOTONIC_RAW, &uptime)) {
        [NSException raise:@"Clock_Gettime_Error" format:@"Could not execute clock_gettime, errno: \(errno)"];
    }
    int64_t result;
    result = uptime.tv_nsec / 1000000;
    result += (int64_t)uptime.tv_sec * 1000;
    return result;
}

~~~

clock_gettime의 첫 인자로 CLOCK_MONOTONIC_RAW를 넘겨준다. MONOTONIC에는 CLOCK_MONOTONIC과 CLOCK_MONOTONIC_RAW 두 종류가 있는데, CLOCK_MONOTONIC은 NTP에 영향을 받으나 CLOCK_MONOTONIC_RAW는 영향을 받지 않는다. 시간이 앞뒤로 널뛰기하는걸 원하지 않으니 CLOCK_MONOTONIC_RAW를 쓴 것으로 보인다.

> [What is Difference between CLOCK_MONOTONIC and CLOCK_MONOTONIC_RAW?](https://stackoverflow.com/questions/14270300/what-is-the-difference-between-clock-monotonic-clock-monotonic-raw)

timespec 구조체를 받아 tv_nsec과 tv_sec을 millisecond로 변환하여 리턴하도록 구현을 조금 수정해 봤다.
또한 변환 과정에서 오버플로우가 일어나지 않도록 int64_t 캐스팅을 하는 것도 잊지 말자.

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

+ (int64_t) ms_uptime {
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

>[출처 : StackOverflow - Getting iOS system uptime that doesn't pause when asleep](https://stackoverflow.com/a/40497811)

출처에서 나와 있듯이, 구버전용 코드에서는 커널 변수를 바로 리턴받아버리면 NTP 싱크 또는 유저가 직접 시간을 변경하는 케이스에 Race Condition이 발생할 수 있다. 이를 막기 위해 do{}while 루프 안에서 시간 변경을 검사하도록 구현했다는듯.  

### 확장 - iOS

어쨌든 요점은 Monotonic Time 로직을 위해서는 clock_gettime 커널 함수가 핵심이라는 사실을 확인할 수 있다. clock_gettime의 구현 세부사항 자체는 OS와 커널 버전에 따라 다른데, 다행이도 Apple과 Android 모두 관련 코드를 오픈해두어 로직을 살펴볼 수 있다.  

> 여기서부터는 개인적인 호기심에 Low-Level로 내려가 볼 수 있는만큼 내려가 봤습니다. Monotonic Time과 관련된 로직 자체는 위에서 모두 다루었으니 이해가 어려우면 넘기셔도 됩니다. 저도 어셈블리나 커널을 자세히 아는 건 아니라 적당히 제가 이해 가능한 범위에서 끊었으니 양해를...^^;

clock_gettime 관련 로직은 구글 검색으로 간단히 애플의 오픈소스 포탈에서 관련 소스를 찾을 수 있었다.

> [opensource.apple.com - clock_gettime.c](https://opensource.apple.com/source/Libc/Libc-1272.200.26/gen/clock_gettime.c.auto.html)

clock_gettime 코드를 한번 뒤져보자.  

~~~c

int
clock_gettime(clockid_t clk_id, struct timespec *tp)
{
    switch(clk_id){
    case CLOCK_REALTIME: {
        struct timeval tv;
        int ret = gettimeofday(&tv, NULL);
        TIMEVAL_TO_TIMESPEC(&tv, tp);
        return ret;
    }
    case CLOCK_MONOTONIC: {
        struct timeval tv;
        uint64_t boottime_usec;
        int ret = _mach_boottime_usec(&boottime_usec, &tv);
        struct timeval boottime = {
            .tv_sec = boottime_usec / USEC_PER_SEC,
            .tv_usec = boottime_usec % USEC_PER_SEC
        };
        timersub(&tv, &boottime, &tv);
        TIMEVAL_TO_TIMESPEC(&tv, tp);
        return ret;
    }
    case CLOCK_PROCESS_CPUTIME_ID: {
        struct rusage ru;
        int ret = getrusage(RUSAGE_SELF, &ru);
        timeradd(&ru.ru_utime, &ru.ru_stime, &ru.ru_utime);
        TIMEVAL_TO_TIMESPEC(&ru.ru_utime, tp);
        return ret;
    }
    case CLOCK_MONOTONIC_RAW:
    case CLOCK_MONOTONIC_RAW_APPROX:
    case CLOCK_UPTIME_RAW:
    case CLOCK_UPTIME_RAW_APPROX:
    case CLOCK_THREAD_CPUTIME_ID: {
        uint64_t ns = clock_gettime_nsec_np(clk_id);
        if (!ns) return -1;

        tp->tv_sec = ns/NSEC_PER_SEC;
        tp->tv_nsec = ns % NSEC_PER_SEC;
        return 0;
    }
    default:
        errno = EINVAL;
        return -1;
    }
}

~~~

흥미롭게도 CLOCK_MONOTONIC은 _mach_boottime_usec 함수를 호출하며 이 함수 내에서는 gettimeofday()를 이용하는 것을 볼 수 있다. 심지어 Race Condition 방지를 위해 do{} while 문으로 감싸둔 것까지, 위에서 iOS < 10에서 구현한 로직과 거의 비슷함을 확인할 수 있다. (해당 로직도 NTP의 영향을 받는다.)

CLOCK_MONOTONIC_RAW의 경우는 다음 clock_gettime_nsec_np() 콜에서 mach_continuous_time() 콜로 내려감을 확인할 수 있다. mach_continuous_time() 함수의 코드를 한번 보자.

> [Apple mach_continuous_time.c](https://opensource.apple.com/source/xnu/xnu-4570.1.46/libsyscall/wrappers/mach_continuous_time.c.auto.html)

~~~c

uint64_t
mach_continuous_time(void)
{
	uint64_t cont_time;
	if (_mach_continuous_hwclock(&cont_time) != KERN_SUCCESS)
		_mach_continuous_time(NULL, &cont_time);
	return cont_time;
}

__attribute__((visibility("hidden")))
kern_return_t
_mach_continuous_hwclock(uint64_t *cont_time __unused)
{
#if defined(__arm64__)
	uint8_t cont_hwclock = *((uint8_t*)_COMM_PAGE_CONT_HWCLOCK);
	uint64_t timebase;
	if (cont_hwclock) {
		__asm__ volatile("isb\n" "mrs %0, CNTPCT_EL0" : "=r"(timebase));
		*cont_time = timebase;
		return KERN_SUCCESS;
	}
#endif
	return KERN_NOT_SUPPORTED;
}

~~~

arm64 CPU에서는 HWCLOCK 컨트롤을 받아 어셈블리 단에서 timebase 레지스터 값을 가져온다. 허나 arm64 CPU가 아닌 경우에는 mach_absolute_time 함수를 이용한다.

>[Apple mach_absolute_time.c](https://opensource.apple.com/source/Libc/Libc-167/mach.subproj/mach_absolute_time.c.auto.html)

Power PC의 어셈블리라면 time base 레지스터를 가져오고 그 외에는 clock_get_time 시스템 콜을 호출한다. 어셈블리 프로그래머가 아니다보니 Time base 레지스터의 존재에 대해서 몰랐는데 새롭게 하나 배운 기분이다.  

clock_get_time 함수까지는 오픈된 소스를 찾지 못했다만 어쨌든 요점은 비슷할 것이다.

>하드웨어 부트 이후 Uptime을 clock_gettime 함수에서 요청하면, Time Base 레지스터에서 값을 받아 리턴한다.

### 확장 - Android

안드로이드 역시 소스 코드가 오픈되어 있으며 [SystemClock.java](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/os/SystemClock.java) 파일을 살펴보면 elapsedRealtime() 함수가 native로 연결되는 을 볼 수 있다.  

[SystemClock.cpp](https://android.googlesource.com/platform/system/core/+/master/libutils/SystemClock.cpp#51) 파일의 내용물을 살펴보면 다음과 같다.

~~~cpp

/*
 * native public static long elapsedRealtime();
 */
int64_t elapsedRealtime()
{
	return nanoseconds_to_milliseconds(elapsedRealtimeNano());
}
/*
 * native public static long elapsedRealtimeNano();
 */
int64_t elapsedRealtimeNano()
{
#if defined(__linux__)
    struct timespec ts;
    int err = clock_gettime(CLOCK_BOOTTIME, &ts);
    if (CC_UNLIKELY(err)) {
        // This should never happen, but just in case ...
        ALOGE("clock_gettime(CLOCK_BOOTTIME) failed: %s", strerror(errno));
        return 0;
    }
    return seconds_to_nanoseconds(ts.tv_sec) + ts.tv_nsec;
#else
    return systemTime(SYSTEM_TIME_MONOTONIC);
#endif
}

~~~

elapsedRealtimeNano 함수를 살펴보면 이제 친숙한 clock_gettime 함수가 보인다. 다만 iOS와 다른 점은 clock id로 CLOCK_BOOTTIME을 쓴다는 점인데, 안드로이드의 경우 CLOCK_MONOTONIC 관련 변수는 suspend 상태에서 카운트가 되지 않아 그렇다고 한다.  

> 참고로 iOS의 경우는 터미널에서 man clock_gettime 을 입력하여 확인해 보면, CLOCK_MONOTONIC 관련 변수는 asleep 상태에서도 카운트가 된다고 적혀 있다.

<br/>


# 참고자료

- [Getting iOS system uptime, that doesn't pause when asleep](https://stackoverflow.com/questions/12488481/getting-ios-system-uptime-that-doesnt-pause-when-asleep/40497811)
- [clock_gettime alternative in Mac OS X](https://stackoverflow.com/questions/5167269/clock-gettime-alternative-in-mac-os-x)
- [Apple clock_gettime.c](https://opensource.apple.com/source/Libc/Libc-1158.1.2/gen/clock_gettime.c.auto.html)
