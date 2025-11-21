AccessibilityService

frameworks/base/core/java/android/accessibilityservice/
AcessibilityService.java #개발자가 상속하는 기본 클래스
AccessibilityServiceInfo.java # 서비스 메타데이터

frameworks/base/services/accessibility/java/com/android/server/accessibility/
AcessibilityManagerService.java #시스템 서비스 핵심
AccessibilityServiceConnection.java #서비스 연결 관리


## 접근성 서비스 생명주기 관리 핵심 코드

```java
public class AccessibilityManagerService extends IAccessibilityManager.Stub{
  //서비스 바인딩 부분
  private void bindAccessibilityServiceLocked(AccessibilityServiceInfo info, int userId) {
    ComponentName componentName = ComponentName.unflattenFromStrin(
      info.getId());
    Intent intent = new Intent();
    Intent.setComponent(componentName);

  //중요: BIND_AUTO_CREATE | BIND_FOREGROUND_SERVICE_WHILE_AWAKE
  long identity = Binder.clearCallingIdentity();
  try{
    if(mContext.bindServiceAsUser(intent, connection,
      Context.BIND_AUTO_CREATE
      | Context.BIND_FOREGROUND_SERVICE_WHILE_AWAKE
      | Context.BIND_ALLOW_BACKGROUND_ACTIVITY_STARTS,
      new UserHandle(userId))) {
    // binding succ
      }
    } finally {
      Binder.restoreCallingIdentity(identity);
    }
  }
}

```

BIND_AUTO_CREATE: 서비스 죽으면 자동 재시작
BIND_FOREGROUND_SERVICE_WHILE_AWAKE: 포그라운드 우선순위 부여
BIND_ALLOW_BACKGROUND_ACTIVITY_STARTS: 백그라운드에서도 작동 허용




## Process Kill 메커니즘
ActivityManagerService의 프로세스 관리

```java
public class ActivityManagerService extends IActivityManager.Stub {
  final ProcessList mProcessList;

  //OOM Adjuster - 프로세스 우선순위 결정
  final 0omAdjuster m0omAdjuster;
}

```

OomAdjuster.java - 프로세스 우선순위 계산

```java
public final class OomAdjuster{
  //접근성 서비스의 우선순위 계산
  private boolean computeOomAdjLocked(ProcessRecord app, int cachedAdj, ProcessRecord TOP_APP, boolean doingAll, long now, boolean cycleReEval, boolean computeClients){

    //서비스 바인딩 체크
    for (int is = app.services.size() - 1; is >= 0; is--) {
      ServiceRecord s = app.services.valueAt(is);

      //접근성 서비스인지 확인
      if(s.hasAccessibilityBindings()) {
        //PERCEPTIBLE_APP_ADJ 수준으로 설정 이는 포그라운드 앱 다음으로 높은 우선 순위
        app.adjType = "accessibility";
        if (adj > ProcessList.PERCEPTIBLE_APP_ADJ) {
            adj = ProcessList.PERCEPTIBLE_APP_ADJ;
        }
        schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
      }
    }
    return success;
  }
}
```

ProcessList.java - OOM ADJ 상수 정의
``` java
public final class ProcessList{
  //OOM Adjuster 레벨 정의
  static final int UNKNOWN_ADJ = 1001;
  static final int CACHED_APP_MAX_ADJ = 906;
  static final int CACHED_APP_MIN_ADJ = 900;
  static final int SERVICE_B_ADJ = 800;//일반백그라운드서비스
  static final int PREVIOUS_APP_ADJ = 700;
  static final int HOME_APP_ADJ = 600;
  static final int SERVICE_ADJ = 500;//실행중인서비스
  static final int HEAVY_WEIGHT_APP_ADJ = 400;
  static final int BACKUP_APP_ADJ = 300;
  static final int PERCEPTIBLE_LOW_APP_ADJ = 250;
  static final int PERCEPTIBLE_APP_ADJ = 200;//접근성서비스
  static final int VISIBLE_APP_ADJ = 100;//보이는앱
  static final int FOREGROUND_APP_ADJ = 0;//포그라운드앱
  static final int PERSISTENT_SERVICE_ADJ = -700;//시스템서비스
  static final int PERSISTENT_PROC_ADJ = -800;
  static final int SYSTEM_ADJ = -900;//시스템프로세스
  static final int NATIVE_ADJ = -1000;//네이티브프로세스

```



