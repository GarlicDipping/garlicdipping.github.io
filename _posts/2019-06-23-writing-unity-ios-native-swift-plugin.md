---
layout: post
title: Swift5로 유니티 iOS 네이티브 플러그인 만들기
date:   2019-06-23 20:00:00
author: GarlicDipping
tags:
- Programming, iOS, Unity3d
categories: Unity3d
---

현재 개발중인 게임의 개발 마무리 단계가 다가보면서, 미뤄뒀던 구현 사항들을 슬슬 하나둘씩 매듭지어야 하는 시기가 왔다.

그 중 하나가 공지사항을 띄우기 위한 용도로 이용할 웹뷰였다. 이런저런 유니티 웹뷰 플러그인들이 있지만 아마 [gree에서 공개한 gree-webview](https://github.com/gree/unity-webview)가 가장 널리 쓰이는 물건이 아닐까 싶은데, 솔직히 말하자면 단순 공지 웹페이지 Get 용도로만 사용할 플러그인으로서는 기능이 지나치게 많고 무거웠다.  

프로그래밍을 배운 뒤 처음으로 프로젝트를 제작했던 플랫폼이 안드로이드였기도 했고, 트위터 로그인 연동을 위해 얼마 전 iOS용 [Swifter](https://github.com/mattdonnelly/Swifter) 프로젝트를 유니티로 연결했던 경험도 있었다 보니 웹뷰 플러그인을 직접 만들기로 결정했다. (레이아웃 만들어서 웹뷰로 꽉 채워버리면 땡이라 제작에 그리 오래 걸리지도 않았다.)  

다만 코딩 편의성을 위해 Objective-C 대신 Swift로 플러그인을 제작했다 보니 유니티 공식 문서의 가이드라인만 보고 연동하기가 쉽지 않았었는데, 그 부분에 대해 적어보고자 한다.

---

우선은 iOS에서의 프로젝트 구조를 간단하게 짚고 넘어가자.

![Image Alt GarlicWebview Workspace Hierarchy](/assets/img/posts/20190623/garlicwebview-xcwspc-hierarchy.png)

[1] GarlicWebviewUnityBridge : 모듈 프로젝트. 유니티와 네이티브 로직의 통신을 담당한다.  
[2] GarlicWebview : 모듈 프로젝트. 코어 로직이 담겨 있다.   
[3] GarlicWebviewApp : 샘플 프로젝트. iOS Single View App이다.
  
>모듈 프로젝트는 Xcode에서 File-New-Project-Cocoa Touch Framework 형식으로 만든 프로젝트를 뜻합니다. 

GarlicWebview 프로젝트는 핵심 로직을 지닌 싱글턴 클래스, GarlicWebviewController를 포함하고 있다.  
해당 클래스는 다음과 같이 사용한다.

<pre>
<code>
//Inside ViewController class...

@IBAction func onClick(_ sender: UIButton) {
    let marginPx = GarlicUtils.PointToPx(pt: 30)
    GarlicWebviewController.shared.SetFixedRatio(width: 16, height: 9)
    GarlicWebviewController.shared.SetMargins(left: marginPx, right: marginPx, top: marginPx, bottom: marginPx)
    GarlicWebviewController.shared.Show(url: "https://www.teamtapas.com")
}
</code>
</pre>

보면 알겠지만 기본적으로 GarlicWebviewController.shared.XXX 형식으로 활용한다. 이제 이 프레임워크를 유니티 브릿지에 연결시켜 보자.  

# 유니티에서 iOS 네이티브 함수로의 호출

본격적으로 시작하기 전에, 우선 유니티에서 iOS 네이티브로 호출을 연결하는 방법을 알아보자. [공식 문서](https://docs.unity3d.com/kr/530/Manual/PluginsForIOS.html)에 적혀 있는 내용이기도 하다.  

유니티에서 네이티브로 함수 호출을 하려면 두 가지 셋업을 해야 한다. 

1. Objective-C  
  extern "C"로 함수 노출
2. Unity3D(C#)  
  [DllImport("__Internal")] 어트리뷰트를 통해 네이티브로 함수 연결

예를 들어, 유니티에서 다음 함수를 정의한 후

<pre>
<code>
//C# Code from Unity

[DllImport("__Internal")]
internal static extern void __IOS_MyFunc();
</code>
</pre>

위 함수를 호출하면, Objective-C 파일에서 extern "C"로 정의된 

<pre>
<code>
#pragma mark - C interface

extern "C" {
    ...

    void __IOS_MyFunc()
    {
        [[YourSwiftWrapperClass instance] MyFunc];
    }
    
    ...
}
</code>
</pre>

위 함수로 호출이 연결되며, 모든 로직을 Objective-C로 구현했다면 여기에서 문제 없이 코딩을 마무리할 수 있을 것이다.  

하지만 로직들이 Swift 코드로 구현되어 있다면 약간의 처리가 더 필요하다. Objective-C 코드 내에서 Swift 클래스와 함수를 호출할 수 있도록 셋업해야 한다는 의미다.

어떻게 Swift 로직을 Objective-C로 연결해 줄 수 있을까?

# Bridge Project

처음 [ProjectName] 프로젝트를 생성하면 기본 생성된 [ProjectName].h 파일이 반길 것이다. 하지만 건드릴 일은 없고, 우리가 작성할 클래스들은 따로 Classes 폴더(그룹)을 만들어 관리한다.

> 이하 본인은 GarlicWebviewUnityBridge 라는 이름으로 브릿지 프로젝트를 작성했다.

![Image Alt GarlicWebview Bridge Project Hierarchy](/assets/img/posts/20190623/garlicwebviewunitybridge-xcwspc-hierarchy.png)  
(최종적으로 작성할 파일들은 위와 같다.)

각 파일들의 역할을 간단히 정리하자면 다음과 같다.  
1. GarlicWebviewUnityWrapper.swift  
  코어 모듈의 로직을 한번 더 감싼다. 이 클래스의 코드는 Objective-C로 노출될 것이다.  
  또한, 코어 모듈에서의 콜백 Receiver 역할도 겸하고 있다. 콜백을 받으면 Objective-C 함수 UnitySendMessage()를 이용해 유니티로 올려주는 로직이 포함되어 있다.
2. GarlicWebviewWrapper.mm  
  extern "C"를 통해 유니티와 통신하는 Obj-C 로직이 구현되어 있다.
3. GarlicWebviewWrapper.h  
  GarlicWebviewUnityWrapper 스위프트 클래스를 Objective-C에서 이용할 수 있도록 헤더 파일을 임포트한다.
4. GarlicWebviewUnityBridge-Bridging-Header.h  
  스위프트 코드에서 Objective-C 함수를 이용하기 위해서는 브릿징 헤더가 필요하다.  
  GarlicWebviewUnityWrapper.swift 클래스가 UnitySendMessage라는 Obj-C 함수를 호출할 수 있도록 돕는다.

> 참고로 UnitySendMessage()는 유니티가 작성한 UnityInterface.h 헤더 파일에 선언되어 있는 Objective-C 함수이다. 이 함수를 통해 Objective-C 코드는 유니티에게 메세지를 전달할 수 있다. UnityInterface.h 파일은 유니티 엔진에서 XCode 프로젝트를 빌드하기 전까지는 프로젝트에 포함되지 않으므로, 브릿징 프로젝트에서 UnitySendMessage() 호출에 해당 함수가 존재하지 않는다는 에러가 뜨거나 UnityInterface.h 파일이 없다고 에러가 뜨는 것은 정상임을 참고하자.

## Objective-C에서 Swift 클래스 및 함수 사용하기

GarlicWebviewUnityWrapper.swift 클래스는 말 그대로 유니티에 노출할 함수들을 다시 한번 래핑한 클래스다. Obj-C 파일에서 접근할 수 있어야 하므로 관련 로직에는 모두 @objc 선언이 붙어있어야 한다.

<pre>
<code>
@objc public class GarlicWebviewUnityWrapper : NSObject, GarlicWebviewProtocol {
    ...

    @objc public func Initialize(parentUIView:UIViewController) {
        GarlicWebviewController.shared.Initialize(parentUIView: parentUIView.view!, garlicDelegate: self)
    }

    ...
</code>
</pre>

위와 같이 스위프트 클래스에 @objc 선언을 붙이면 Objective-C 코드에서도 스위프트 클래스 및 함수에 접근할 수 있다. 여기서부터는 유니티와는 별개로 Swift 클래스를 Objective-C에서 쓸 수 있도록 설정하는 부분에 대한 이해가 필요하다.

---

스위프트 클래스를 Objetive-C에서 사용하기 위해서는 @objc 선언를 붙인 뒤 한 가지 더 할 일이 있다. [ProjectName]-Swift.h 헤더 파일을 임포트하는 것이다. 현재 GarlicWebviewUnityBridge가 프로젝트 명이므로, GarlicWebviewWrapper.h 파일에서 다음과 같이 import 선언을 하면...

>#import "GarlicWebviewUnityBridge-Swift.h"

...아마도 안 될 것이다.  

![Image Alt PePe Question](/assets/img/common/pepe_question.jpg)  

## 무슨 일이 생긴 것이지?

일반적으로 App Target으로 생성된 프로젝트는 [ProjectName]-Swift.h 헤더 파일이 XCode 프로젝트에 의해 자동 생성된다. (만약 Build Setting에서 Product Module Name 필드를 수정했다면 [Product Module Name]-Swift.h 파일이 생성될 것이다.)  

그러나 현재 작성중인 브릿지 프로젝트는 Framework Target으로 만들어졌으므로 임포트시 프레임워크를 같이 정의해 주어야 한다. 형식은 다음과 같다.

> #import <ProductName/ProductModuleName-Swift.h>

* 여기서 Build Settings-Packaging-Defines Module 세팅이 Yes로 되어있는지도 반드시 체크하자.

위 형식을 따르면, 유니티 브릿징 프로젝트의 Swift 헤더는 다음과 같이 임포트해야 할 것이다.

> #import "GarlicWebviewUnityBridge/GarlicWebviewUnityBridge-Swift.h"

이제 .h와 .mm파일에서 에러 없이 GarlicWebviewUnityWrapper 스위프트 클래스에 접근할 수 있을 것이다!

> 더 자세한 자료를 보고 싶다면 [애플 공식 문서](https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/importing_swift_into_objective-c) 를 참고하자. 잘 설명되어 있다.

아직 모든 셋업이 끝난 건 아니다. Build Settings에서 SWIFT_OBJC_INTERFACE_HEADER_NAME 옵션을 추가하는 단계가 남아 있다. 하지만 이 부분은 좀 더 뒤쪽에서 다른 옵션과 한꺼번에 다루기로 하자.

## Swift에서 Objective-C 함수 사용하기

위에서 잠깐 짚고 넘어갔듯이, 유니티에서 네이티브로 호출이 내려간다면 네이티브에서 유니티로 호출을 올려주는 로직이 필요한 순간도 있다. (콜백이라던가, 콜백이라던가, 콜백이라던가......)  

이를 위해서는 UnityInterface.h 헤더 파일에 정의된 UnitySendMessage 함수를 호출하면 된다. 이 헤더 파일은 유니티 엔진에서 아무 프로젝트나 만들어 Xcode 빌드를 해 보면 자동으로 포함되어 있음을 알 수 있을 것이다.(거꾸로 말하면 유니티에서 빌드하기 전까지는 헤더 파일을 직접 사용하기에 애로사항이 꽃핀다는 의미이기도 하다...)

![Image Alt UnityInterface](/assets/img/posts/20190623/unityinterface.png)  

직접 빌드해 확인해 보면 위와 같이 UnityInterface.h 파일의 존재를 확인할 수 있다. UnitySendMessage()의 정의는 다음과 같다.

> void UnitySendMessage(const char* obj, const char* method, const char* msg);

Objective-C로 모든 네이티브 플러그인 로직을 짜고 있었다면, #import "UnityInterface.h" 후 UnitySendMessage()로 간단하게 콜백 전달이 가능하겠지만, Swift에서 UnitySendMessage()를 쓰고 싶다면 이야기가 약간 달라진다.  

아까와는 반대로, Swift에서 Objective-C 함수를 호출하기 위해서는 어떤 셋업이 필요할까?

## Bridging Header

브릿징 헤더 파일을 작성하여 프로젝트에 추가하면 스위프트에서 Obj-C 클래스와 함수를 활용할 수 있게 된다. 하지만 Xcode Project의 Build Setting에서 브릿징 헤더 파일을 추가해 주어야 한다.

![Image Alt SwiftCompiler](/assets/img/posts/20190623/xcode-swift-compiler.png)  

위에 있는 두 옵션의 의미는 다음과 같다.

- Objective-C Bridging Header (=SWIFT_OBJC_BRIDGING_HEADER)
  - 스위프트 코드가 Obj-C 코드를 인식할 수 있도록 도우는 Bridging Header 파일 위치를 추가
- Objective-C Generated Interface Header Name (=SWIFT_OBJC_INTERFACE_HEADER_NAME)
  - Obj-C 코드가 Swift 코드를 인식할 수 있도록 도우는 Interface Header 파일 위치를 추가

따라서 위 두 필드는 다음과 같이 채워져야 할 것이다.

<pre>
<code>
SWIFT_OBJC_BRIDGING_HEADER = <strong><em>/path/to/bridging-header/</em></strong>GarlicWebviewUnityBridge-Bridging-Header.h
SWIFT_OBJC_INTERFACE_HEADER_NAME = GarlicWebviewUnityBridge/GarlicWebviewUnityBridge-Swift.h
</code>
</pre>

이 사실을 기억해 두고 브릿지 프로젝트를 프레임워크와 함께 복사하여 유니티의 /Plugins/iOS/ 폴더에 복사해 붙여넣은 뒤 빌드하면 작업은 일단락될 것이다.

# iOS PostProcessing

Bridge 프로젝트를 코어 프레임워크와 함께 유니티에 복사한 후 빌드하면 위 두 필드가 비어있다. 따라서 해당 필드를 적절히 채워 주면 무사히 네이티브 프레임워크를 활용할 수 있게 될 것이다.

다만 SWIFT_OBJC_BRIDGING_HEADER 필드의 경우 프레임워크를 유니티의 어느 위치에 어떻게 임포트했느냐에 따라 위치가 달라진다. 일단 빌드한 후 Path를 확인하여 Build Setting을 설정해도 되지만, 만약 매번 유니티에서 빌드할때마다 이것을 수정하는 일이 귀찮다면 유니티의 [PostProcessBuild] 기능을 이용할 수도 있다. 유니티 2017부터 소개된 Xcode Extensions 기능을 활용하면 쉽게 이 부분을 자동화할 수 있다.
<pre>
<code>
public static void OnPostProcessBuild(BuildTarget buildTarget, string buildPath) {
    if(buildTarget == BuildTarget.iOS) {
        var projPath = buildPath + "/Unity-iPhone.xcodeproj/project.pbxproj";
        var proj = new PBXProject();
        proj.ReadFromFile(projPath);
        var targetGuid = proj.TargetGuidByName(PBXProject.GetUnityTargetName());

        // Configure build settings
        proj.AddBuildProperty(targetGuid, "SWIFT_OBJC_BRIDGING_HEADER", "<strong><em>/path/to/bridging-header/ProjectName-Bridging-Header.h</em></strong>");
        proj.AddBuildProperty(targetGuid, "SWIFT_OBJC_INTERFACE_HEADER_NAME", "<strong><em>ProductName/ProjectName-Swift.h</em></strong>");
        proj.SetBuildProperty (targetGuid, "SWIFT_VERSION", "<strong><em>your_swift_version</em></strong>");

        const string defaultLocationInProj = "<strong><em>/Plugin-path/</em></strong>Plugins/iOS/";
        const string coreFrameworkName = "<strong><em>FrameworkName</em></strong>.framework";
        string framework = Path.Combine(defaultLocationInProj, coreFrameworkName);
        string fileGuid = proj.AddFile(framework, "Frameworks/" + framework, PBXSourceTree.Sdk);
        proj.SetBuildProperty(targetGuid, "LD_RUNPATH_SEARCH_PATHS", "$(inherited) @executable_path/Frameworks");
        PBXProjectExtensions.AddFileToEmbedFrameworks(proj, targetGuid, fileGuid);
        
        proj.WriteToFile(projPath);
    }
}
</code>
</pre>

프레임워크도 매번 Embedded에 직접 추가하기에는 불편하니 AddFileToEmbedFrameworks를 이용해 자동화한다.  
위와 같이 설정하면 이제 일일히 Build Settings를 만져줘야 하는 불편함을 덜 수 있을 것이다.

>프레임워크 이름, 폴더 구조 등에 따라 Path는 제각각이므로 프레임워크가 제대로 로드되지 않는다면 유니티에서 Xcode 프로젝트 빌드 후 Frameworks 폴더와 Libraries 폴더를 점검해 보자.

<br/>
<br/>
참고자료

- https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/importing_swift_into_objective-c
- https://docs.unity3d.com/kr/current/Manual/PluginsForIOS.html
- http://seorenn.blogspot.com/2014/07/swift-objective-c.html 
- http://seorenn.blogspot.com/2014/08/objective-c-swift.html
