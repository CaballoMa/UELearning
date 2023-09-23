# IKRetargetProcessor
Retarget就是根据source和target两套骨骼存储的骨骼信息（index，name等）进行转换，转换过程是从root到end遍历每个骨骼并对其位置和角度进行调整（parent * self）

## Skeleton
FRetargetSkeleton主要存储骨骼的相关信息和姿势数据的结构体。
### FRetargetSkeleton 存储骨骼的相关信息和姿势数据的结构体。
* ChainThatContainsBone：记录每个骨骼实际上由哪个链控制的数组。
* Function: 
一些Global/Local的Pose的更新计算，Child Bone的获取，Bone之间Parents的判断。

### FTargetSkeleton：FRetargetSkeleton
在FRetargetSkeleton的基础上添加了输出全局姿势和骨骼是否重新定位的信息。
* OutputGlobalPose：输出的全局Pose数组。
* IsBoneRetargeted：记录骨骼是否重新定位的数组。
* Functions：
设置骨骼为IsRetargeted；
将所有骨骼转到全局坐标；

### FResolvedBoneChain
将骨骼链解析为实际骨骼：验证起始骨骼和结束骨骼之间的关系。如果bEndIsStartOrChildOfStart为true则OutBoneIndices 包含正确顺序的骨骼索引。
* bFoundStartBone：起始骨骼是否存在。
* bFoundEndBone：结束骨骼是否存在。
* bEndIsStartOrChildOfStart：结束骨骼是否等于起始骨骼或者是起始骨骼的子骨骼。
* IsValid()：检查解析的骨骼链是否有效。

## Root
### FRootSource 源骨骼的根骨骼信息
* InitialHeightInverse：初始高度的倒数。

### FRootTarget 目标骨骼的根骨骼信息
* RootTranslationDelta：根骨骼平移的增量。
* RootRotationDelta：根骨骼旋转的增量。

### FRootRetargeter 根骨骼的重定位器
* GetGlobalScaleVector()：获取全局缩放向量的函数。

### FPoleVectorMatcher 极点矢量匹配器
* TargetToSourceAngularOffsetAtRefPose：参考姿势下目标极点与源极点的角度偏移量。
* AllChildrenWithinChain：链内的所有子骨骼的索引。

## FKChain
### FChainFK: FK链
* FillTransformsWithLocalSpaceOfChain()：在全局骨骼姿势中填充与链中每个骨骼的本地空间姿势相对应的转换矩阵.
* PutCurrentTransformsInRefPose()：将当前的骨骼链的转换矩阵设置为参考姿势。

### FChainEncoderFK：FChainFK FK链的编码器
* ChainParentCurrentGlobalTransform：链的父骨骼的当前全局变换。
* TransformCurrentChainTransforms()：根据新的父骨骼转换矩阵，转换当前骨骼链的转换矩阵。

### hainDecoderFK：FChainFK FK链的解码器
* InitializeIntermediateParentIndices()：初始化中间父骨骼的索引。
* MatchPoleVector()：让Target匹配Source的极轴点矢量。

## IKChain
### FDecodedIKChain 解码后的IK链的信息
* EndEffectorPosition：末端效应器位置。
* EndEffectorRotation：末端效应器旋转。
* PoleVectorPosition：极点矢量位置。

### FSourceChainIK 源IK链的信息
* PreviousEndPosition：上一帧的末端位置。
* CurrentEndDirectionNormalized：当前末端方向的归一化向量。
* CurrentHeightFromGroundNormalized：当前高度相对于地面的归一化值。

### FTargetChainIK 目标IK链的信息
* PrevEndPosition：上一帧的末端位置。
ChainPair
### FRetargetChainPair
骨骼链对的基类，用于存储源链和目标链的相关信息
* Settings：目标链设置的结构体。

### FRetargetChainPairFK：FRetargetChainPair 使用FK进行骨骼链重定位的骨骼链对
* PoleVectorMatcher：极点矢量匹配器的结构体。

