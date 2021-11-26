---
layout : single
title : BindingAdapter 정리
---

BindingAdapter 는 적절한 프레임워크를 호출해 값을 설정한다. 
setText() 메서드를 호출하는 것과 같이 속성값을 설정하는 작업을 예로 들수 있다.

### 자동 메서드
---
```java
@RemoteView
public class TextView extends View implements OnPreDrawListener {

 public final void setText(CharSequence text) {
        throw new RuntimeException("Stub!");
    }

    public void setText(CharSequence text, TextView.BufferType type) {
        throw new RuntimeException("Stub!");
    }

    public final void setText(char[] text, int start, int len) {
        throw new RuntimeException("Stub!");
    }

    public final void setText(int resid) {
        throw new RuntimeException("Stub!");
    }

    public final void setText(int resid, TextView.BufferType type) {
        throw new RuntimeException("Stub!");
    }

    ...

}
```

```xml
 <TextView
        android:text="@string/app_name"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"/>
```

TextView 클래스의 setText 메소드만 모아온것이다. 만약 XML 클래스에서 속성중 text 속성을 지정할경우 기본적으로 위의 setText 메소드를 추적해서 그중 인자값이 일치하는 경우를 찾는다. 위의 예시에선 
Int 타입을 값으로 넣엇기 때문에 int 를 인자로 받은 setText 메소드가 실행된다.
`android:text="@string/app_name" -> setText(int resid)`

이름이 example 인 속성의 경우 setExample(arg) 메서드를 찾게되는데 여기서 네임스페이스는 고려하지 않고
속성 이름과 유형만으로 검색을 한다. 이름 속성이 없어도 setter 메소드가 등록되있다면 해당 메소드를 추적해서
사용한다. 하지만 setter 마저 등록되있지 않는경우 컴파일시 build 에러가 발생한다.

> AAPT: error: attribute example (aka kuma.test.twowaydatabindingsample:example) not found.


### 맞춤 메서드 이름 제공
---
일부 속성중에는 이름이 일치하지 않는 setter 가 있다. 이럴 경우 BindingMethods 주석을 사용해 setter 와 연결할 수 있다.

```java
    @BindingMethods(value = [
        BindingMethod(
            type = android.widget.ImageView::class,
            attribute = "android:tint",
            method = "setImageTintList")])

```

### 맞춤 로직 제공
---
`android:paddingLeft` 속성에는 연결된 setter 가 없다 . 대신 실제로 내부로직상의 setPadding (int left, int top, int right, int bottom) 메소드로 연결된다.

```java
public class View implements Callback, android.view.KeyEvent.Callback, AccessibilityEventSource {
    ...
  public void setPadding(int left, int top, int right, int bottom) {
        throw new RuntimeException("Stub!");
    }
    ...
}
```

이럴 경우 BindingAdapter 주석이 있는 BindingAdapter 메서드 사용시 속성의 setter 가 호출되는 방식을 
맞춤 설정을 할 수 있다.

```java
  @BindingAdapter("android:paddingLeft")
    fun setPaddingLeft(view: View, padding: Int) {
        view.setPadding(padding,
                    view.getPaddingTop(),
                    view.getPaddingRight(),
                    view.getPaddingBottom())
    }
```

```xml
    <TextView
        android:paddingLeft="10dp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"/>
```

이렇게 설정해두면 원래 실제 setPadding (int left, int top, int right, int bottom) 메소드를 타는게 아니라 자신이 지정한 `@BindingAdapter` 부분을 타게 된다. 단 BindingAdapter 를 선언할 경우 매개변수 유형은
중요하다. 첫번째 매개변수는 속성과 연결된 뷰의 유형을 결정하므로 xml 에 맞는 매개변수를 선언해야한다.
그리고 두번째 매개변수의 경우 지정된 속성의 유형을 뜻한다. 만약 BindingAdapter 쪽에서 충돌이 일어날 경우
Android 프레임워크에서 제공되는 기본 어댑터보다 먼저 적용된다.

그 밖에도 여러 매개변수를 받고 싶다면 속성을 여러개 추가해서 받을수도 있다.

```java
@BindingAdapter("imageUrl", "error")
    fun loadImage(view: ImageView, url: String, error: Drawable) {
        Picasso.get().load(url).error(error).into(view)
}
```

