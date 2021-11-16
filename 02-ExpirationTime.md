# ExpirationTime

在创建更新的时候有一个 expirationTime 变量
在 createUpdate 创建 update 对象传入 expirationTime 变量，
```
const classComponentUpdater = {
  isMounted,
  enqueueSetState(inst, payload, callback) {
    const fiber = getInstance(inst);
    const currentTime = requestCurrentTimeForUpdate();
    const suspenseConfig = requestCurrentSuspenseConfig();
    /* 触发state更新时 ，产生优先级*/
    const expirationTime = computeExpirationForFiber(
      currentTime,
      fiber,
      suspenseConfig,
    );

    const update = createUpdate(expirationTime, suspenseConfig);
    
};
```
__computeExpirationForFiber__ 函数接收两个参数，currentTime，fiber，通过计算返回 expirationTime ，

```
export const Sync = MAX_SIGNED_31_BIT_INT;
export const Batched = Sync - 1;

const UNIT_SIZE = 10;
const MAGIC_NUMBER_OFFSET = Batched - 1;

function computeExpirationBucket(
  currentTime,
  expirationInMs,
  bucketSizeMs,
): ExpirationTime {
  return (
    MAGIC_NUMBER_OFFSET -
    ceiling(
      MAGIC_NUMBER_OFFSET - currentTime + expirationInMs / UNIT_SIZE,
      bucketSizeMs / UNIT_SIZE,
    )
  );
}
```
* 根据优先级的高低，会获取不一样的常量，当前 fiber 标记优先级的那个属性，代表优先级越高，常量的值就越小，那么导致计算出来的expirationTime 数值也就越小, React 就会优先调度这个相应的更新。
* React 让两个相近的 update（25ms）得到相同的 expirationTime ，目的就是为了合并这两个 update，从而达到批量更新的目的 
* 在 React 中，为防止某个 update 因为优先级的原因一直被打断而未能执行，React 就会设置一个 expirationTime ，当时间到了expirationTime 的时候，如果某个 update 还未执行，React 将强制执行这个 update，
