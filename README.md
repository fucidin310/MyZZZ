# MyZZZ

이 프로젝트는 **액션 RPG**로 **호요버스**의 **젠레스 존 제로**와 **원신**을 참고하여 만들었습니다. 언리얼 엔진을 사용하여 제작했습니다.

- **개발 기간**: 50일+
- **개발 인원**: 1인
- **사용 엔진**: 언리얼 엔진 5.5.4

## 플레이 영상

[![Video Label](http://img.youtube.com/vi/12kmY2aP2SY/0.jpg)](https://youtu.be/12kmY2aP2SY)

## 플레이용 빌드 버전

https://drive.google.com/file/d/1ZYOvYS56eSa4_rt6HCc5HcEDY9wDpYjW/view?usp=drive_link

- **WASD**: 이동
- **F**: 상호작용
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
- **퀘스트**: 특정 몬스터 사냥, 특정 NPC와 대화 등 퀘스트 목표와 보상, 퀘스트 설명 등을 만들 수 있습니다.
- **대화**: CustomAsset을 이용하여 대화 에셋을 제작할 수 있습니다. 대화문과 선택지를 구현할 수 있고, 선택지에 따라 대화 분기가 갈라질 수 있습니다.
- **인벤토리**: 성유물, 모라(돈), 강화 아이템 등을 저장할 수 있습니다.
- **강화**: 캐릭터의 레벨, 스킬 레벨을 올릴 수 있습니다. 성유물을 장착하여 캐릭터의 스펙을 올릴 수 있고, 같은 종류의 성유물을 2개, 4개를 장착하면 각각 이로운 효과를 얻습니다.
- **그 외**: 세이브 로드, 보물 상자, 경치 카메라, 스토리, 카메라와 겹칠 시 반투명화 등

## 전투

[![Video Label](http://img.youtube.com/vi/AfuFAma8Gmo/0.jpg)](https://youtu.be/AfuFAma8Gmo)<br />
Countess, Phase, Aurora의 스킬을 설명한 영상
<br />
### SkillEffectDatas
<br />
<img width="756" height="416" alt="Image" src="https://github.com/user-attachments/assets/71446975-f29e-4eb4-ac89-6ff9576c797e" />
<br />
SkillEffectDatas를 수정해 데미지를 부여할 때 사용할 정보를 구성한다.
<br />
<br />
예를 들어 방어도 50% + 공격력 70%만큼 데미지를 준다고 하자.
<br />
그럼 Stat Modifiers를 2개 만들고,
<br />
첫번째에는 Attribute에 DefancePower, Multi Handle에 0.5를 넣어준다. 
<br />
두번째에는 Attribute에 AttackPower, Multi Handle에 0.7를 넣어준다.
<br />
데미지 배율의 경우에는 스킬에 레벨에 따라 변화하기 때문에 커브드 테이블을 넣도록 했다. 데미지를 계산할 때 스킬의 레벨에 맞는 값을 가져온다.
<br />
<br />
이제 해당 데미지가 발동해야 하는 타이밍에 해당되는 구조채를 넘겨 아래의 함수를 실행하게 한다.
<br />
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
<br />
그럼 SetByCaller로 기본데미지가 타겟으로 넘어가고, 데미지를 부여하는 게임이펙트가 실행된다.
<br />
그럼 아래의 UGEExecCalc_Damage에 의해 최종 데미지가 계산되고 부여된다.
<br />
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
<br />	

### 모션 워핑
<img width="762" height="242" alt="Image" src="https://github.com/user-attachments/assets/1dc86920-a0e4-4ca3-ac7f-f082e802f099" />

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

## 퀘스트

### Quest
<img width="749" height="810" alt="Image" src="https://github.com/user-attachments/assets/1505e0c9-6009-44a3-a5d9-5ebc05ca1460" />
<br />
<br />

- Title: 말그대로 퀘스트의 타이틀
- QuestStage: 퀘스트의 진행 단계
  - QuestStageStartGameEffectList, QuestStageClearGameEffectList: 각각 QuestStage를 시작할때와 성공할때 적용할 GameEffectList, 사실 하나만 있어도 되지만 햇갈려서 둘다 만들었다. GameEffectList는 아래서 따로 설명
  - QuestObjectives: 퀘스트의 세부 목표, 예를 들면 NPC A에게 돈 주기라는 QuestStage가 있으면 특정 액수만큼 돈 보유와 NPC A와 대화하기가 QuestObjectives인 식, 현재는 특정 NPC와 대화하기, 특정 몬스터 잡기가 구현됨
- Rewards: 퀘스트 클리어 시 보상, 꼭 퀘스트가 아니더라도 던전 클리어나 보물 상자를 통해서도 얻을 수 있도록 따로 클래스로 뺐다.
  - Common Reward: 특정 아이템을 특정 수량만큼 획득
  - Handle GameEffect: 특정 GameEffect를 활성화 혹은 비활성화
  - Reward Random Amount: 특정 아이템을 랜덤한 수량만큼 획득
  - Reward Random Choice: 정해진 아이템 풀에서 특정 수량만큼 랜덤 획득

<br />
위처럼 만든 퀘스트 에셋을 PlayerState에 있는 QuestComponent의 함수 AddQuest에 추가하면 Quest를 추가할 수 있다. 보통 대화를 통해 추가한다.

<details>
<summary>AddQuest</summary>
	
```
void UMyQuestComponent::AddQuest(TSubclassOf<UQuest> NewQuest)
{
	if (ActiveQuests.Find(NewQuest)) return;

	UQuest* QuestInstance = NewObject<UQuest>(this, NewQuest);
	QuestInstance->OnQuestActivated();
	ActiveQuests.Add(NewQuest, QuestInstance);
	OnQuestAdded.Broadcast(QuestInstance);
}
```

</details>

Quest를 TMap에 TSubclassOf<UQuest>를 키, UQuest 인스턴스를 벨류로 저장해 같은 Quest가 중복으로 저장되지 않도록 했다.
<br />
<br />
### GameEffect
<br />
퀘스트A를 진행할 경우 원래 마을에 있던 NPC가 농장으로 이동한다고 하자.<br />
그 때, 퀘스트A를 취소할 경우 다시 농장에 있는 NPC가 마을로 이동해야한다.<br />
그렇기에 활성화/비활성화 시 작동/원상복구 되는 효과가 필요하다고 생각했다.<br />
<br />
GameEffect에는 ActivateGameEffect/DeactivateGameEffect 라는 함수가 있다.<br />
여기에 작동/원상복구를 각각 구현한다.<br />
그리고 커스텀한 GameEffect를 GameState에 있는 GameEffectSystemComponent의 ApplyGameEffect/RemoveGameEffect를 통해 활성화/비활성화 시킨다.<br /><br />
<img width="481" height="282" alt="Image" src="https://github.com/user-attachments/assets/c8eb74d5-4cb2-4c2a-91bd-a3913639c142" /><br /><br />

- Components: GameEffect에 적용할 효과, GameplayEffect를 참고해서 만들었다.<br />
  - Handle DialogueMode: DialogueMode를 추가하거나 제거하는 효과, 어떤 레벨의 어떤 NPC에게 어떤 DialogueMode를 추가/제거 할지 정할 수 있다.<br />
  - Toggle Visibly: 특정 NPC를 Visible/Hidden 상태로 변경하는 효과, 어떤 레벨의 어떤 NPC에게 적용할지 정할 수 있다.<br />
  
<br />
이는 퀘스트에서만 사용하는 건 아니다.<br />
예를 들면 이 게임에서는 게임에 처음 접속하면 처음 만나는 촌장 NPC를 제외한 다른 NPC를 Hidden 상태로 바꿔야 한다.<br />
이를 위해 기본상태인 다른 NPC는 Visible, 처음만나는 촌장 NPC는 Hidden인 상태에서<br />
다른 NPC는 Hidden, 처음 만나는 촌장은 Visible로 바꾸는 GameEffect를 만들고,<br />
InitGameEffectDataAssets에 넣고, 그걸 MyGameInstance의 변수에 넣는다.<br /><br />
<img width="573" height="202" alt="Image" src="https://github.com/user-attachments/assets/9faaa4b5-1639-475d-85de-88d62d157f13" /><br /><br />
그럼 마지막으로 플레이한 버전이 InitGameEffectDataAssets의 버전보다 낮으면 해당 GameEffect를 적용한다. 물론 이것도 비활성화 할 수 있다.<br />
이렇게 하면 처음 0.0.0.1 버전 혹은 더 높은 버전을 플레이할 경우 다른 NPC는 Hidden, 처음 만나는 촌장은 Visible로 바뀌게 되고,<br />
퀘스트를 진행하면서 점점 GameEffect를 비활성화 하다보면, 만날 수 있는 NPC가 생기게 된다.<br />
<br />
<br />

## 대화

[![Video Label](http://img.youtube.com/vi/lM4dApp4Mpo/0.jpg)](https://youtu.be/lM4dApp4Mpo)<br />
DialogueAsset을 만들고 그걸 NPC의 DialogueComponent에 적용시킨 후 대화하는 영상<br />
<br />
### DialogueAsset
<br />
<img width="716" height="439" alt="Image" src="https://github.com/user-attachments/assets/f46a1b0f-1f77-4bea-99f2-f427fb266dda" /><br /><br />
위에서처럼 만든 CustomGraph를 통해 만든 DialogueAsset을 아래처럼 DialogueComponent를 가진 NPCCharacter에 넣어주면
<br />
NPC 근처에서 플레이어가 상호작용을 하면 NPC에 구현된 Interact함수에서 해당하는 Dialogue를 재생한다.
<br />
<img width="675" height="175" alt="Image" src="https://github.com/user-attachments/assets/b59545a4-428e-4d4b-b672-6dcedee16a45" />
<br />
- DialogueAsset: 말그대로 DialogueAsset
- DialogueMode: 어떤 DialogueMode일 때 재생할건지
- DialogueEffect: 대화가 종료될때 적용할 효과(예: 퀘스트 추가, DialogueMode 추가 및 삭제, 퀘스트 이벤트 보내기 등)
<br />
<br />
위 사진에서는 DialogueMode가 1001일 때, DA_AllanStory_First를 재생하고, 대화가 끝나면 Quest_AllanStory를 추가한다.
<br />
<br />

### DialogueMode
<br />
예를 들어 어떤 NPC가 평소, 퀘스트 A를 진행할 때, 퀘스트 B를 진행할 때 다른 대화문을 재생한다고 할 때<br />
서로 다른 퀘스트가 어떤 대화문을 재생해야하는지를 덮어쓰는 문제가 생길 수 있어서,<br />
int형 배열을 만들어 저장하고 가장 높은 숫자의 DialogueMode를 재생하게 하였다.<br />
그럼 퀘스트에서 어떤 Dialogue를 재생하게 할건지 정하면 되지 않나 할 수 도 있지만<br />
예를 들어 어떤 퀘스트를 완료하기 전후나 혹은 어떤 지역에 다녀온 전후로 대화문이 바뀌는 경우처럼<br />
꼭 퀘스트가 어떤 대화를 할지 정하는 게 아니기에 NPC 자체에서 어떤 대화문을 재생할지 정하게 구현했다.<br />
<br />

## 인벤토리

![Image](https://github.com/user-attachments/assets/1220c0b6-c434-4562-8dd7-a32bd32a9daf)
<br />

현재, 강화아이템, 귀중품, 성유물 인벤토리가 있고,<br />
강화아이템, 귀중품은 TMap<UClass*, int>으로 성유물은 TArray<UItem*>으로 저장하고 있다.<br />
성유물의 경우에는 같은 Class라도 각 인스턴스마다 부여하는 능력치가 다르기 때문에 인스턴스로 저장한다.<br />
아이템을 인벤토리로 추가할 때는 PlayerState의 InventoryComponent의 AddItem 함수에 원하는 아이템 클래스와 수량을 넘겨 실행한다.<br />

<details>
<summary>AddItem</summary>
	
```
void UMyInventoryComponent::AddItem(TSubclassOf<UItem> NewItem, int Amount)
{
	if (NewItem == nullptr) return;

	bool bIsFinded = false;

	switch (NewItem.GetDefaultObject()->GetItemType())
	{
	case EItemType::LevelUpMaterial:
		if (LevelUpMaterialInventory.Contains(NewItem))
		{
			bIsFinded = true;
			LevelUpMaterialInventory[NewItem] += Amount;
		}
		break;
	case EItemType::Precious:
		if (PreciousInventory.Contains(NewItem))
		{
			bIsFinded = true;
			PreciousInventory[NewItem] += Amount;
		}
		break;
	case EItemType::Artifact:
		for (int i = 0; i < Amount; i++)
		{
			UArtifact* ArtifactInstance = NewObject<UArtifact>(this, NewItem);
			ArtifactInstance->MakeRandomBonus();
			ArtifactInventory.Add(ArtifactInstance);
			OnItemAdded.Broadcast(Cast<UItem>(ArtifactInstance), 1);
		}
		bIsFinded = true;
		break;
	default:
		break;
	}

	if (!bIsFinded)
	{
		switch (NewItem.GetDefaultObject()->GetItemType())
		{
		case EItemType::LevelUpMaterial:
			LevelUpMaterialInventory.Add(NewItem, Amount);
			break;
		case EItemType::Precious:
			PreciousInventory.Add(NewItem, Amount);
			break;
		default:
			break;
		}
	}

	switch (NewItem.GetDefaultObject()->GetItemType())
	{
	case EItemType::LevelUpMaterial:
	case EItemType::Precious:
		OnItemAdded.Broadcast(NewItem.GetDefaultObject(), Amount);
		break;
	default:
		break;
	}
}
```

</details>
<br />

성유물의 스팩은 이때 정해진다.

<details>
<summary>MakeRandomBonus</summary>
	
```
void UArtifact::MakeRandomBonus()
{
	TArray<int32> BonusTypeIndices;
	for (int32 i = 0; i < (int32)EEquipmentBonusType::MAX; i++)
	{
		BonusTypeIndices.Add(i);
	}

	if (BonusTypeIndices.Num() >= 4)
	{
		Algo::RandomShuffle(BonusTypeIndices);

		for (int32 i = 0; i < 4; i++)
		{
			FEquipmentBonusData NewBonus;
			NewBonus.EquipmentBonusType = (EEquipmentBonusType)BonusTypeIndices[i];
			UMyFunctionLibrary::AddEquipmentBonus(NewBonus);
			ArtifactEquipmentBonusData.Add(NewBonus);
		}
	}
}
```

</details>
<br />

<details>
<summary>AddEquipmentBonus</summary>
	
```
void UMyFunctionLibrary::AddEquipmentBonus(FEquipmentBonusData& EquipmentBonusData)
{
	int BonusAmount = FMath::RandRange(0, 4);;

	float AddValue = 0.f;

	switch (EquipmentBonusData.EquipmentBonusType)
	{
	case EEquipmentBonusType::Health:
		AddValue = 209 + 30 * BonusAmount;
		break;
	case EEquipmentBonusType::HealthRatio:
		AddValue = 0.041 + 0.005 * BonusAmount;
		break;
	case EEquipmentBonusType::AttackPower:
		AddValue = 14 + 2 * BonusAmount;
		break;
	case EEquipmentBonusType::AttackPowerRatio:
		AddValue = 0.041 + 0.005 * BonusAmount;
		break;
	case EEquipmentBonusType::DefensePower:
		AddValue = 16 + 3 * BonusAmount;
		break;
	case EEquipmentBonusType::DefensePowerRatio:
		AddValue = 0.051 + 0.007 * BonusAmount;
		break;
	case EEquipmentBonusType::CriticalDamage:
		AddValue = 0.054 + 0.008 * BonusAmount;
		break;
	case EEquipmentBonusType::CriticalChance:
		AddValue = 0.027 + 0.004 * BonusAmount;
		break;
	default:
		break;
	}

	EquipmentBonusData.StackCount++;
	EquipmentBonusData.Value += AddValue;
}
```

</details>
<br />

## 강화
![Image](https://github.com/user-attachments/assets/8c7da264-dd2a-42ec-8671-5dec3526f398)<br />

### 캐릭터 레벨업
경험치 책을 이용해 레벨업 한다. 경험치 책은 그 종류마다 서로 다른 경험치 양을 가진다.<br />
각 레벨당 필요한 골드와 경험치는 커브드 테이블로 저장된다.<br />

### 성유물 장착
5종류의 부위에 각각 성유물을 장착할 수 있다.<br />
성유물을 장착하면 그 성유물에 부여된 스탯보너스를 얻고,<br />
같은 종유의 성유물을 2개, 4개 장착할 경우 특수한 보너스를 얻게 된다.<br />
<br />
예를 들어 영상 속 검투사의 피날레의 경우 2셋은 공격력 +18%, 4셋은 일반공격시 35% 데미지 부스트를 얻게된다.<br />
이는 장착 시 스텟 보너스와 세트 효과는 GameplayEffect로 구현한다.<br />

<details>
<summary>EquipArtifact</summary>
	
```
void UMyEquipmentComponent::EquipArtifact(UArtifact* InArtifact)
{
	if (!InArtifact) return;

	if (AMyPlayableCharacter* Other = InArtifact->GetOwner())
	{
		Other->GetEquipmentComponent()->UnequipArtifact(InArtifact);
	}

	UArtifact* lastArtifact = nullptr;
	int ArtifactTypeToInt = (int)InArtifact->GetArtifactType();
	lastArtifact = ArtifactList[ArtifactTypeToInt].Get();
	if (lastArtifact)
	{
		UnequipArtifact(lastArtifact);
	}

	AMyPlayableCharacter* OwnerCharacter = Cast<AMyPlayableCharacter>(GetOwner());
	InArtifact->SetOwner(OwnerCharacter);
	ArtifactList[ArtifactTypeToInt] = InArtifact;

	int ArtifactSetCount = GetArtifactSetCount(InArtifact);
	if (ArtifactSetCount == 2)
	{
		InArtifact->ApplyTwoPieceBouns();
	}
	else if (ArtifactSetCount == 4)
	{
		InArtifact->ApplyFourPieceBouns();
	}

	UAbilitySystemComponent* AbilitySystemComponent = OwnerCharacter->GetAbilitySystemComponent();
	if (!AbilitySystemComponent) return;

	if (ArtifactEquipmentEffectHandle.IsValid())
	{
		AbilitySystemComponent->RemoveActiveGameplayEffect(ArtifactEquipmentEffectHandle);
	}

#pragma region EquipmentEffect
	FGameplayEffectContextHandle ContextHandle = AbilitySystemComponent->MakeEffectContext();
	ContextHandle.AddSourceObject(OwnerCharacter);

	FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(ArtifactEquipmentEffect, 1.0f, ContextHandle);

	if (SpecHandle.IsValid())
	{
		FStatBonusSpec StatBonusSpec = GetArtifactsStatBonusSpec();

		SpecHandle.Data->SetSetByCallerMagnitude(MyGameplayTags::SetByCaller_Equipment_MaxHealth, StatBonusSpec.Health);
		SpecHandle.Data->SetSetByCallerMagnitude(MyGameplayTags::SetByCaller_Equipment_MaxHealthRatio, StatBonusSpec.HealthRatio);
		SpecHandle.Data->SetSetByCallerMagnitude(MyGameplayTags::SetByCaller_Equipment_AttackPower, StatBonusSpec.AttackPower);
		SpecHandle.Data->SetSetByCallerMagnitude(MyGameplayTags::SetByCaller_Equipment_AttackPowerRatio, StatBonusSpec.AttackPowerRatio);
		SpecHandle.Data->SetSetByCallerMagnitude(MyGameplayTags::SetByCaller_Equipment_DefensePower, StatBonusSpec.DefensePower);
		SpecHandle.Data->SetSetByCallerMagnitude(MyGameplayTags::SetByCaller_Equipment_DefensePowerRatio, StatBonusSpec.DefensePowerRatio);
		SpecHandle.Data->SetSetByCallerMagnitude(MyGameplayTags::SetByCaller_Equipment_CriticalDamage, StatBonusSpec.CriticalDamage);
		SpecHandle.Data->SetSetByCallerMagnitude(MyGameplayTags::SetByCaller_Equipment_CriticalChance, StatBonusSpec.CriticalChance);

		ArtifactEquipmentEffectHandle = AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
	}
#pragma endregion
}
```

</details>
<br />
장착 시 스텟 보너스는 SetByCaller를 이용해 스탯보너스를 주고,<br />
세트 효과의 경우, 그 종류에 따라 조금 다른데, 일반공격 시 35% 데미지 부스트는 일반 공격 시 부여되는 테그를 Reguire Tags to Apply This Effect에 넣어서 일반공격시에만 적용되도록 하였다.<br />

## 그 외
### 세이브 로드
SaveGames을 이용해 간단하게 세이브 로드를 구현했습니다.<br />
게임을 종료할 때 세이브 되도록 했습니다.<br /><br />

### 보물 상자
![Image](https://github.com/user-attachments/assets/f38ef7ec-0d30-42cc-a092-62f6be0631e4)<br />
IInteractable 인터페이스를 이용해 상호작용 해 위에서 설명한 Reward를 통해 보상을 받도록 했다.<br />
물론 SaveGames를 통해 이미 얻은 상자는 보이지 않도록 했다.

### 경치 카메라
![Image](https://github.com/user-attachments/assets/6269f8a7-5432-49db-9243-e0cdfab01cd1)

### 카메라와 겹칠 시 반투명화
![Image](https://github.com/user-attachments/assets/a560dd2f-de7f-4c53-93d9-4d440926798b)<br />
카메라부터 대상까지 벡터와 카메라부터 캐릭터까지 벡터의 내적을 이용해해 그 값을 디더링해서 반투명화.<br />
몬스터나 건물에 적용해 캐릭터가 가려지지 않도록 했다.<br />
<img width="1484" height="533" alt="Image" src="https://github.com/user-attachments/assets/73a0033b-dce6-4748-b4f7-8df419eaa552" /><br />
