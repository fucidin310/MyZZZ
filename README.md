# MyZZZ

이 프로젝트는 **액션 RPG**로 **호요버스**의 **젠레스 존 제로**와 **원신**을 참고하여 만들었습니다. 언리얼 엔진을 사용하여 제작했습니다.

- **개발 기간**: 약 100일
- **개발 인원**: 1인
- **사용 엔진**: 언리얼 엔진 5.5.4

## 플레이 영상

[![Video Label](http://img.youtube.com/vi/12kmY2aP2SY/0.jpg)](https://youtu.be/12kmY2aP2SY)

## 플레이용 빌드 버전

https://drive.google.com/file/d/1ZYOvYS56eSa4_rt6HCc5HcEDY9wDpYjW/view?usp=drive_link

- **WASD**: 이동
- **마우스 좌클릭**: 일반공격
- **마우스 우클릭**: 회피
- **E**: 스킬
- **R**: 궁극기
- **Space**: 교체
- **ESC**: 메뉴
- **QE**: 메뉴를 띄운 상태에서 세부 매뉴 전환

## 개발용 프로젝트 파일

https://drive.google.com/file/d/18zehQXnHLd8GQER4SQSt7WthxxlE0XsJ/view?usp=drive_link

## 프로젝트 개요

- **전투**: GAS를 이용해 전투 시스템 구현, 플레이블 캐릭터는 일반공격, 스킬, 궁극기, 회피, 회피공격을 가지고, 버프, 디버프 등을 가집니다.
- **대화**: CustomAsset을 이용하여 대화 에셋을 제작할 수 있습니다. 대화문과 선택지를 구현할 수 있고, 선택지에 따라 대화 분기가 갈라질 수 있습니다.
- **퀘스트**: 특정 몬스터 사냥, 특정 NPC와 대화 등 퀘스트 목표와 보상, 퀘스트 설명 등을 만들 수 있습니다.
- **인벤토리**: 성유물, 모라(돈), 강화 아이템 등을 저장할 수 있습니다.
- **강화**: 캐릭터의 레벨, 스킬 레벨을 올릴 수 있습니다. 성유물을 장착하여 캐릭터의 스펙을 올릴 수 있고, 같은 종류의 성유물을 2개, 4개를 장착하면 각각 이로운 효과를 얻습니다.
- **세이브 로드**: 게임을 종료하면 자동으로 진행사항이 저장됩니다. SaveGames 폴더를 삭제하면 새 게임을 할 수 있습니다.

## 전투

[![Video Label](http://img.youtube.com/vi/AfuFAma8Gmo/0.jpg)](https://youtu.be/AfuFAma8Gmo)

<details>
<summary>데미지 로직</summary>
<br />
<img width="756" height="416" alt="Image" src="https://github.com/user-attachments/assets/71446975-f29e-4eb4-ac89-6ff9576c797e" />
<br />
SkillEffectDatas를 수정해 데미지를 부여할 때 사용할 정보를 구성한다.

예를 들어 방어도 50% + 공격력 70%만큼 데미지를 준다고 하자.
<br />
그럼 Stat Modifiers를 2개 만들고,
<br />
첫번째에는 Attribute에 DefancePower, Multi Handle에 0.5를 넣어준다. 
<br />
두번째에는 Attribute에 AttackPower, Multi Handle에 0.7를 넣어준다.
<br />
데미지의 경우에는 스킬에 레벨에 따라 변화하기 때문에 커브드 테이블을 넣도록 했다.
<br />
이제 해당 데미지가 발동해야 하는 타이밍에 해당되는 구조채를 넘겨 아래의 함수를 실행하게 한다.

<details>
<summary>ApplyDamageEffect</summary>

```
void UMyGameplayAbility::ApplyDamageEffect(AActor* TargetActor, const FGameplayEffectSpecHandle& InSpecHandle, float DamageAmount)
{
	if (!InSpecHandle.IsValid() || !TargetActor) return;

	FGameplayEffectSpec* Spec = InSpecHandle.Data.Get();
	Spec->SetSetByCallerMagnitude(MyGameplayTags::SetByCaller_BaseDamage, DamageAmount);

	if (AMyZZZCharacter* TargetCharacter = Cast<AMyZZZCharacter>(TargetActor))
	{
		UAbilitySystemComponent* TargetAbilitySystemComponent = TargetCharacter->GetAbilitySystemComponent();

		check(TargetAbilitySystemComponent && InSpecHandle.IsValid());

		if (UMyAbilitySystemComponent* TargetMyAbilitySystemComponent = Cast<UMyAbilitySystemComponent>(TargetAbilitySystemComponent))
		{
			TargetMyAbilitySystemComponent->ApplyGameplayEffectSpecToTarget(
				*InSpecHandle.Data,
				TargetAbilitySystemComponent
			);
		}
	}
}
```
</details>

그럼 SetByCaller로 기본데미지가 타겟으로 넘어가고, 데미지를 부여하는 게임이펙트가 실행된다.
<br />
그럼 아래의 UGEExecCalc_Damage에 의해 최종 데미지가 계산되고 부여된다.

<details>
<summary>UGEExecCalc_Damage::Execute_Implementation</summary>
	
```
void UGEExecCalc_Damage::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
    const FGameplayEffectSpec& EffectSpec = ExecutionParams.GetOwningSpec();

    FAggregatorEvaluateParameters EvalParam;
    EvalParam.SourceTags = EffectSpec.CapturedSourceTags.GetAggregatedTags();
    EvalParam.TargetTags = EffectSpec.CapturedTargetTags.GetAggregatedTags();

	float SourceBaseDamage = EffectSpec.GetSetByCallerMagnitude(MyGameplayTags::SetByCaller_BaseDamage, false, 0.f);

	float TargetDefensePower = 0.f;
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(GetMyDamageCapture().DefensePowerDef, EvalParam, TargetDefensePower);

	float SourceDamageBoost = 1.f;
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(GetMyDamageCapture().DamageBoostDef, EvalParam, SourceDamageBoost);

	float SourceCriticalChance = 0.f;
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(GetMyDamageCapture().CriticalChanceDef, EvalParam, SourceCriticalChance);

	float SourceCriticalDamage = 0.f;

	bool bIsCritical = false;
	EDamageType DamageType = EDamageType::Physical;

	float RandomChance = FMath::RandRange(0.0f, 100.0f);
	if (SourceCriticalChance > RandomChance || SourceCriticalChance >= 100.f)
	{
		ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(GetMyDamageCapture().CriticalDamageDef, EvalParam, SourceCriticalDamage);
		bIsCritical = true;
	}

	const float FinalDamage = SourceBaseDamage * (100 / (100 + TargetDefensePower)) * (1 + (SourceCriticalDamage / 100)) * SourceDamageBoost;
	UE_LOG(LogTemp, Warning, TEXT("%f * (100 / (100 + %f)) * (1 + (%f / 100)) * %f"), SourceBaseDamage, TargetDefensePower, SourceCriticalDamage, SourceDamageBoost);

	if (FinalDamage > 0.f)
	{
		OutExecutionOutput.AddOutputModifier(
			FGameplayModifierEvaluatedData(
				GetMyDamageCapture().DamageInfoProperty,
				EGameplayModOp::Override,
				UMyFunctionLibrary::MakeDamageInfo(bIsCritical, DamageType)
			)
		);

		OutExecutionOutput.AddOutputModifier(
			FGameplayModifierEvaluatedData(
				GetMyDamageCapture().DamageTakenProperty,
				EGameplayModOp::Override,
				FinalDamage
			)
		);
	}
}
```

</details>
</details>

<details>
<summary>모션 워핑 로직</summary>
	
<img width="762" height="242" alt="Image" src="https://github.com/user-attachments/assets/1dc86920-a0e4-4ca3-ac7f-f082e802f099" />

모션 워핑을 위한 구조체다.
- DiatanceWeight, AngleWeight:
  <br />
  가장 적합한 타겟을 찾을때 필요한 가중치다.
  <br />
  예를 들어 100m 앞에 있고 카메라 방향과 45도 떨어진 적과 200m 앞에 있고 카메라 방향과 10도 떨어진 적 중 어떤 적에게 접근할지 정해야 할때, 거리 가중치와 각도 가중치의 비율을 조절한다.
  <br />
  예를 들어 나는 거리와 상관없이 카메라 방향과 가장 일치하는 적에게 접근하겠다하면 거리 가중치를 0으로 둔다.
- MaxDistance, MaxAngle:
  <br />
  접근 가능한 최대 거리와 최대 각도이다.
  <br />
  위 이미지의 경우 거리가 10m, 각도는 30도 이내의 적만 인식한다.
- Approch Diatance, ShouldMoveBack:
  <br />
  얼마나 접근할지, 타겟의 뒤로 이동할지 정하는 값이다.
- TargetLocationName, TargetRotaionName:
  <br />
  모션워핑의 WarpTargetName이다.

위에서 만든 구조체를 아래 함수에 넘겨 실행하면 모션 워핑이 적용된다.

<details>
<summary>ApprochBestTarget</summary>

```
void UMyFunctionLibrary::ApprochBestTarget(const UObject* WorldContextObject, AActor* Owner, const FTargetApproachParams& TargetApproachParams)
{
	AMyZZZCharacter* OwnerCharacter = Cast<AMyZZZCharacter>(Owner);
	if (!OwnerCharacter) return;

	UMotionWarpingComponent* OwnerMotionWarpingComponent = OwnerCharacter->GetMotionWarpingComponent();
	if (!OwnerMotionWarpingComponent) return;
	OwnerMotionWarpingComponent->RemoveAllWarpTargets();

	UWorld* World = nullptr;
	if (GEngine) World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull);
	if (!World) return;

	FVector OwnerLocation = OwnerCharacter->GetActorLocation();

	TArray<FHitResult> HitResults;

	FCollisionQueryParams QueryParams;
	QueryParams.AddIgnoredActor(Owner);

	bool bHit = World->SweepMultiByProfile(
		HitResults,
		OwnerLocation,
		OwnerLocation + FVector(0, 0, 100),
		FQuat::Identity,
		FName("PawnProfile"),
		FCollisionShape::MakeSphere(TargetApproachParams.MaxDistance),
		QueryParams
	);

	if (!bHit) return;

	AActor* BestActor = FindBestTargetFromHits(Owner, HitResults, TargetApproachParams.DistanceWeight, TargetApproachParams.AngleWeight, TargetApproachParams.MaxDistance, TargetApproachParams.MaxAngle);
	if (!BestActor) return;

	FVector TargetLocation = BestActor->GetActorLocation();
	OwnerMotionWarpingComponent->AddOrUpdateWarpTargetFromLocation(TargetApproachParams.TargetRotationName, TargetLocation);

	FVector ToTarget = (TargetLocation - OwnerLocation);
	if (TargetApproachParams.bShouldMoveBack || (!TargetApproachParams.bShouldMoveBack && ToTarget.Length() > TargetApproachParams.ApprochDistance))
	{
		OwnerMotionWarpingComponent->AddOrUpdateWarpTargetFromLocation(TargetApproachParams.TargetLocationName, TargetLocation - ToTarget.GetSafeNormal() * TargetApproachParams.ApprochDistance);
	}
	else
	{
		OwnerMotionWarpingComponent->RemoveWarpTarget(TargetApproachParams.TargetLocationName);
	}
}
```
</details>
</details>
