# AutoSelectScroll_UE5

Unreal Engine의 `UScrollBox`와 `SScrollBox`를 확장한 **자동 선택·중앙 정렬 스크롤 위젯**입니다. 사용자가 스크롤을 놓으면 중앙에 가까운 항목을 선택하고, 해당 항목이 중앙에 오도록 이동합니다. 중앙과의 거리를 UMG 애니메이션 재생 위치에 연결해 스크롤 중에도 항목의 크기나 강조 연출이 변하도록 구성했습니다.

프로젝트 설정의 엔진 버전은 **Unreal Engine 5.4**입니다. C++에서 입력·좌표·선택 처리를 담당하고, 항목의 시각적 연출은 UMG 애니메이션으로 구성합니다.

![자동 선택 스크롤 위젯 동작](Animation.gif)

## 주요 동작

- **자동 선택과 중앙 정렬**: 스크롤 영역의 중앙과 항목 중심 사이의 거리를 비교해 선택 대상을 찾고, `ScrollDescendantIntoView()`로 중앙에 정렬합니다.
- **사용자 입력과 자동 이동의 조율**: 터치 드래그 또는 스크롤 중에는 자동 정렬을 보류합니다. 레이아웃이 준비되고 입력이 끝난 뒤 정렬을 진행합니다.
- **선택 변경 이벤트**: 선택 대상이 바뀔 때 이전 항목의 선택 해제와 새 항목의 선택 이벤트를 전달합니다.
- **양 끝 항목의 정렬 공간**: 스크롤 영역 크기의 절반에 해당하는 `SSpacer`를 앞뒤에 배치해 첫 항목과 마지막 항목도 중앙에 도달할 수 있도록 합니다.
- **거리와 애니메이션의 연결**: 중앙에 가까워질수록 UMG 애니메이션의 재생 위치가 진행되도록 합니다. 연출을 바꾸려면 해당 애니메이션을 수정할 수 있습니다.
- **명시적인 항목 이동**: `RequestScrollInToView()`로 특정 항목을 중앙에 배치할 수 있습니다. 입력 중 요청은 `WidgetToFind`에 보류하며, 보류 중 추가 요청이 들어오면 마지막 요청으로 교체됩니다.

## 구현 구조

| 구성 | 역할 | 코드 |
| --- | --- | --- |
| `SAutoSelectScrollBox` | Slate의 입력 상태 확인, 중앙 거리 계산, 선택 대상 탐색과 정렬 | [AutoSelectScrollBox.h](Source/AutoSelectScroll/AutoSelectScrollBox.h), [AutoSelectScrollBox.cpp](Source/AutoSelectScroll/AutoSelectScrollBox.cpp) |
| `UAutoSelectScrollBox` | UMG 위젯과 Slate 구현 연결, 자식 슬롯 구성, 선택·스크롤 이벤트 전달 | [RebuildWidget 및 요청 처리](Source/AutoSelectScroll/AutoSelectScrollBox.cpp) |
| `UAutoSelectScrollBoxItem_Base` | 스크롤 이벤트를 구독하고 항목의 중앙 거리를 전달 | [AutoSelectScrollBoxItem.cpp](Source/AutoSelectScroll/AutoSelectScrollBoxItem.cpp) |
| `UAutoSelectScrollBoxItem_UseAnimation` | 중앙 거리를 UMG 애니메이션 재생 위치로 변환 | [애니메이션 연결](Source/AutoSelectScroll/AutoSelectScrollBoxItem.cpp) |

## 핵심 C++ 구현

아래 코드는 현재 저장소 구현에서 발췌했습니다. 전체 구현은 각 소스 링크에서 확인할 수 있습니다.

### 1. 입력 상태를 확인한 뒤 선택·정렬하기

[`ScrollModify()`](Source/AutoSelectScroll/AutoSelectScrollBox.cpp)는 먼저 `IsOk()`로 터치 드래그·스크롤 종료와 레이아웃 준비 여부를 확인합니다. 특정 항목으로 이동하라는 요청이 있으면 먼저 처리하고, 그 외에는 중앙에 가까운 항목을 탐색해 선택 변경 이벤트와 정렬을 연결합니다.

```cpp
void SAutoSelectScrollBox::ScrollModify()
{
    if (!IsOk())
        return;

    if (WidgetToFind)
    {
        ScrollDescendantIntoView(WidgetToFind->GetCachedWidget(), true, EDescendantScrollDestination::Center);
        WidgetToFind = nullptr;
        return;
    }

    if (!IsCenter(TargetWidget))
    {
        const auto FindWidget = FindTargetWidget();
        if (TargetWidget != FindWidget)
        {
            OnAutoUnSelectedScrollBoxItem.Broadcast(TargetWidget);
            TargetWidget = FindWidget;
            OnAutoSelectedScrollBoxItem.Broadcast(TargetWidget);
        }
        if (TargetWidget)
            ScrollDescendantIntoView(TargetWidget->GetCachedWidget(), true, EDescendantScrollDestination::Center);
    }
}
```

