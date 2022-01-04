---
layout: single
title : ExoPlayer 샘플 만들기 및 정리
---
### 개요
---
ExoPlayer 는 안드로이드 어플리케이션단의 미디어 플레이어다.
안드로이드 [MediaPlayer](https://developer.android.com/guide/topics/media/mediaplayer?hl=ko) 과 똑같이 구글에서 만들었지만 오픈 소스 미디어 플레이 라이브러리로 MediaPlayer 보다 장점과 기능이 많다 (내장되있지 X).   
음악,비디오 재생을 로컬이나 스트리밍단에서 동작하도록 지원하고 MediaPlayer 에서는 지원하지 않는 DASH 나 SmoothStreaming 재생 등도 지원하고 커스터 마이징하기도 쉽고 확장성도 좋다.

##### 작업 진행 순서

1. SimpleExoplayer 인스턴스 생성 준비 및 여러 미디어 리소스 재생 
2. 백그라운드, 포그라운드 재생,일시정지 등을 앱의 액티비티 생명주기에 맞게 통합
3. MediaItem 객체 사용해 playlist 만들기
4. 미디어 품질을 사용가능한 대역폭에 맞게 조정하는 적응형 비디오 스트림 재생
5. 재생 상태를 체크하기 위한 listener 등록
6. ExoPlayer 기본 UI 말고 커스텀 UI 적용해보기

### 사전작업
---

##### git
`git clone https://github.com/googlecodelabs/exoplayer-intro.git`

##### dependency
```gradle
implementation 'com.google.android.exoplayer:exoplayer-core:2.12.0'
implementation 'com.google.android.exoplayer:exoplayer-dash:2.12.0'
implementation 'com.google.android.exoplayer:exoplayer-ui:2.12.0'
```

##### build.gradle
```gradle
repositories {
    google()
    jcenter()
}
```

##### app 의 build.gradle
```gradle
android {
    compileOptions {
        sourceCompatibility 1.8
        targetCompatibility 1.8
    }
}
```

### 화면구성 및 재생
---
위의 프로젝트에서 `exoplayer-codelab-00` 모듈만 가지고 작업을 하였다.   
해당 프로젝트에서 `activity_player.xml` FrameLayout 안에 PlayerView 안에 만들어 보자.

```xml
<com.google.android.exoplayer2.ui.PlayerView
   android:id="@+id/video_view"
   android:layout_width="match_parent"
   android:layout_height="match_parent"/>
```

이제 위의 vidio_view 를 참조하기 위해 viewBinding 을 해야하므로 Activity 에 해당 부분을 선언하자

```java
private val viewBinding by lazy(LazyThreadSafetyMode.NONE) {
    ActivityPlayerBinding.inflate(layoutInflater)
}
```

이제 ExoPlayer 객체를 만들자. 가장 간단히 만드는 법은 SimpleExoPlayer.Builder 클래스를 사용하는 것이다.  
(빌더패턴을 사용한 클래스로 인스턴스를 간단하게 얻을 수 있다.)
Player 인스턴스를 필요때마다 만들어야 하므로 (생명주기에 맞게) 변수로 선언해 초기화를 하자.

```java
private var player: SimpleExoPlayer? = null
   
private fun initializePlayer() {
player = SimpleExoPlayer.Builder(this)
    .build()
    .also { exoPlayer ->
      viewBinding.videoView.player = exoPlayer
    }
}
```

플레이어도 만들었고 이제 재생할 컨텐츠를 넣어야 하는데 `[MediaItem](https://exoplayer.dev/doc/reference/com/google/android/exoplayer2/MediaItem.html)` 객체를 통해서 player 가 재생해준다.  
가장 간단히 MediaItem 을 만드는 법은 `MediaItem.fromUri` 를 사용해 만드는것이고 이제 만들었다면 해당  
MediaItem 을 `setMediaItem` 으로 set 해주자.

```java
private fun initializePlayer() {
player = SimpleExoPlayer.Builder(this)
    .build()
    .also { exoPlayer ->
        val mediaItem = MediaItem.fromUri(getString(R.string.media_url_mp3))
        viewBinding.videoView.player = exoPlayer
    }
}
```
단순히 player 를 onCreate 에서만 만들고 해제등을 안해줄 경우 메모리 리소스 누수라던가 cpu 작업 , 네트워크 접속 , 하드웨어 코덱등 리소스가 낭비 될수 있다. 특히 하드웨어 코덱은 많은양을 잡아먹는다. 즉 백그라운드같이  화면이 보이지 않을땐 리소스를 해제하고 반대로 화면이 보일때 다시 인스턴스를 만들어 사용하는 식으로 구성해야 리소스 낭비를 덜할 수 있다. 

안드로이드는 생명주기가 있고 우리는 거기에 맞게 생성 해제를 해야한다.

```java
public override fun onStart() {
 super.onStart()
 if (Util.SDK_INT >= 24) {
   initializePlayer()
 }
}

public override fun onResume() {
 super.onResume()
 hideSystemUi()
 if ((Util.SDK_INT < 24 || player == null)) {
   initializePlayer()
 }
}
```

Android ApI 24 레벨에서 multiple windows 를 지원하는데 해당 분기를 적용안할경우 분할모드에서 안보일수 있으므로 적용해줘야 한다.


```java
@SuppressLint("InlinedApi")
private fun hideSystemUi() {
 viewBinding.videoView.systemUiVisibility = (View.SYSTEM_UI_FLAG_LOW_PROFILE
     or View.SYSTEM_UI_FLAG_FULLSCREEN
     or View.SYSTEM_UI_FLAG_LAYOUT_STABLE
     or View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY
     or View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
     or View.SYSTEM_UI_FLAG_HIDE_NAVIGATION)
}
```
전체화면을 위해 `onResume` 상태에서 적용하기 위한 함수다.

```java

private var playWhenReady = true
private var currentWindow = 0
private var playbackPosition = 0L

private fun releasePlayer() {
    player?.run {
        playbackPosition = this.currentPosition
        currentWindow = this.currentWindowIndex
        playWhenReady = this.playWhenReady
        release()
    }
    player = null
}

public override fun onPause() {
 super.onPause()
 if (Util.SDK_INT < 24) {
   releasePlayer()
 }
}


public override fun onStop() {
 super.onStop()
 if (Util.SDK_INT >= 24) {
   releasePlayer()
 }
}
```

위에서 언급한것 처럼 리소스 누수를 막기위해 `onStop` 이나  `onPause` 에서 releasePlayer 함수로 리소스를 해제하고 동시에 다시 앱에 진입했을때 플레이어의 정보를 입력하기 위해 currentPosition 이라던가 ,currentWindowIndex, playWhenReady 등 정보를 변수로 담아둔다.

```java
private fun initializePlayer() {
player = SimpleExoPlayer.Builder(this)
    .build()
    .also { exoPlayer ->
        val mediaItem = MediaItem.fromUri(getString(R.string.media_url_mp3))
        playWhenReady = this@PlayerActivity.playWhenReady
        seekTo(currentWindow, playbackPosition)
        prepare()
        viewBinding.videoView.player = exoPlayer
    }
}
```
마지막으로 시작시 상태값 및 시작점 위치를 플레이어 시작시 동작하도록 작업해두고 prepare() 함수로 준비상태로 만든다.

- playWhenReady : 재생에 필요한 모든 리소르가 확보되는 즉시 재생할지 플레이어에게 알린다.  
`playWhenReady` 는 초기값이 true 이므로 앱 최초실행시 재생이 자동시작된다.
- seekTo : 플레이어에게 특정 위치를 찾도록 지정한다. 최초 앱이 실행시 `currentWindow` `playbackPosition ` 둘다 0으로 초기화 된다.
- prepare : 플레이어에게 재생에 필요한 모든 리소스를 얻었다고 알린다.

{% include image.html id="13JLgOQc2T-WQN9Nj15nbsL9bUCi07phJ" %}

위에 mediaItem 을 동영상으로 바꿔도 문제없이 재생된다

```java
val mediaItem = MediaItem.fromUri(getString(R.string.media_url_mp4))
```

### 플레이 리스트 만들기
---
MediaItem 객체에 addMediaItem 을 호출하는것 만으로도 플레이 리스트가 만들어진다. 
위의 initializePlayer 함수에 추가해보자.

```java
private void initializePlayer() {
  [...]
  exoPlayer.addMediaItem(mediaItem)
  val secondMediaItem = MediaItem.fromUri(getString(R.string.media_url_mp3));
  exoPlayer.addMediaItem(secondMediaItem);
  [...]
}
```

{% include image.html id="13avMjGFcyg8U2XevtWR88Hi03y-hY-m5" %}

위에 이미지에 보이듯 가장 끝부분에 다음 재생 , 이전재생 버튼이 활성화 되엇다. 

### 적응형 스트리밍 (Adaptive streaming)
---

[적응형 스트리밍](https://ko.wikipedia.org/wiki/%EC%A0%81%EC%9D%91_%EB%B9%84%ED%8A%B8%EB%A0%88%EC%9D%B4%ED%8A%B8_%EC%8A%A4%ED%8A%B8%EB%A6%AC%EB%B0%8D) 이란 기술은 가능한 네트워크 대역폭에 맞게 다양한 퀄리티의 미디어를 스트리밍 하는 기술이다. 예를들어 유튜브 같은경우도 인터넷 네트워크 속도가 느릴경우 480p 나 그 이하로 스트리밍을 해주다가 다시 상태가 올라가면 1080p로 자동 재생을 해주는것과 같다. 

보통 같은 컨텐츠가 여러 트랙으로 나뉘에 있고 네트워크 상태에 맞게 선택되서 재생된다. 
각 트랙은 보통 2~10초 사이의 덩어리로 나뉘어 있다. 그리고 플레이어는 빠르게 네트워크 대역폭에 변화에 맞게 변경 해준다. 

적응형 스트리밍의 핵심은 현재 환경에 맞게 가장 적절한 트랙을 선택하게 해주는 것이다. 그래서 ExoPlayer 에서는 [track selection](https://exoplayer.dev/track-selection.html) 를 제공한다. 

```java
private fun initializePlayer() {
        val trackSelector = DefaultTrackSelector(this).apply {
            setParameters(buildUponParameters().setMaxVideoSizeSd())
        }
        player = player ?: SimpleExoPlayer.Builder(this)
            .setTrackSelector(trackSelector)
            .build()
            .also { exoPlayer ->
               val mediaItem = MediaItem.Builder()
                .setUri(getString(R.string.media_url_dash))
                .setMimeType(MimeTypes.APPLICATION_MPD)
                .build()
                 exoPlayer.run {
                    addMediaItem(mediaItem)
                    playWhenReady = this@PlayerActivity.playWhenReady
                    seekTo(currentWindow, playbackPosition)
                    addListener(playbackStateListener)
                    prepare()
                    viewBinding.videoView.player = exoPlayer
                }
        }
}
```

먼저 적절한 mediaItem 을 골라줄 [DefaultTrackSelector](https://exoplayer.dev/doc/reference/com/google/android/exoplayer2/trackselection/DefaultTrackSelector.html) 를 만든다. `trackSelector` 가 
저화질 혹은 고화질로 맞춰준다.  

##### DASH
[DASH](https://ko.wikipedia.org/wiki/HTTP_%EB%8F%99%EC%A0%81_%EC%A0%81%EC%9D%91_%EC%8A%A4%ED%8A%B8%EB%A6%AC%EB%B0%8D) 는 광범위하게 사용되는 적응형 스트리밍 기술이다. DASH 컨텐츠를 스트리밍 하기 위해선 위의 방법처럼 `fromUri ` 을 통해 MediaItem 을 만들면 안되고 적절한 MIME TYPE 으로 맞춰서 만들어야 한다.  
`fromUri` 는 media 포맷 확장자를 기본적으로 사용하지만 DASH 타입은 `APPLICATION_MPD` MIME TYPE 을 사용한다.

##### MediaItem.Builder 로 만들수 있는 다양한 속성들

- 미디어 콘텐츠의 MIME TYPE 
- DRM 유형이나 라이센스 서버 URI, 라이센스 요청 헤더 등을 포함한 보호된 콘텐츠 속성
- 재생중 사용할 [사이드로드](https://zetawiki.com/wiki/%EC%82%AC%EC%9D%B4%EB%93%9C%EB%A1%9C%EB%94%A9) 된 자막파일
- 광고 태그 URI 및 광고 주입

```java
private fun initializePlayer() {
        val trackSelector = DefaultTrackSelector(this).apply {
            setParameters(buildUponParameters().setMaxVideoSizeSd())
        }
        player = player ?: SimpleExoPlayer.Builder(this)
            .setTrackSelector(trackSelector)
            .build()
            .also { exoPlayer ->
               val mediaItem = MediaItem.Builder()
                .setUri(getString(R.string.media_url_dash))
                .setMimeType(MimeTypes.APPLICATION_MPD)
                .build()
                 exoPlayer.run {
                    addMediaItem(mediaItem)
                    playWhenReady = this@PlayerActivity.playWhenReady
                    seekTo(currentWindow, playbackPosition)
                    addListener(playbackStateListener)
                    prepare()
                    viewBinding.videoView.player = exoPlayer
                }
        }
}
```

앱을 재시작 해보면 DASH 를 통한 적응형 비디오 스트리밍이 적용된것이 보일 것이다.

##### 다른 적응형 스트리밍 기술들

대표적으로 애플이 개발한 [HLS(MimeTypes.APPLICATION_M3U8)](https://ko.wikipedia.org/wiki/HTTP_%EB%9D%BC%EC%9D%B4%EB%B8%8C_%EC%8A%A4%ED%8A%B8%EB%A6%AC%EB%B0%8D) 와 마이크로소프트가 개발한 [SmoothStreaming(MimeTypes.APPLICATION_SS)](https://www.microsoft.com/silverlight/smoothstreaming/) 이 공통적으로 사용된다. 물론 두 기술다 ExoPlayer 에서 지원한다. 


### 리스너 적용
---
ExoPlayer 는 위에서 언급한것 말고도 화면뒤에서 여러작업을 한다.

- 메모리 할당
- 컨테이너 파일 다운
- 컨테이너로 부터 온 메타정보 추출
- 데이터 디코딩
- 비디오,오디오,화면을 위한 텍스트 렌더링

ExoPlayer 가 런타임에서 동작하는것을 이해하는것은 유저가 더 편하게 재생을 하기위해서 유용하다. 예를 들어 플레이어가 아직 버퍼링 중이라면 로딩 스피너를 띄운다던가 컨텐츠가 다 재생되고 나서 다음화 재생등의 화면을 띄운다던가 등. 이러한 부분은 플레이어가 제공해주는 `listerer`  를 통해 이벤트 콜백을 받을 수 있다.

```java
private val playbackStateListener: Player.EventListener = playbackStateListener()

private fun playbackStateListener() = object : Player.EventListener {
    override fun onPlaybackStateChanged(playbackState: Int) {
        val stateString: String = when (playbackState) {
            ExoPlayer.STATE_IDLE -> "ExoPlayer.STATE_IDLE      -"
            ExoPlayer.STATE_BUFFERING -> "ExoPlayer.STATE_BUFFERING -"
            ExoPlayer.STATE_READY -> "ExoPlayer.STATE_READY     -"
            ExoPlayer.STATE_ENDED -> "ExoPlayer.STATE_ENDED     -"
            else -> "UNKNOWN_STATE             -"
        }
        Log.d(TAG, "changed state to $stateString")
    }
}
```
해당 리스너를 구현함으로 재생이나 정지등의 상태에 대한 콜백을 받아 볼 수 있고 거기에 맞는 작업을 할 수 있다.

- ExoPlayer.STATE_IDLE : 플레이어가 시작됫음. 하지만 준비중
- ExoPlayer.STATE_BUFFERING : 현재 위치에서 데이터가 충분히 모이지 않아 플레이어가 재생을 할 수 없음
- ExoPlayer.STATE_READY : 현재 위치에서 바로 재생 가능. `playWhenReady` 속성값 true 일 경우 자동 재생가능.
- ExoPlayer.STATE_ENDED : 미디어 재생 완료.

그밖에 실제 재생 중일때 다음과 같은 상태를 가진다.

- 재생시 상태는 STATE_READY 고 playWhenReady 값은 true 다.
- 다른 이유로 재생이 중단되지 않는다 (오디오 포커스 손실등)
- ExoPlayer.isPlaying 로 현재 재생중인지 알수 있다. 
 

##### 리스너 등록 및 해제

```java
private void initializePlayer() {
    [...]
    exoPlayer.seekTo(currentWindow, playbackPosition)
    exoPlayer.addListener(playbackStateListener)
    [...]
}

private void releasePlayer() {
 player?.run {
   [...]
   removeListener(playbackStateListener)
   release()
 }
  player = null
}

```

##### 그밖에 리스너

[AnalyticsListener](https://exoplayer.dev/doc/reference/com/google/android/exoplayer2/analytics/AnalyticsListener.html) 를 등록하면 오디오,비디오 재생에 디테일한 정보를 관찰할 수 있다. 


### 플레이어 화면 커스텀

{% include image.html id="13b5VpUDdVrba3JqlCSNyWXONXOqiVzeN" %}

아무것도 설정안하고 사용시 위처럼 기본 재생 컨트롤러가 적용된다. 근데 만약 버튼 색상이나 기능등을 수정하고 싶다면 어떻게 해야할까? ExoPlayer 는 이러한 커스텀 부분도 제공해준다.

일단 기본 컨트롤러를 안쓸려면 `activity_player.xml` 에서 use_controller 속성을 false 로 적용한다.

```xml
<com.google.android.exoplayer2.ui.PlayerView
   [...]
   app:use_controller="false"/>
```

`PlayerControlView` 는 그밖에도 다양한 속성을 가지는데 대표적으로 show_timeout 속성을 적용하면 적용한 시간후에 컨트롤러가 유저액션이 있기 전까지 사라지게된다.

```xml
<com.google.android.exoplayer2.ui.PlayerView
   android:id="@+id/video_view"
   android:layout_width="match_parent"
   android:layout_height="match_parent"
   app:show_timeout="10000"/>
```

이제 커스텀 컨트롤러 화면을 만들어서 적용해보자. `custom_player_control_view.xml` 를 만들고 playerView 
를 해당 xml 을 참조하도록 적용하자.  
[원본 컨트롤러 리소스](https://raw.githubusercontent.com/google/ExoPlayer/release-v2/library/ui/src/main/res/layout/exo_player_control_view.xml) 에서 복사를 해서 `@id/exo_prev`,`@id/exo_next.` ImageButton 만 제거해보자.

```xml
<com.google.android.exoplayer2.ui.PlayerView  
   android:id="@+id/video_view"
   android:layout_width="match_parent"
   android:layout_height="match_parent"
   app:controller_layout_id="@layout/custom_player_control_view"/>
```

{% include image.html id="13kSJrrHeBmcQEEs7tpZXOxnznADJ2tdP" %}

이제 위처럼 이전목록 다음목록 버튼이 사라진것이 보일 것이다. 주의할 점은 `PlayerControlView` 는 
그들의 id 값들 보고 식별한다. 즉 커스터마이징 하더라도 id 값은 있는 그대로 `@id/exo_play` 나
`@id/exo_pause` 등 으로 적용해야 한다. 바꿀경우 `PlayerControlView` 가 찾지를 못한다.

이제 버튼색 등을 색상을 바꾸고 적용해보자 

```xml
custom_player_control_view.xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_gravity="bottom"
    android:layout_marginBottom="30dp"
    android:layoutDirection="ltr"
    android:background="#CC000000"
    android:orientation="vertical"
    tools:targetApi="28">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:orientation="horizontal">

        <ImageButton android:id="@id/exo_rew"
            android:tint="#FF00A6FF"
            style="@style/ExoMediaButton.Rewind"/>

        <ImageButton android:id="@id/exo_shuffle"
            style="@style/ExoMediaButton"/>

        <ImageButton android:id="@id/exo_repeat_toggle"
            style="@style/ExoMediaButton"/>

        <ImageButton android:id="@id/exo_play"
            android:tint="#FF00A6FF"
            style="@style/ExoMediaButton.Play"/>

        <ImageButton android:id="@id/exo_pause"
            android:tint="#FF00A6FF"
            style="@style/ExoMediaButton.Pause"/>

        <ImageButton android:id="@id/exo_ffwd"
            android:tint="#FF00A6FF"
            style="@style/ExoMediaButton.FastForward"/>

        <ImageButton android:id="@id/exo_vr"
            style="@style/ExoMediaButton.VR"/>

    </LinearLayout>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="4dp"
        android:gravity="center_vertical"
        android:orientation="horizontal">

        <TextView android:id="@id/exo_position"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textSize="14sp"
            android:textStyle="bold"
            android:paddingLeft="4dp"
            android:paddingRight="4dp"
            android:includeFontPadding="false"
            android:textColor="#FF00A6FF"/>

        <View android:id="@id/exo_progress_placeholder"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="26dp"/>

        <TextView android:id="@id/exo_duration"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textSize="14sp"
            android:textStyle="bold"
            android:paddingLeft="4dp"
            android:paddingRight="4dp"
            android:includeFontPadding="false"
            android:textColor="#FF00A6FF"
            />

    </LinearLayout>

</LinearLayout>
```

{% include image.html id="13r6fg8NMku5i1-KGXJoWwzEslNlP3zc-" %}

위처럼 controller_layout_id 속성을 적용해서 하는 방법도 있지만   
실제 `PlayerControllerView` 의 경우 `R.layout.exo_player_control_view` 레이아웃을 사용하므로   
직접 같은 파일명으로 만들경우 해당 파일을   참조하게 된다.


### 참조
---
[https://developer.android.com/codelabs/exoplayer-intro](https://developer.android.com/codelabs/exoplayer-intro)  
[https://exoplayer.dev/](https://exoplayer.dev/)
