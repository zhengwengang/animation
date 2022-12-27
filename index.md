# 《元素动效》前端文档

### 1. 页面总时长与元素的入场延时、入场时长、进出场分界时长、出场延时、出场时长的关系

1. 页面总时长计算规则
> 1. 当前页面有视频元素时: 当前页面视频(裁剪后)时长最长的时间
> 2. 当前页面没有视频元素时: 取页面设置的时长: pageJson.pageAnimationDuration 范围 0.1s ~ 30s，
> 3. 默认为 5s

```javascript
get pageDuration() {
    // 当前页面的json数据
    const pageJson = getPageJson(getFocusPageIndex());
    // 元素
    const elems = pageJson.elems
    const totalDuration = elems.reduce((maxDuration, elem)=>{
      // 是视频元素 获取视频的时长
      if(elem['data-type'] === 'video') {
        const dataElem = elem['data-elem'];
        // 考虑视频裁剪的情况
        if(dataElem['clip-start'] && dataElem['clip-end']){
          // 单位毫秒
          const clipStart = Number(dataElem['clip-start']);
          const clipEnd = Number(dataElem['clip-end']);
          const clipDuration = (clipEnd - clipStart)/1000;

          return clipDuration > maxDuration ? clipDuration : maxDuration
        }
        // 视频时长
        const videoDuration = elem.duration;
        return videoDuration && videoDuration > maxDuration ? videoDuration : maxDuration
      }
      return maxDuration
    }, 0)

    return totalDuration || pageJson.pageAnimationDuration ||  5;
  }
  ```

2. 各阶段时长分布规则

> $n_{numberOfInAnimationElement}$:  参与入场动画的元素个数
> $n_{numberOfOutAnimationElement}$: 参与出场动画的元素个数

| 总时长T | 入场延时$t_1$<br/>$f_{t1}$(T)                            | 入场时长$t_2$<br/>$f_{t2}$(T)       | 进出场分界时长$t_3$<br/>$f_{t3}$(T)| 出场延时$t_4$<br/>$f_{t4}$(T)                               | 出场时长$t_5$<br/>$f_{t5}$(T) |
|:------:| :------------------------------------------------------:|:---------------------------------:|:--------------------------------:|:----------------------------------------------------------:|:----------------------------:|
| T >= 4 | max(0, min(1, (numberOfInAnimationElement - 1) * 0.2))  | T - $t_1$ - $t_3$ - $t_4$ - $t_5$ | 0.5                              | max(0, min(0.5, (numberOfOutAnimationElement - 1) * 0.15)) | 1.5 - $t_4$                  |
| T < 4  | (T/4) * $f_{t1}$(4)                                    | (T/4)  * $f_{t2}$(4)               |(T/4)  * $f_{t3}$(4)              | (T/4)  * $f_{t4}$(4)                                       | (T/4)  * $f_{t5}$(4)     |

> T: 页面总时长
> t1: 入场延时时间
> t2: 入场时长
> t3: 进出场分界时长
> t4: 出场延时
> t5: 出场时长

> 1. T>=4s时
     t1最大为1s，最小为0s 入素出场延时间隔默认为2s
     t1 = max(0, min(1, (numberOfInAnimationElement - 1) * 0.2))
     t3为0.5s 
     t4最大为0.5s 最小为0s 元素出场延时间隔默认0.15s 
     t4 = max(0, min(0.5, (numberOfOutAnimationElement - 1) * 0.15))
     t4+t5为1.5s
     t5 = 1.5- t4
     t2= T - t1 - t3 - t4 -t5

> 2. T < 4s时
     t1 t3 t4 t5 为T>=4s时的值按照T/4的比例进行缩小
     t1 = (T/4) * t1 ...
     t2= T - t1 - t3 - t4 -t5 

```javascript
getAllAnimationStageTimeLine(numberOfInAnimationElement, numberOfOutAnimationElement){
  // 入场延迟最大为1s，最小为0s 入素出场延时间隔默认为2s
    let inDelayT = max(0, min(1, (numberOfInAnimationElement - 1) * 0.2))
    // 出入场间隔
    let intervalT = 0.5
    // 出场延迟最大为0.5s，最小为0s 入素出场延时间隔默认为0.15s
    let outDelayT = max(0, min(0.5, (numberOfOutAnimationElement - 1) * 0.15))
    // 出场延迟 + 出场时长 = 1.5s 则 出场时长 = 1.5-出场延迟
    let outT = 1.5 - outDelayT
    // 总时长T小于4s的情况
    const timeRatio = min(1, pageDuration / 4)
    // 按照pageDuration / 4等比例缩小
    ;[inDelayT, intervalT, outDelayT, outT] = [inDelayT, intervalT, outDelayT, outT].map(value => value * timeRatio)
    // 计算入场时长
    const inT = [inDelayT, intervalT, outDelayT, outT].reduce((acc, item) => acc - item, pageDuration)
    // 取入场元素与出场元素的最大值 作为返回结果数组的长度
    const length = max(numberOfInAnimationElement, numberOfOutAnimationElement)
    // 根据总的入场延迟时长以及入场元素计算 每个元素入场延迟间隔时长
    const inGap = inDelayT / max(1, (numberOfInAnimationElement - 1))
    // 根据总的出场延迟时长以及出场元素计算 每个元素出场延迟间隔时长
    const outGap = outDelayT / max(1, (numberOfOutAnimationElement - 1))
    // 得到结果数据
    const animationStageTimeLines = Array.from({ length }, (_, index) => [index * inGap, inT, intervalT, index * outGap, outT])
    return animationStageTimeLines

    // 返回结果数据example:
    // 根据页面时长和动效元素个数, 返回这种数据结构
    // [[入场延时时长, 入场时长, 出入场间隔, 出场延时时长, 出场时长]]
    // const animationStageTimeLines = [
    //   [0.0, 15, 0.5, 0.0, 1.5],  
    //   [0.2, 15, 0.5, 0.2, 1.5],
    //   [0.4, 15, 0.5, 0.4, 1.5],
    //   [0.6, 15, 0.5, 0.6, 1.5],
    //   [0.8, 15, 0.5, 0.8, 1.5],
    //   [1.0, 15, 0.5, 1.0, 1.5],
    // ]
}
```