```xml
<ImageView
        app:imageUrl="@{imageUrl}"
        app:error="@{@drawable/ic_launcher_background}"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        android:minHeight="40dp"
        android:minWidth="40dp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />
/>
```

이렇게 설정해 두면 imageUrl 주소값과 Drawable 값을 xml 을 통해 받아서 
Picasso 이미지를 처리하는 로직도 구성할수 있다. 리소스를 `@{}` 로 묶으면 결합 표현식이 된다.
만약 받는 매개변수중에 모두 받을 필요가 없다면 requireAll 플래그를 추가하면 된다.


```java
 @BindingAdapter(value = ["imageUrl", "placeholder"], requireAll = false)
    fun setImageUrl(imageView: ImageView, url: String?, placeHolder: Drawable?) {
        if (url == null) {
            imageView.setImageDrawable(placeholder);
        } else {
            MyImageLoader.loadInto(imageView, url, placeholder);
        }
    }
```

만약 이렇게 설정해둿는데 xml 쪽에서 값을 넣지 않을경우 컴파일에서 에러가 발생한다.
(기본값은 require = true 로 설정되있다.) 

```java
  @BindingAdapter("android:paddingLeft")
    fun setPaddingLeft(view: View, oldPadding: Int, newPadding: Int) {
        if (oldPadding != newPadding) {
            view.setPadding(padding,
                        view.getPaddingTop(),
                        view.getPaddingRight(),
                        view.getPaddingBottom())
        }
}
```

BindingAdapter 는 선택적으로 이전값도 사용이 가능하다 매개변수 선언시 위처럼 2개를 선언해서 비교도 가능하다. 단 이전값을 먼저 선언하고 다음 새값을 선언해야한다. 

```java
@BindingAdapter("android:onClickTest")
fun setOnLayoutChangeListener(
    view: View,
    oldValue: View.OnClickListener?,
    newValue: View.OnClickListener?
) {
        if (oldValue != newValue) {
            view.setOnClickListener(newValue)
        }
}
```

```xml
<View
            android:background="@color/black"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintRight_toRightOf="parent"
            app:layout_constraintLeft_toLeftOf="parent"
            android:layout_height="30dp"
            android:layout_width="30dp"
            android:onClickTest="@{() -> mainViewModel.onClick()}"
            />
```

매개변수로 이벤트 핸들러도 받을 수 있는데 하나의 추상 메서드가 있는 인터페이스 또는 추상 클래스에서만 사용할 수 있습니다. 만약 추상 메서드나 인터페이스 메서드가 두개이상이거나 할경우는 라이브러리는 2개의 인터페이스를 제공하여 이러한 메서드의 속성 및 핸들러를 구별합니다.

```java

  public interface OnAttachStateChangeListener {
        void onViewAttachedToWindow(View var1);

        void onViewDetachedFromWindow(View var1);
    }

    @TargetApi(Build.VERSION_CODES.HONEYCOMB_MR1)
    interface OnViewDetachedFromWindow {
        fun onViewDetachedFromWindow(v: View)
    }

    @TargetApi(Build.VERSION_CODES.HONEYCOMB_MR1)
    interface OnViewAttachedToWindow {
        fun onViewAttachedToWindow(v: View)
    }
    
```

위처럼 하나에 2개가 들어있는 인터페이스를 밑에처럼 2개로 나눴다. 그리고 각각을 속성으로 받아서 
처리하면 된다.

### 객체 변환
---
결합 표현식에서 `Object` 가 반환시 라이브러리는 속성 값을  설정하는데 사용되는 메서드를 선택한다.
즉 `Object` 가 선택한 메서드의 매개변수 유형으로 변환이 이루어진다. 

### 맞춤 변환
---

```java
<View
       android:background="@{isError ? @color/red : @color/white}"
       android:layout_width="wrap_content"
       android:layout_height="wrap_content"/>
    
```

특정 유형 간의 변환도 쉽게 할 수 있는데 View 의 android:backround 속성에 Drawable 이 필요하지만 
값 자체는 color 값이 정수로 들어오게 된다. 이럴 경우 `BindingConversion` 를 사용해서 쉽게 변환 할수 있다.

```java
@BindingConversion
    fun convertColorToDrawable(color: Int) = ColorDrawable(color)
```

단 지정하는 값 유형은 일관되야 한다. 
`android:background="@{isError ? @drawable/error : @color/white}"` 이런식으로 처리할 경우 오류가 발생한다.