### FRetargetChainPairIK：FRetargetChainPair 使用IK Rig进行骨骼链重定位的骨骼链对
* PoleVectorGoalName：极点矢量目标。

## UIKRetargetProcessor
### Class UIKRetargetProcessor：
实现骨骼动画的IK重定位功能，用于初始化和运行重定位过程，包括根骨骼的处理、FK链和IK链的处理，以及与目标骨骼网格的交互等。
* Initialize()：初始化 IK 重定位处理器，设置源骨骼网格、目标骨骼网格和重定位器资产。接受源骨骼网格、目标骨骼网格、重定位器资产和是否忽略警告作为参数。
* RunRetargeter(): 运行重定位器，生成新的姿势。接受源骨骼的全局姿势、速度曲线的速度值和时间增量作为参数，并返回目标骨骼的全局姿势。
* RunStrideWarping()：在基本 IK 重定位之后根据实际地形障碍物等情况来进一步调整IK。
* ApplyNewRetargetPose()：应用新的重定向姿势到UIKRetargetProcessor中:
    1. re-generate the retarget pose
    RetargetSkeleton.GenerateRetargetPose(NewRetargetPoseName, NewRetargetPose, RootBoneName);
    
    2. re-initialize the bone chains
    FKChainPair.Initialize(*SourceBoneChain, *TargetBoneChain, SourceSkeleton, TargetSkeleton, Log);
    
    3. re-initialize the root
    RootRetargeter.InitializeSource(RootRetargeter.Source.BoneName, SourceSkeleton, Log);

# IK Rig Solver
## Limb Solver
        //一：骨骼数量为3
        //Length(RootLocation, GoalLocation) >= LimbLength:
        //Just go in a straight line
        InOutLinks[Index].Location = InOutLinks[Index-1].Location + Direction * InOutLinks[Index].Length;
        
        //Length(RootLocation, GoalLocation) < LimbLength:
        //Two Bone Limb IK
        //先求出HipToFoot所在平面上Knee的位置，再根据BendDir来调整膝盖的偏移方向
        //pC: Hip / Root
        //C: 膝盖弯曲角度
        //BendDir/Dir_LegLineToKnee: 膝盖的弯曲方向
        
        const FVector NewProjectedKneeLoc = pC + HipToFootDir * a * CosC;
        const FVector NewKneeLoc = NewProjectedKneeLoc + Dir_LegLineToKnee * a * FMath::Sin(C);
        
        //二：骨骼数量大于3
        //SolveFABRIK
        //一：计算ChildAxisX、ChildAxisY 和 LinkAxisZ用于后续的旋转计算
        const FVector ChildAxisX = (ChildLink.Location - CurrentLink.Location).GetSafeNormal();
        const FVector ChildAxisY = FVector::CrossProduct(PlaneNormal, ChildAxisX);
        const FVector ParentAxisX = (ParentLink.Location - CurrentLink.Location).GetSafeNormal();
        // Orient Z, so that ChildAxisY points 'up' and produces positive Sin values.
        CurrentLink.LinkAxisZ = FVector::DotProduct(ParentAxisX, ChildAxisY) > 0.f ? PlaneNormal : -PlaneNormal;
        
        //二：根据PullDistribution调整IK拉伸效果
        //PullDistribution是一个0到1之间的标量值，表示IK链在求解过程中拉伸力的分布比例。当PullDistribution为0时，拉力均匀分布在IK链上的每个关节，而当PullDistribution为1时，拉力集中在链的末端。
        const FVector PullDistributionOffset = PullDistribution * (EndToTarget) + (1.f - PullDistribution) * (RootToTarget);for (int32 Index = 0; Index < NumLinks; Index++)
        {
            InOutLinks[Index].Location += PullDistributionOffset;
        }
        //三：循环ForwardReach & BackwardReach优化IK
        //Plus：根据bNeedPullAveraging的值来判断是否进行拉力平均化 (如果为true，在每次迭代中，拉力会根据关节的连接关系和距离进行重新分配，以确保拉力在整个IK链上的均匀分布)
        
        TArray<FLimbLink> ForwardPull = InOutLinks;
        TArray<FLimbLink> BackwardPull = InOutLinks;
        
        FABRIK_ForwardReach(ForwardPull, InTargetLocation, ReachStepAlpha, bUseAngleLimit, MinRotationAngleRadians);
        FABRIK_BackwardReach(BackwardPull,RootTargetLocation, ReachStepAlpha, bUseAngleLimit, MinRotationAngleRadians);
        
        // Average pulls
        for (int32 LinkIndex = 0; LinkIndex < NumLinks; LinkIndex++)
        {
            InOutLinks[LinkIndex].Location = 0.5f * (ForwardPull[LinkIndex].Location +  BackwardPull[LinkIndex].Location);
        }
        
        //四：root位置矫正
        if (!InOutLinks[0].Location.Equals(RootTargetLocation))
        {
            FABRIK_BackwardReach(InOutLinks, RootTargetLocation, ReachStepAlpha, bUseAngleLimit, MinRotationAngleRadians);
        }
        
