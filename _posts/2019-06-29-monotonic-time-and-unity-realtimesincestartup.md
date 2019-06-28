---
layout: post
title: 유니티 Time.realtimeSinceStartup의 맹점과 Monotonic Time
date:   2019-06-23 20:00:00
author: GarlicDipping
tags:
- Programming, iOS, Unity3d
categories: Unity3d
published: false
---

# 서론

유니티의 Time.realtimeSinceStartup은 이런저런 용도로 쓸 곳이 많다. timeScale의 영향을 받지 않기도 하고, 게임이 백그라운드로 내려간 상황에서도 시간을 카운트하므로 실시간 서버를 사용할 수 없는 환경에서 시간을 시뮬레이션하는 용도로도 이용 가능하다.  

하지만 2017.4.26f1 버전 기준으로 안드로이드에서는 치명적인 버그가 있다. 앱을 백그라운드로 내리는 부분은 괜찮은데, 앱을 띄운 상태에서 화면을 껐다가 다시 켜면 realtimeSinceStartup이 제대로 카운트되지 않는 문제다.  

도당체 무슨 로직으로 이 놈이 돌아가나 알 수만 있으면 좋겠는데, 유니티는 오픈 소스 프로젝트가 아니니 Time 클래스를 뜯어보고 싶어도 뜯어 볼 수가 없다. 그럼 어떡하겠나. 없으면 만드는 수밖에.  

## 네이티브에서 받을 수 있는 시간 관련 함수



###참고자료

- [Getting iOS system uptime, that doesn't pause when asleep](https://stackoverflow.com/questions/12488481/getting-ios-system-uptime-that-doesnt-pause-when-asleep/40497811)
- [clock_gettime alternative in Mac OS X](https://stackoverflow.com/questions/5167269/clock-gettime-alternative-in-mac-os-x)