### 2. 같은 좌표계에서 중앙 거리 계산하기

항목의 절대 위치를 스크롤 영역의 로컬 좌표로 변환한 뒤, 항목 중심과 스크롤 영역 중심을 비교합니다. `Orientation`에 따라 X 또는 Y축을 사용하며, 거리를 항목 크기의 절반으로 나누어 연출에 사용할 값으로 정규화합니다. 중앙에서는 0, 항목 반 크기만큼 떨어지면 1이 됩니다.

[`GetCenterGab()`](Source/AutoSelectScroll/AutoSelectScrollBox.cpp) 중 좌표 계산 부분:

```cpp
const bool bIsVertical = Orientation == Orient_Vertical;
const FGeometry WidgetGeometry = FindChildGeometry(CachedGeometry, _Widget->TakeWidget());
const auto LocalSize = WidgetGeometry.GetLocalSize();
const auto FindWidgetVector = CachedGeometry.AbsoluteToLocal(WidgetGeometry.GetAbsolutePosition()) + (LocalSize / 2);
const auto MyVector = CachedGeometry.GetLocalSize() * FVector2D(0.5f, 0.5f);
const float WidgetPosition = bIsVertical ? FindWidgetVector.Y : FindWidgetVector.X;
const float MyPosition = bIsVertical ? MyVector.Y : MyVector.X;
return FMath::Abs<float>(WidgetPosition - MyPosition) / (bIsVertical ? LocalSize.Y / 2 : LocalSize.X / 2);
```

### 3. 중앙 거리를 UMG 애니메이션에 연결하기

스크롤 박스는 항목의 중앙 거리를 전달하고, 항목은 그 값을 애니메이션 재생 위치로 바꿉니다. 입력·정렬 로직과 시각적 연출을 서로 다른 클래스에서 담당하도록 구성했습니다.

[`UAutoSelectScrollBoxItem_UseAnimation::ScrollTick(float)`](Source/AutoSelectScroll/AutoSelectScrollBoxItem.cpp) 중 재생 위치 계산·적용 부분:

```cpp
const float fAnimEndTime = ScrollAni->GetEndTime();
const float fAnimCurrentTime = FMath::Clamp(fAnimEndTime * (1.f - _CenterGab), 0.0f, fAnimEndTime * 0.99);
if (UMGSequencePlayer)
{
    UMGSequencePlayer->SetCurrentTime(fAnimCurrentTime);

    if (0 < fAnimCurrentTime)
        UMGSequencePlayer->Play(fAnimCurrentTime, 1, EUMGSequencePlayMode::Forward, 0.001f, false);
    else
        UMGSequencePlayer->Pause();
}
```

전체 함수는 [항목 구현](Source/AutoSelectScroll/AutoSelectScrollBoxItem.cpp)에서 확인할 수 있습니다.

## 프로젝트 열기와 위젯 구성

1. 저장소를 내려받고 Unreal Engine 5.4에서 `AutoSelectScroll.uproject`를 엽니다. C++ 모듈 재빌드가 필요한 경우 프로젝트 파일을 생성해 빌드합니다.
2. UMG에서 `UAutoSelectScrollBox`를 배치하고, 스크롤 항목으로 `UAutoSelectScrollBoxItem_UseAnimation`을 상속한 Widget Blueprint를 구성합니다.
3. 항목에 중앙 접근에 따른 연출을 담은 UMG 애니메이션을 만들고, `ScrollAnimationName`에 해당 애니메이션 이름을 지정합니다.
4. 항목의 `RequestScrollIntoView()`를 호출하면 해당 항목으로 이동할 수 있습니다. C++에서 동적으로 항목을 추가할 때는 `UAutoSelectScrollBox::AddItem()`을 사용해 스크롤 이벤트도 연결합니다.
5. 선택 변경에 따른 화면 갱신은 `OnAutoSelectedScrollBoxItem`과 `OnAutoUnSelectedScrollBoxItem`에 연결합니다.

프로젝트의 모듈 구성은 [AutoSelectScroll.Build.cs](Source/AutoSelectScroll/AutoSelectScroll.Build.cs)에 있으며, `UMG`, `Slate`, `SlateCore`를 사용합니다.
