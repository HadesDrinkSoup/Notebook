### 用C++控制Pawn类第三人称镜头物体移动(X: Pitch，Y:Roll，Z:Yaw，X左右，Y前后，Z上下)

#### 创建父子级层级关系

`SetupAttachment(USceneComponent* InParent, FName InSocketName = NAME_None);`创建父子级层级关系

```c++
//InParent：要附加到的父场景组件。
//InSocketName（可选）：父组件上的插槽名称，用于精确指定附加点。
```

#### 增强输入

工作流程：初始化设置`BeginPlay()`添加输入映射 ==> 绑定连接`SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)`将输入事件与处理函数互相绑定 ==> 运行时触发处理函数

##### 增强输入必须添加以下三个头文件

```c++
//增强输入三件套
#include "InputActionValue.h" //输入映射Value值的头文件
#include "EnhancedInputComponent.h" //增强映射的头文件
#include "EnhancedInputSubsystems.h" //增强子系统的头文件
```

##### 添加输入映射上下文

在游戏开始或生成是要设置输入映射上下文

```c++
// 游戏开始或生成时调用
void APlayerPawnCase::BeginPlay()
{
	Super::BeginPlay();
	// 获取玩家控制器
	if (APlayerController* PC = Cast<APlayerController>(GetController()))
	{
		// 添加增强输入映射上下文
		if (UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer()))
		{
			Subsystem->AddMappingContext(DefaultMappingContext, 0);
		}
	}
}
```

##### 添加输入操作、操作映射、操作处理函数

```c++
//添加头文件
#include "EnhancedInputLibrary.h"

// 输入动作：移动
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
class UInputAction* MoveAction;

// 输入动作：观察（旋转视角）
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
class UInputAction* LookAction;

// 输入动作：镜头缩放
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
class UInputAction* ZoomAction;

// 默认输入映射上下文
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
class UInputMappingContext* DefaultMappingContext;

// 处理移动输入
void Move(const FInputActionValue& Value);

// 处理观察（旋转）输入
void Look(const FInputActionValue& Value);

// 处理缩放输入
void Zoom(const FInputActionValue& Value);
```

##### 设置玩家输入组件

`PlayerPawn.h`头文件添加函数

```c++
virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;
```

`PlayerPawn.cpp`中重写`SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent)`函数

设置输入组件绑定事件操作

```c++
// 设置玩家输入组件
void APlayerPawnCase::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);

	// 绑定增强输入动作
	if (UEnhancedInputComponent* EnhancedInputComponent = CastChecked<UEnhancedInputComponent>(PlayerInputComponent))
	{
		// 绑定移动动作
		EnhancedInputComponent->BindAction(MoveAction, ETriggerEvent::Triggered, this, &APlayerPawnCase::Move);
	
		// 绑定视角动作
		EnhancedInputComponent->BindAction(LookAction, ETriggerEvent::Triggered, this, &APlayerPawnCase::Look);
	
		// 绑定缩放动作
		EnhancedInputComponent->BindAction(ZoomAction, ETriggerEvent::Triggered, this, &APlayerPawnCase::Zoom);
	}

}
```

##### 处理移动输入

想要跟着镜头方向移动时，需根据控制器朝向计算向前和向右向量

```c++
// 处理移动输入
void APlayerPawnCase::Move(const FInputActionValue& Value)
{
	// 获取输入向量
	FVector2D MoveVector = Value.Get<FVector2D>();

	if (Controller != nullptr)
	{
		// 获取控制器的Yaw旋转（忽略Pitch和Roll）
		FRotator YawRotation(0, Controller->GetControlRotation().Yaw, 0);
	
		// 根据控制器朝向计算前进和右方向量
		FVector ForwardDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
		FVector RightDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);
	
		// 应用移动输入
		AddMovementInput(ForwardDirection, MoveVector.Y); // 前后移动
		AddMovementInput(RightDirection, MoveVector.X); // 左右移动
	}

}
```

##### 处理视角旋转输入

```c++
// 处理视角旋转输入
void APlayerPawnCase::Look(const FInputActionValue& Value)
{
	// 获取输入向量
	FVector2D LookAxisVector = Value.Get<FVector2D>();

	if (Controller != nullptr)
	{
		// 获取当前控制器旋转
		FRotator CurrentRotation = Controller->GetControlRotation();
	
		// 计算新的俯仰角（上下看），限制在-80到80度之间
		float NewPitch = FMath::Clamp(CurrentRotation.Pitch + LookAxisVector.Y, -80.0f, 80.0f);
	
		// 计算新的偏航角（左右看）
		float NewYaw = CurrentRotation.Yaw + LookAxisVector.X;
	
		// 应用新的旋转，Roll固定为0
		Controller->SetControlRotation(FRotator(NewPitch, NewYaw, 0.0f));
	}

}
```

