# Animation

### BlendSpace

Blend Spaces是由若干个Anim Sequences构成,所以当所有的Anim Sequences都为叠加动画时,即可输出叠加型Pose。

### Montage

* 一个Montage可以设置若干个Slot，具体哪个slot生效，由运行时刻AnimGraph的情况决定置每个Slot中可以拖入若干个AnimSequence,顺序可以按需更改。如果AnimSequence为叠加型动画,则这个Montage也为叠加型MontageMontage可以有若干个Section, Section把整个Montage拆分成若干块，这些块之间可以自由的衔接和跳转。

* Montage的属性页可以设置此Montage在播放时的淡入和淡出融合,使动作不会显得很突兀如果Montage所使用的AnimSequence勾选了RootMotion,则这个动画在播放过程中根骨骼的位移会使角色产生移动。这个功能要与CharacterMovement组件配合。移动组件在更新速度时会优先使用根骨骼位移来作为移动的值。注意:根骨动画需要客户端和服务器同时播,如果服务器不播动画或者时间差太多,客户端会被拉扯。

### Inertialization惯性插值

##### StandBlend
* Evaluate both SourcePose and TargetPose

##### Inertialization

* Evaluate only TargetPose

### FootIK

根据双脚向下发出的射线求offset，来判断角色root的位置：
```
if (LeftFootOffset> RightFootOffset)
{
    MeshOffset= RightFootOffset;
    LeftFootOffset = LeftFootOffset-RightFootOffset;
    RightFootOffset= 0.f;
}
```
