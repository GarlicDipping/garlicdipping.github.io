---
layout: post
title: 유니티 realtimeSinceStartup 대체하기 - 모바일 환경에서의 Monotonic Time 구현 및 분석
date:   2019-07-06 20:00:00
author: GarlicDipping
tags:
- Programming, iOS, Android, Linux, OS, Kernel
categories: Programming
published: true
---

# 서론

유니티의 Time.realtimeSinceStartup은 이런저런 용도로 쓸 곳이 많다. timeScale의 영향을 받지 않기도 하고, 게임이 백그라운드로 내려간 상황에서도 시간을 카운트하므로 실시간 서버를 사용할 수 없는 환경에서 시간을 시뮬레이션하는 용도로도 이용 가능하다.  

하지만 2017.4.26f1 버전 기준으로 안드로이드에서는 치명적인 버그가 있다. 앱을 백그라운드로 내리는 부분은 괜찮은데, 앱을 띄운 상태에서 화면을 껐다가 다시 켜면 realtimeSinceStartup이 제대로 카운트되지 않는 문제다.  

도당체 무슨 로직으로 이 놈이 돌아가나 알 수만 있으면 좋겠는데, 유니티는 오픈 소스 프로젝트가 아니니 Time 클래스를 뜯어보고 싶어도 뜯어 볼 수가 없다...  

없으면 만드는 법, 앱이 백그라운드로 내려가더라도 정상적인 Elapsed Time을 받아올 수 있도록 네이티브 코드를 짜기 위해 Time에 대해 이런저런 조사를 하며 알게 된 부분들을 적어둔다.

<!--more-->

# Monotonic Time