想要操作物体或者玩家时须添加移动组件

```c++
//浮空移动组件UFloatingPawnMovement
UFloatingPawnMovement* MovementComp=
CreateDefaultSubobject<UFloatingPawnMovement(TEXT("MovementComp");
/* 
角色移动组件UCharacterMovementComponent
UCharacterMovementComponent组件时专为ACharacter类设计的组件，其他组件不适用
*/        
UCharacterMovementComponent* MovementComp =                               CreateDefaultSubobject<UCharacterMovementComponent(TEXT("MovementComp"));
```

##### 第三人称镜头摄像机配置

需要设置摄像机不随控制器旋转、禁用控制器旋转

```c++
// 配置摄像机属性
CameraComp->bUsePawnControlRotation = false; // 摄像机不直接跟随控制器旋转
// 禁用控制器旋转对Pawn旋转的影响
bUseControllerRotationYaw = false;
bUseControllerRotationPitch = false;
bUseControllerRotationRoll = false;
```

##### 源码

###### `PlyaerPawn.h`

```c++
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Pawn.h"
#include "Camera/CameraComponent.h"
#include "GameFramework/SpringArmComponent.h"
#include "GameFramework/FloatingPawnMovement.h"
#include "EnhancedInputLibrary.h"
#include "PlayerPawnCase.generated.h"

// 自定义玩家Pawn类，继承自APawn
UCLASS()
class MYCPPDEMO_API APlayerPawnCase : public APawn
{
    GENERATED_BODY()

public:
    // 构造函数
    APlayerPawnCase();

    // 静态网格组件，用于视觉表示
    UPROPERTY(VisibleAnywhere, Category = "Components")
    class UStaticMeshComponent* StaticMeshComp;
    
    // 摄像机组件，用于玩家视角
    UPROPERTY(VisibleAnywhere, Category = "Components")
    class UCameraComponent* CameraComp;
    
    // 弹簧臂组件，用于平滑摄像机移动
    UPROPERTY(VisibleAnywhere, Category = "Components")
    class USpringArmComponent* SpringArmComp;
    
    // 浮动Pawn移动组件，处理移动逻辑
    UPROPERTY(VisibleAnywhere, Category = "Components")
    class UFloatingPawnMovement* MovementComp;
    
    // 缩放灵敏度，控制缩放速度
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Movement")
    float ZoomSensitivity = 20.0f;
    
    // 最小缩放距离，限制弹簧臂长度
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Movement")
    float MinZoomLength = 200.0f;
    
    // 最大缩放距离，限制弹簧臂长度
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Movement")
    float MaxZoomLength = 800.0f;

protected:
    // 游戏开始或生成时调用
    virtual void BeginPlay() override;

    // 输入动作：移动
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
    class UInputAction* MoveAction;
    
    // 输入动作：观察（旋转视角）
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
    class UInputAction* LookAction;
    
    // 输入动作：镜头缩放
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
    class UInputAction* ZoomAction;
    
    // 默认输入映射上下文
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
    class UInputMappingContext* DefaultMappingContext;
    
    // 处理移动输入
    void Move(const FInputActionValue& Value);
    
    // 处理观察（旋转）输入
    void Look(const FInputActionValue& Value);
    
    // 处理缩放输入
    void Zoom(const FInputActionValue& Value);

public:
    // 每帧调用
    virtual void Tick(float DeltaTime) override;

    // 设置玩家输入组件
    virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;

};
```

###### `PlyaerPawn.cpp`