* FABRIK算法References：

    [FABRIK原理和实现](https://zhuanlan.zhihu.com/p/508496762)
    
    [Videos for FABRIK & FABRIK Extension](http://andreasaristidou.com/FABRIK.html)

## Pole Solver

            //求出两个plane之间的旋转角度来进行Translate和Rotate
            const FVector RootToEnd = (EndLocation - RootLocation).GetSafeNormal();
            const FVector RootToKnee = (KneeLocation - RootLocation).GetSafeNormal();
            const FVector RootToPole = (GoalLocation - RootLocation).GetSafeNormal();
            const FVector InitPlane = FVector::CrossProduct(RootToEnd, RootToKnee).GetSafeNormal();
            const FVector TargetPlane = FVector::CrossProduct(RootToEnd, RootToPole).GetSafeNormal();
            const FQuat DeltaRotation = FQuat::FindBetweenNormals(InitPlane, TargetPlane);
            const FQuat TargetRotation = FMath::Lerp(BoneRotation, DeltaRotation * BoneRotation, Effector->Alpha);
            const FVector TargetTranslation = FMath::Lerp( BoneTranslation, RootLocation + DeltaRotation.RotateVector(BoneTranslation - RootLocation), Effector->Alpha);
            Set Transform Solver
            //根据Alpha来对Transform进行简单的Lerp
            if (Effector->bEnablePosition)
            {
                  const FVector TargetPosition = FMath::Lerp(CurrentTransform.GetTranslation(), 
                                                 InGoal->FinalBlendedPosition, Effector->Alpha);
                  CurrentTransform.SetLocation(TargetPosition);
            }
            
            if (Effector->bEnableRotation)
            {
                  const FQuat TargetRotation = FMath::Lerp(CurrentTransform.GetRotation(), 
                                               InGoal->FinalBlendedRotation, Effector->Alpha);
                  CurrentTransform.SetRotation(TargetRotation);
            }
            
## Body Mover Solver

            //一：通过 InitialPoints 和 CurrentPoints 数组计算旋转偏移（RotationOffset），并返回旋转偏移后的初始质心（InitialCentroid）和当前质心（CurrentCentroid）。
            InitialPoints.Add(InitialEffector.GetTranslation());
            CurrentPoints.Add(Goal->FinalBlendedPosition);
            const FQuat RotationOffset = GetRotationFromDeformedPoints(InitialPoints,C urrentPoints,In itialCentroid,Cur rentCentroid);
            
            //二：根据当前质心和初始质心计算位置偏移，并根据偏移的方向确定权重，用于决定正向或负向的位置偏移，然后更新 BodyBoneIndex 对应骨骼的位置
            const FVector Offset = (CurrentCentroid - InitialCentroid);
            const FVector Weight(
                Offset.X > 0.f ? PositionPositiveX : PositionNegativeX,
                Offset.Y > 0.f ? PositionPositiveY : PositionNegativeY,
                Offset.Z > 0.f ? PositionPositiveZ : PositionNegativeZ);
            CurrentBodyTransform.AddToTranslation(Offset * (Weight*PositionAlpha));
            
            //三：执行每个轴向的旋转偏移：通过将 RotationOffset乘以对应轴向的旋转系数，计算出每个轴向的Euler角，并将其转换为四元数。然后使用RotationAlpha对整体旋转偏移进行混合。
            FVector Euler = RotationOffset.Euler() * FVector(RotateXAlpha, RotateYAlpha, RotateZAlpha);
            FQuat FinalRotationOffset = FQuat::MakeFromEuler(Euler);
            FinalRotationOffset = FQuat::FastLerp(FQuat::Identity, FinalRotationOffset, RotationAlpha).GetNormalized();
            CurrentBodyTransform.SetRotation(FinalRotationOffset * CurrentBodyTransform.GetRotation());

## Full Body Solver

            //Effector Goal Setting
            Solver.SetEffectorGoal(
                Effector->IndexInSolver,
                Goal->FinalBlendedPosition,
                Goal->FinalBlendedRotation,
                Settings);
            
            //Bones Setting
            FPBIKSolver.Solve(Settings);
            //一：准备工作
            //1. Update骨骼Transform信息
            //2. 根据骨骼Transform信息通过计算质心来调整Body的Transform
            //3. 更新PinPoint的Transform信息
            //4. 更新Effector对象的位置、旋转和距离属性
            
            //二：进行所有解算器的计算:
            UpdateBodies(Settings); 
            
            1. ApplyRootPrePull(Settings.RootBehavior, Settings.PrePullRootSettings);
            //根据质心算出Body的位置和旋转偏移，通过融合用户的PrePullSettings来得出Body位置和旋转的偏移
            
            2. ApplyPreferredAngles();
            //根据骨骼的压缩百分比来对骨骼的旋转（Body->Rotation）进行部分旋转设置（PartialRotation）
            
            3. ApplyPullChainAlpha(Settings.GlobalPullChainAlpha);
            //先得出chain的旋转和位置变化
            //根据PullChainAlpha对一系列效应器（Effectors）和它们相连的骨骼节点进行位置和旋转的调整
            //骨骼的Position还根据效应器与父子根节点之间骨骼链的长度来进一步调整，通过通过将骨骼链上每个骨骼与效应器的距离与整个骨骼链的长度进行比例计算，可以得到一个介于0和1之间的拉伸强度值，用来乘以之前根据PullChainAlpha求出的ChainDeltaPosition
            
            4. SolveConstraints(Settings);
            //根据迭代数来进行Constraint的迭代计算：Joint Constraint + Pin Constraint:
                
            //a)根据XPBD来跟新Rotation：
            //Equation 8 in "Detailed Rigid Body Simulation with XPBD"
            //UE5对公式有所修改和简化
            Omega = InvMass * (1.0f - J.RotationStiffness) * FVector::CrossProduct(Offset, Push);
            ApplyRotationDelta(Omega);
            
            //b)根据joint limits的设置来调整旋转
            UpdateJointLimits();
            
            //c)根据XPBD来更新Position
            //Equation 6 in "Detailed Rigid Body Simulation with XPBD"
            //UE5对公式有所修改和简化
            //OverRelaxation = 1.3 (在1-2之间，值越大收敛越快，稳定性更差)
            Position += (N * (C * (WA / W))) * (1.0f - J.PositionStiffness) * SolverSettings->OverRelaxation;
            
            //三：根据解算器结果更新骨骼Transform信息
            UpdateBonesFromBodies();