운영 체제에서 시간은 하드웨어의 상황에 영향을 받는다. High Level단에서 시간 관련 함수(C#이라면 DateTime.UTCNow 등)에 액세스할 경우, 가끔씩 미묘하게 시간이 어긋나는 현상이 발생하기도 하는데 이는 DateTime.UTCNow 등의 시간 관련 함수들이 컴퓨터에 내장된 [Real-Time Clock(RTC)](https://ko.wikipedia.org/wiki/%EC%8B%A4%EC%8B%9C%EA%B0%84_%EC%8B%9C%EA%B3%84)을 사용하기 때문이다.  

이런 종류의 시간 관련 함수들은 NTP(Network Time Protocol)라고 불리는 프로토콜로 네트워크를 통해 싱크되는데, 싱크가 수행되기 전까지 하드웨어 클록의 미묘한 오차로 인해 가끔 작게는 몇ms에서 많게는 몇초까지도 차이가 나는 것이다.(= Clock Drifting)  

>[참고자료 : Introduction to NTP](https://www.akadia.com/services/ntp_synchronize.html)

또한 High Level단의 함수들은 유저가 직접 시간대를 조작하는 부분에 있어 바로 영향을 받기에, 만약 유저가 수정한 시간의 영향을 받지 않고 프로그램 내에서 시간을 시뮬레이션하고 싶다면 DateTime.UTCNow 등의 변수를 믿기보다는 다소 다른 방법이 필요하다.  

이런저런 조사를 해 본 결과 클라이언트에서 유저가 설정한 시간과는 별개로 *실제 시간*을 시뮬레이션하기 위해서는 일반적으로 다음과 같은 방법이 많이 쓰이는 것으로 보인다. (*모든 로직 실행에 서버를 통하는 케이스는 논외)

1. 서버에서 시간을 받아온 뒤 클라이언트에 저장한다.(Timezone 문제를 피하기 위해 보통 UTC Time을 이용)
2. 서버에서 시간을 받아온 순간의 하드웨어 Uptime을 같이 저장한다.
3. 하드웨어 Uptime을 가지고 Elapsed Time을 측정해 시간을 시뮬레이션한다.

> 하드웨어 Uptime은 하드웨어가 부팅된 이후 지난 시간을 의미한다.

위와 같은 방법을 쓰면 클라이언트에서 유저의 시간 조작 여부와 관계없이 실제 시간을 시뮬레이션할 수 있다. 하드웨어 Uptime은 개념상 유저가 시간을 변경하든, 화면이 꺼져 있든 상관없이 언제나 실제 시간에 따라 누적되기 때문이다.  

다만 위에서 언급했듯이 현실에서는 이러한 Uptime 카운터 역시도 하드웨어의 클럭이나 이런저런 조건에 영향을 받으므로, 오차가 점점 누적되기 시작하면 적게는 1~2초에서 많게는 몇 분정도 실제 시간과 차이가 날 수 있다. 따라서 일정 시간 간격을 두고 꾸준히 서버와 Time Sync를 맞추는 로직이 필요하다.

> 또한 팁을 한 가지 더 쓰자면, 네트워크에 타임 싱크용 패킷이 왔다갔다 하는 시간도 고려해야 함을 기억하자. 예를 들어 클라이언트가 11시 59분 59초에 타임 싱크 요청을 보냈고 서버에 12시 정각에 도착했다면 응답 값은 12시겠지만, Timeout을 얼마로 설정했는지에 따라 클라이언트가 이 응답 값을 받는 시간은 12시 00분 1초일수도, 12시 00분 30초일 수도 있다는 것이다. 따라서 타임 싱크용 요청은 상대적으로 짧은 Timeout 값을 두어야 한다.

하드웨어의 Uptime과 같이 지속적으로 일정하게 증가하는 시간을 Monotonic Time이라고 부른다. 

## Monotonic Time in Mobile Environment

### Android

안드로이드에서 하드웨어 부트 후 지난 시간을 구하는 건 쉽다. Android API단에서 이미 해당 함수를 제공하기 때문이다.

>[Android SystemClock.elapsedRealtime()](https://developer.android.com/reference/android/os/SystemClock.html#elapsedRealtime())

해당 함수는 Sleep 상태에서도 지나간 시간을 시뮬레이션하므로, 유저의 시간 설정값을 무시하고 *진짜* 현재 시간이 언제인지 알아내는데에 유용하게 쓰일 수 있다.

### iOS

그러나 아쉽게도 iOS 환경에서는 하드웨어의 부팅 이후 Uptime을 구하는 API를 High Level단에서 제공하지 않는다. 커널 함수를 통해야 해당 변수를 가져올 수 있다.

참고로 여기서 iOS 10버전 이후와 이전의 구현 방법이 다소 다르다. iOS10 이후부터 clock_gettime 함수가 구현되었는데, 그 이전 버전의 경우에는 [sysctl](https://ko.wikipedia.org/wiki/Sysctl)을 통해 부트 타임을 가져온 뒤 gettimeofday() 함수를 이용해 현재 시간에서 빼는 방법으로 하드웨어 Uptime을 계산해야한다.  

> sysctl은 커널의 속성을 읽고 수정하기 위해 이용하는 함수이다.

gettimeofday()는 obsolescent 함수로 지정되었으므로 사용을 피하는 게 좋겠지만 iOS 9.0에서도 Monotonic Time을 활용하고자 한다면 지금 시점으로는 유일한 방법으로 보인다. 한 1~2년만 지나도 iOS9.0 지원 자체를 할 필요가 없어질 테니 뭐...  

iOS에서 시스템 Uptime을 알아내기 위한 함수 구현에 대한 논의는 [스택오버플로우의 이 질답 (Getting iOS system uptime, that doesn't pause when asleep)](https://stackoverflow.com/questions/12488481/getting-ios-system-uptime-that-doesnt-pause-when-asleep/40497811#40497811)에 잘 정리되어 있다. 아래 적은 코드의 출처 역시 이곳임을 밝힌다.

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

위 코드에서 clock_gettime의 첫 인자로 넘길 파라미터에는 몇 가지 선택지가 있는데, iOS 환경에서 asleep 상태에서도 카운트가 돌아야 하는 경우 CLOCK_MONOTONIC 또는 CLOCK_MONOTONIC_RAW를 넘겨준다. 참고로, CLOCK_MONOTONIC은 NTP에 영향을 받으나 CLOCK_MONOTONIC_RAW는 영향을 받지 않는다. 

여기서 어떤 파라미터를 선택할지는 요구사항에 따라 다르다. 중간에 시간이 약간 튀더라도 최대한 정확한 실제 시간을 아는 것이 중요하다면 CLOCK_MONOTONIC을, 코드 프로파일링 등의 짧고 상대적인 Delta Time이 중요해 시간이 튀면 안 되는 케이스에는 CLOCK_MONOTONIC_RAW를 추천한다.

- [What is Difference between CLOCK_MONOTONIC and CLOCK_MONOTONIC_RAW?](https://stackoverflow.com/questions/14270300/what-is-the-difference-between-clock-monotonic-clock-monotonic-raw)
- [CLOCK_MONOTONIC: Avoiding problems with leap seconds](https://www.reddit.com/r/programming/comments/y3mxv/clock_monotonic_avoiding_problems_with_leap/)

> 자세한 사항을 알고 싶으면 맥OS 터미널에서 man clock_gettime 을 입력하자.

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

출처에서 나와 있듯이, 부트 타임을 받은 뒤 gettimeofday() 리턴을 받기 전 NTP싱크 또는 유저의 시간 변경이 적용되어버리면 Race Condition이 발생한다. ***sysctl을 통해 리턴받은 KERN_BOOTTIME 값은 디바이스 시간 변경에 영향을 받기 때문이다.*** 이를 막기 위해 do{}while 루프 안에서 gettimeofday() 함수 호출 전후에 시간 변경 여부를 검사하도록 구현되어 있음을 확인할 수 있다. 

### 확장 - 커널 소스 분석

#### iOS

어쨌든 요점은 Monotonic Time 로직을 위해서는 clock_gettime 커널 함수를 쓰면 된다는 것이다. clock_gettime의 구현 세부사항 자체는 OS와 커널 버전에 따라 다른데, 다행이도 Apple과 Android 모두 관련 코드를 오픈해두어 로직을 살펴볼 수 있다.  

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

우리가 관심있는 파라미터는 CLOCK_MONOTONIC 및 CLOCK_MONOTONIC_RAW 두 가지이다. CLOCK_MONOTONIC 케이스에 걸리면 _mach_boottime_usec 함수를 호출하는데, 내용물을 살펴보자.

~~~c

static int
_mach_boottime_usec(uint64_t *boottime, struct timeval *realtime)
{
    uint64_t bt1 = 0, bt2 = 0;
    int ret;
    do {
        bt1 = mach_boottime_usec();
        if (os_slowpath(bt1 == 0)) bt1 = _boottime_fallback_usec();

        atomic_thread_fence(memory_order_seq_cst);

        ret = gettimeofday(realtime, NULL);
        if (ret != 0) return ret;

        atomic_thread_fence(memory_order_seq_cst);

        bt2 = mach_boottime_usec();
        if (os_slowpath(bt2 == 0)) bt2 = _boottime_fallback_usec();
    } while (os_slowpath(bt1 != bt2));
    *boottime = bt1;
    return 0;
}

~~~

_mach_boottime_usec에서는 현재 시간을 받기 위해 gettimeofday()를 이용하는 것을 볼 수 있다. 잘 보면 iOS < 10 구현 흐름과 거의 비슷함을 알 수 있는데, mach_boottime_usec은 이름대로 디바이스 부트 시간의 timestamp 역할일 것이며, Race Condition 방지를 위해 do{} while 문으로 감싸둔 것까지 동일하다.  

CLOCK_MONOTONIC_RAW의 경우는 clock_gettime_nsec_np() 함수로 넘어간다.

~~~c

uint64_t
clock_gettime_nsec_np(clockid_t clock_id)
{
    switch(clock_id){
    case CLOCK_REALTIME: {
        struct timeval tv;
        int ret = gettimeofday(&tv, NULL);
        if (ret) return 0;
        return timeval2nsec(tv);
    }
    case CLOCK_MONOTONIC: {
        struct timeval tv;
        uint64_t boottime;
        int ret = _mach_boottime_usec(&boottime, &tv);
        if (ret) return 0;
        boottime *= NSEC_PER_USEC;
        return timeval2nsec(tv) - boottime;
    }
    case CLOCK_PROCESS_CPUTIME_ID: {
        struct rusage ru;
        int ret = getrusage(RUSAGE_SELF, &ru);
        if (ret) return 0;
        return timeval2nsec(ru.ru_utime) + timeval2nsec(ru.ru_stime);
    }
    default:
        // calls that use mach_absolute_time units fall through into a common path
        break;
    }

    // Mach Absolute Time unit-based calls
    mach_timebase_info_data_t tb_info;
    if (mach_timebase_info(&tb_info)) return 0;
    uint64_t mach_time;

    switch(clock_id){
    case CLOCK_MONOTONIC_RAW:
        mach_time = mach_continuous_time();
        break;
    case CLOCK_MONOTONIC_RAW_APPROX:
        mach_time = mach_continuous_approximate_time();
        break;
    case CLOCK_UPTIME_RAW:
        mach_time = mach_absolute_time();
        break;
    case CLOCK_UPTIME_RAW_APPROX:
        mach_time = mach_approximate_time();
        break;
    case CLOCK_THREAD_CPUTIME_ID:
        mach_time = __thread_selfusage();
        break;
    default:
        errno = EINVAL;
        return 0;
    }

    return (mach_time * tb_info.numer) / tb_info.denom;
}

~~~

로직을 간단히 정리해 보면,

1. CLOCK_REALTIME, CLOCK_MONOTONIC, CLOCK_PROCESS_CPUTIME_ID는 각 용도에 적용되는 로직으로 구현
2. 그 외에는 timebase info가 초기화되었는지 체크
3. CLOCK_MONOTONIC_RAW 케이스에는 mach_continuous_time() 함수 콜

기왕 여기까지 온 거 어디까지 넘어가나 한번 보기 위해 mach_continuous_time()의 구현 사항도 살펴봤다.

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

arm64 CPU에서는 HWCLOCK 컨트롤을 받아 어셈블리 단에서 timebase 레지스터 값을 가져온다. 허나 arm64 CPU가 아닌 경우에는 _mach_continuous_time 함수를 이용한다.

~~~c

__attribute__((visibility("hidden")))
kern_return_t
_mach_continuous_time(uint64_t* absolute_time, uint64_t* cont_time)
{
    volatile uint64_t *base_ptr = (volatile uint64_t*)_COMM_PAGE_CONT_TIMEBASE;
    volatile uint64_t read1, read2;
    volatile uint64_t absolute;

    do {
        read1 = *base_ptr;
        absolute = mach_absolute_time();
#if	defined(__arm__) || defined(__arm64__)
            /*
             * mach_absolute_time() contains an instruction barrier which will
             * prevent the speculation of read2 above this point, so we don't
             * need another barrier here.
             */
#endif
        read2 = *base_ptr;
    } while (__builtin_expect((read1 != read2), 0));

    if (absolute_time) *absolute_time = absolute;
    if (cont_time) *cont_time = absolute + read1;

    return KERN_SUCCESS;
}

~~~

복잡해 보이지만, 잘 살펴보면 여기서도 iOS < 10에서 구현되었던 로직 흐름이 그대로 존재함을 확인할 수 있다.  gettimeofday() 역할은 mach_absolute_time() 함수가, BootTime 역할은 _COMM_PAGE_CONT_TIMEBASE 변수가 수행하고 있다. 문서화가 제대로 되어있지 않으나 검색을 해 보면 mach_absolute_time 함수는 부트타임 후 Tick을 리턴하는 역할로 보인다. iOS에서 clock_gettime 로직 구현의 세부사항은 사실상 거의 다 살펴보았으나 기왕 여기까지 온 거 mach_absolute_time도 한번 까보자.

>[Apple mach_absolute_time.c](https://opensource.apple.com/source/Libc/Libc-167/mach.subproj/mach_absolute_time.c.auto.html)

~~~c

#include <stdint.h>
#include <mach/clock.h>

extern mach_port_t clock_port;

uint64_t mach_absolute_time(void) {
#if defined(__ppc__)
    __asm__ volatile("0: mftbu r3");
    __asm__ volatile("mftb r4");
    __asm__ volatile("mftbu r0");
    __asm__ volatile("cmpw r0,r3");
    __asm__ volatile("bne- 0b");
#else
    mach_timespec_t now;
    (void)clock_get_time(clock_port, &now);
    return (uint64_t)now.tv_sec * NSEC_PER_SEC + now.tv_nsec;
#endif
}

~~~

Power PC의 어셈블리 명령어가 보인다. 어셈블리 언어 부분이 복잡해 보이지만 결국 핵심은 mftbu 및 mftb 호출이다. 요약하자면 time base 레지스터에서 값을 가져와 64비트 공간에 저장 후 리턴한다. 어셈블리 프로그래머가 아니다보니 Time base 레지스터의 존재에 대해서 몰랐는데 새롭게 하나 배운 기분이다.  

그 외에는 clock_get_time 시스템 콜을 호출한다. clock_get_time 함수까지는 오픈된 소스를 찾지 못했다만 어쨌든 지금까지의 흐름으로 미루어보건데 비슷하게 하드웨어 틱을 가져와 리턴하는 함수일 것이다.

함수를 주욱 따라 내려오느라 말이 길어졌지만 요약하면 다음과 같다.

>하드웨어 부트 이후 Uptime을 clock_gettime 함수에서 요청하면, CLOCK_MONOTONIC의 경우 gettimeofday()를 활용하고, CLOCK_MONOTONIC_RAW는 Time Base 레지스터(또는 비슷한 역할을 하는 하드웨어 장치)에서 값을 받아 리턴한다.

#### Android

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

elapsedRealtimeNano 함수를 살펴보면 이제 친숙한 clock_gettime 함수가 보인다. 다만 iOS와 다른 점은 clock id로 CLOCK_BOOTTIME을 쓴다는 점인데, 안드로이드의 경우 CLOCK_MONOTONIC 관련 변수는 suspend 상태에서 카운트가 되지 않아 그렇다고 한다.(즉 여기서 유니티의 realtimeSinceStartup은 clock_gettime에서 clockid로 CLOCK_BOOTTIME 이외의 값을 넘기는 것이 아닌가 추측해 볼 수 있다...)  

안드로이드는 Linux Kernel 베이스이므로 적당한 리눅스 버전을 골라 코드를 살펴볼 수 있다. 갤럭시 S8에서 커널 버전을 보니 4.4.111이므로 해당 커널 코드를 다운받아 살펴봤다.

일반적으로 Linux Kernel에서는 kernel/time/posix-timers.c에 POSIX 규격 타이머 관련 시스템 콜이 모여있다. 여기서 clock_gettime을 살펴보기 전에, k_clock 구조체에 대해 알아야 한다.

k_clock 구조체는 clock과 관련된 각종 함수 포인터가 정의된 데이터 타입이다. 일종의 인터페이스 역할으로 보인다. posix-timers.h에 정의되어 있다.  

~~~c

struct k_clock {
    int (*clock_getres) (const clockid_t which_clock, struct timespec *tp);
    int (*clock_set) (const clockid_t which_clock,
              const struct timespec *tp);
    int (*clock_get) (const clockid_t which_clock, struct timespec * tp);
    int (*clock_adj) (const clockid_t which_clock, struct timex *tx);
    int (*timer_create) (struct k_itimer *timer);
    int (*nsleep) (const clockid_t which_clock, int flags,
               struct timespec *, struct timespec __user *);
    long (*nsleep_restart) (struct restart_block *restart_block);
    int (*timer_set) (struct k_itimer * timr, int flags,
              struct itimerspec * new_setting,
              struct itimerspec * old_setting);
    int (*timer_del) (struct k_itimer * timr);
#define TIMER_RETRY 1
    void (*timer_get) (struct k_itimer * timr,
               struct itimerspec * cur_setting);
};

~~~

우리가 관심있는 부분은 CLOCK_BOOTTIME에 대한 k_clock 정의부일 것이다. posix-timers.c에 관련 코드를 찾을 수 있다.

~~~c

struct k_clock clock_boottime = {
        .clock_getres	= posix_get_hrtimer_res,
        .clock_get	= posix_get_boottime,
        .nsleep		= common_nsleep,
        .nsleep_restart	= hrtimer_nanosleep_restart,
        .timer_create	= common_timer_create,
        .timer_set	= common_timer_set,
        .timer_get	= common_timer_get,
        .timer_del	= common_timer_del,
    };

~~~

이제 다시 posix-timers.c의 clock_gettime 시스템 콜 구현으로 돌아와 보자.

~~~c

SYSCALL_DEFINE2(clock_gettime, const clockid_t, which_clock,
        struct timespec __user *,tp)
{
    struct k_clock *kc = clockid_to_kclock(which_clock);
    struct timespec kernel_tp;
    int error;

    if (!kc)
        return -EINVAL;

    error = kc->clock_get(which_clock, &kernel_tp);

    if (!error && copy_to_user(tp, &kernel_tp, sizeof (kernel_tp)))
        error = -EFAULT;

    return error;
}

~~~

clockid_to_kclock는 clock_gettime과 함께 전달되는 clockid(= CLOCK_MONOTONIC, CLOCK_REALTIME 등등)에 따라 사용할 k_clock을 리턴받는다. k_clock의 clock_get 함수 포인터를 사용하므로, posix_get_boottime를 따라가면 구현부를 찾을 수 있을 것이다.

~~~c

static int posix_get_boottime(const clockid_t which_clock, struct timespec *tp)
{
    get_monotonic_boottime(tp);
    return 0;
}

~~~

get_monotonic_boottime은 timekeeping.h에 inline 함수로 정의되어 있다. 

~~~c

/*
 * Timespec interfaces utilizing the ktime based ones
 */
static inline void get_monotonic_boottime(struct timespec *ts)
{
    *ts = ktime_to_timespec(ktime_get_boottime());
}

...

/**
 * ktime_get_boottime - Returns monotonic time since boot in ktime_t format
 *
 * This is similar to CLOCK_MONTONIC/ktime_get, but also includes the
 * time spent in suspend.
 */
static inline ktime_t ktime_get_boottime(void)
{
    return ktime_get_with_offset(TK_OFFS_BOOT);
}

...

//timekeeping.c

ktime_t ktime_get_with_offset(enum tk_offsets offs)
{
    struct timekeeper *tk = &tk_core.timekeeper;
    unsigned int seq;
    ktime_t base, *offset = offsets[offs];
    s64 nsecs;

    WARN_ON(timekeeping_suspended);

    do {
        seq = read_seqcount_begin(&tk_core.seq);
        base = ktime_add(tk->tkr_mono.base, *offset);
        nsecs = timekeeping_get_ns(&tk->tkr_mono);

    } while (read_seqcount_retry(&tk_core.seq, seq));

    return ktime_add_ns(base, nsecs);

}
EXPORT_SYMBOL_GPL(ktime_get_with_offset);

~~~

iOS 관련 파트를 읽었다면 익숙한 do{} while 문이 보인다. timekeeper는 하드웨어 틱에 따라 지속적으로 시간을 추적하는데, 이 때 timekeeper의 데이터 Version에 따른 유효성 확인이 필요하다.(일종의 mutex, 또는 락의 개념과 동일하다. NTP 싱크 또는 유저 조작에 의해 언제든 time 관련 값은 변경될 수 있음을 잊지 말자!) 따라서 이를 확인하기 위한 seqcount_t 타입의 시퀀스 카운터 데이터를 함께 묶은 tk_core 구조체가 timekeeping.c에 정의되어 있다.  

다시 코드로 돌아오면, 결국 핵심 라인은 ```base = ktime_add(tk->tkr_mono.base, *offset);``` 코드이다. tkr_mono는 monolitic timeskeeping readout 값을 의미하며, offset은 monotonic clock을 boottime으로 변환하기 위한 오프셋 값이다.  

timeskeeper 구조체 주석에 의하면 CLOCK_MONOTONIC의 경우 tkr_mono를, CLOCK_MONOTONIC_RAW의 경우 tkr_raw를 이용한다고 하니 결국 CLOCK_BOOTTIME은 NTP 싱크에 영향을 받는다는 사실도 알 수 있다.

## 요약

- 유니티 엔진의 Time.realtimeSinceStartup은 불안정하다. 특히 안드로이드에서 앱이 포커싱된 상태로 화면을 껐다가 켜면 카운트가 돌지 않는다. 에디터에서 활용하거나 간단한 통계 용도 이외에는 되도록 쓰지 말자.
- 안드로이드는 SystemClock.elapsedRealtime() 함수를 통해 화면이 꺼진 상태에서도 디바이스 부팅 이후 지난 시간을 카운트할 수 있다. 공식 문서에서도 권장하는 함수이므로 잘 활용하자.
- iOS는 부팅 이후 시간을 카운트하기 위해서는 커널 함수를 호출해야 한다. ios >= 10 환경에서는 clock_gettime(CLOCK_MONOTONIC) 또는 clock_gettime(CLOCK_MONOTONIC_RAW) 등을 이용해 값을 얻을 수 있다.
  - 단, 편의상 '부팅 이후 시간'이라고 했지만 공식 문서에서는 arbitary point 이후의 값을 리턴한다고 되어있음에 주의하자. 상대 시간이 필요하다면 별 문제는 아니다.

호기심에 어디까지 내려가볼 수 있나 한번 살펴봤는데 커널 코드까지 다운받아서 뒤져보게 될 줄은 몰랐다... 개인 시간이 약간 남아서 가능했지 아니었으면 적당히 중간에 그만뒀을듯. 뭐 그래도 간만에 학교 과제 하는 느낌 나서 재미있었다.  

생각했던 이상으로 컴퓨터의 시간과 관련된 로직은 복잡하다. 인생을 쉽게 살 수 있도록 뒤에서 열심히 일하신 커널 개발자들에게 항상 감사하자.

<br/>


# 참고자료

- [Getting iOS system uptime, that doesn't pause when asleep](https://stackoverflow.com/questions/12488481/getting-ios-system-uptime-that-doesnt-pause-when-asleep/40497811)
- [clock_gettime alternative in Mac OS X](https://stackoverflow.com/questions/5167269/clock-gettime-alternative-in-mac-os-x)
- [Apple opensource portal](https://opensource.apple.com/)
- [bootlin](https://elixir.bootlin.com/)