```c++
#include "PlayerPawnCase.h"
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"

// 设置默认值
APlayerPawnCase::APlayerPawnCase()
{
	// 设置此Pawn每帧调用Tick()。如果不需要，可以关闭以提高性能。
	PrimaryActorTick.bCanEverTick = true;

	// 创建根组件
	RootComponent = CreateDefaultSubobject<USceneComponent>(TEXT("RootComponent"));
	// 创建静态网格组件
	StaticMeshComp = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("MeshComp"));
	// 创建摄像机组件
	CameraComp = CreateDefaultSubobject<UCameraComponent>(TEXT("CameraComp"));
	// 创建弹簧臂组件
	SpringArmComp = CreateDefaultSubobject<USpringArmComponent>(TEXT("SpringArmComp"));
	// 创建浮动Pawn移动组件
	MovementComp = CreateDefaultSubobject<UFloatingPawnMovement>(TEXT("MovementComp"));
	
	// 设置组件层级关系
	StaticMeshComp->SetupAttachment(RootComponent);
	SpringArmComp->SetupAttachment(StaticMeshComp);
	CameraComp->SetupAttachment(SpringArmComp, USpringArmComponent::SocketName);
	
	// 配置弹簧臂属性
	SpringArmComp->TargetArmLength = 400.0f; // 初始弹簧臂长度
	SpringArmComp->bEnableCameraLag = true; // 启用摄像机延迟
	SpringArmComp->CameraLagSpeed = 3.0f; // 摄像机延迟速度
	SpringArmComp->SetRelativeLocation(FVector(0.0f, 0.0f, 50.0f)); // 设置相对位置
	SpringArmComp->SocketOffset = FVector(0.0f, 0.0f, 200.0f); //设置插槽偏移 - 这将使摄像机位置相对于弹簧臂末端有一定偏移
	SpringArmComp->bUsePawnControlRotation = true; // 弹簧臂使用Pawn控制旋转
	
	// 配置摄像机属性
	CameraComp->bUsePawnControlRotation = false; // 摄像机不直接跟随控制器旋转
	
	// 自动拥有玩家0
	AutoPossessPlayer = EAutoReceiveInput::Player0;
	
	// 禁用控制器旋转对Pawn旋转的影响
	bUseControllerRotationYaw = false;
	bUseControllerRotationPitch = false;
	bUseControllerRotationRoll = false;

}

// 游戏开始或生成时调用
void APlayerPawnCase::BeginPlay()
{
	Super::BeginPlay();
	// 获取玩家控制器
	if (APlayerController* PC = Cast<APlayerController>(GetController()))
	{
		// 添加增强输入映射上下文
		if (UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer()))
		{
			Subsystem->AddMappingContext(DefaultMappingContext, 0);
		}
	}
}

// 处理移动输入
void APlayerPawnCase::Move(const FInputActionValue& Value)
{
	// 获取输入向量
	FVector2D MoveVector = Value.Get<FVector2D>();

	if (Controller != nullptr)
	{
		// 获取控制器的Yaw旋转（忽略Pitch和Roll）
		FRotator YawRotation(0, Controller->GetControlRotation().Yaw, 0);
	
		// 根据控制器朝向计算前进和右方向量
		FVector ForwardDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
		FVector RightDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);
	
		// 应用移动输入
		AddMovementInput(ForwardDirection, MoveVector.Y); // 前后移动
		AddMovementInput(RightDirection, MoveVector.X); // 左右移动
	}

}

// 处理视角旋转输入
void APlayerPawnCase::Look(const FInputActionValue& Value)
{
	// 获取输入向量
	FVector2D LookAxisVector = Value.Get<FVector2D>();

	if (Controller != nullptr)
	{
		// 获取当前控制器旋转
		FRotator CurrentRotation = Controller->GetControlRotation();
	
		// 计算新的俯仰角（上下看），限制在-80到80度之间
            float NewPitch = FMath::Clamp(CurrentRotation.Pitch + LookAxisVector.Y, -80.0f, 80.0f);
	
		// 计算新的偏航角（左右看）
		float NewYaw = CurrentRotation.Yaw + LookAxisVector.X;
	
		// 应用新的旋转，Roll固定为0
		Controller->SetControlRotation(FRotator(NewPitch, NewYaw, 0.0f));
	}

}

// 处理缩放输入
void APlayerPawnCase::Zoom(const FInputActionValue& Value)
{
	// 获取缩放输入值
	float ZoomValue = Value.Get<float>();

	// 计算新的弹簧臂长度
	float NewTargetArmLength = SpringArmComp->TargetArmLength + ZoomValue * ZoomSensitivity;
	
	// 限制弹簧臂长度在最小和最大值之间
	NewTargetArmLength = FMath::Clamp(NewTargetArmLength, MinZoomLength, MaxZoomLength);
	
	// 应用新的弹簧臂长度
	SpringArmComp->TargetArmLength = NewTargetArmLength;

}

// 每帧调用
void APlayerPawnCase::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);
	// 此处可以添加每帧更新的逻辑
}

// 设置玩家输入组件
void APlayerPawnCase::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);

	// 绑定增强输入动作
	if (UEnhancedInputComponent* EnhancedInputComponent = CastChecked<UEnhancedInputComponent>(PlayerInputComponent))
	{
		// 绑定移动动作
		EnhancedInputComponent->BindAction(MoveAction, ETriggerEvent::Triggered, this, &APlayerPawnCase::Move);
	
		// 绑定视角动作
		EnhancedInputComponent->BindAction(LookAction, ETriggerEvent::Triggered, this, &APlayerPawnCase::Look);
	
		// 绑定缩放动作
		EnhancedInputComponent->BindAction(ZoomAction, ETriggerEvent::Triggered, this, &APlayerPawnCase::Zoom);
	}

}
```

