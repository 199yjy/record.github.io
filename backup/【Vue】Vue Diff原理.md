## 一、Diff算法基础概念

### 1.1 什么是Diff算法
Diff算法是计算两个虚拟DOM树差异的算法，用于最小化DOM操作，提升渲染性能。

### 1.2 Diff算法复杂度
- 传统树Diff：O(n³)
- Vue优化后：O(n)

## 二、Vue 2 Diff算法原理

### 2.1 核心源码位置
```js
// src/core/vdom/patch.js
function updateChildren(parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
  // Diff核心算法
}
```

### 2.2 双端比较算法

Vue 2采用经典的**双端比较算法**，通过四个指针进行比对：

```js
function updateChildren(parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
  let oldStartIdx = 0
  let newStartIdx = 0
  let oldEndIdx = oldCh.length - 1
  let oldStartVnode = oldCh[0]
  let oldEndVnode = oldCh[oldEndIdx]
  let newEndIdx = newCh.length - 1
  let newStartVnode = newCh[0]
  let newEndVnode = newCh[newEndIdx]
  let oldKeyToIdx, idxInOld, vnodeToMove, refElm

  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    if (isUndef(oldStartVnode)) {
      // 情况1：旧开始节点为空
      oldStartVnode = oldCh[++oldStartIdx]
    } else if (isUndef(oldEndVnode)) {
      // 情况2：旧结束节点为空
      oldEndVnode = oldCh[--oldEndIdx]
    } else if (sameVnode(oldStartVnode, newStartVnode)) {
      // 情况3：头头相同
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
      oldStartVnode = oldCh[++oldStartIdx]
      newStartVnode = newCh[++newStartIdx]
    } else if (sameVnode(oldEndVnode, newEndVnode)) {
      // 情况4：尾尾相同
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
      oldEndVnode = oldCh[--oldEndIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldStartVnode, newEndVnode)) {
      // 情况5：头尾相同
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
      canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
      oldStartVnode = oldCh[++oldStartIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldEndVnode, newStartVnode)) {
      // 情况6：尾头相同
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
      canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
      oldEndVnode = oldCh[--oldEndIdx]
      newStartVnode = newCh[++newStartIdx]
    } else {
      // 情况7：四种情况都不匹配，使用key映射
      if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
      idxInOld = isDef(newStartVnode.key)
        ? oldKeyToIdx[newStartVnode.key]
        : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
      if (isUndef(idxInOld)) {
        // 新节点，创建
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
      } else {
        // 移动节点
        vnodeToMove = oldCh[idxInOld]
        if (sameVnode(vnodeToMove, newStartVnode)) {
          patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
          oldCh[idxInOld] = undefined
          canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
        } else {
          // key相同但元素不同，创建新元素
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        }
      }
      newStartVnode = newCh[++newStartIdx]
    }
  }

  // 处理剩余节点
  if (oldStartIdx > oldEndIdx) {
    // 添加新节点
    refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
    addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
  } else if (newStartIdx > newEndIdx) {
    // 移除旧节点
    removeVnodes(oldCh, oldStartIdx, oldEndIdx)
  }
}
```

### 2.3 双端比较的四种命中情况

#### 情况1：头头相同
```
旧: A B C D
新: A B E F
处理: 移动指针，不移动DOM
```

#### 情况2：尾尾相同
```
旧: A B C D  
新: E F C D
处理: 移动指针，不移动DOM
```

#### 情况3：头尾相同
```
旧: A B C D
新: D C B A
处理: 将A移动到D后面
```

#### 情况4：尾头相同
```
旧: A B C D
新: D A B C  
处理: 将D移动到A前面
```

### 2.4 相同节点判断

```js
function sameVnode(a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```

## 三、Vue 3 Diff算法原理

### 3.1 核心源码位置
```ts
// packages/runtime-core/src/renderer.ts
const patchKeyedChildren = (
  c1: VNode[],
  c2: VNode[],
  container: RendererElement,
  parentAnchor: RendererNode | null,
  parentComponent: ComponentInternalInstance | null,
  parentSuspense: SuspenseBoundary | null,
  isSVG: boolean,
  slotScopeIds: string[] | null,
  optimized: boolean
) => {
  // Vue 3 的Diff算法
}
```

### 3.2 快速Diff算法

Vue 3采用**快速Diff算法**，分为三个步骤：

```ts
const patchKeyedChildren = (
  c1: VNode[], // 旧子节点
  c2: VNode[], // 新子节点  
  // ...其他参数
) => {
  let i = 0
  const l2 = c2.length
  let e1 = c1.length - 1 // 旧子节点的结束索引
  let e2 = l2 - 1        // 新子节点的结束索引

  // 1. 从前向后同步
  while (i <= e1 && i <= e2) {
    const n1 = c1[i]
    const n2 = c2[i]
    if (isSameVNodeType(n1, n2)) {
      patch(n1, n2, container, null, parentComponent, parentSuspense, isSVG, slotScopeIds, optimized)
    } else {
      break
    }
    i++
  }

  // 2. 从后向前同步  
  while (i <= e1 && i <= e2) {
    const n1 = c1[e1]
    const n2 = c2[e2]
    if (isSameVNodeType(n1, n2)) {
      patch(n1, n2, container, null, parentComponent, parentSuspense, isSVG, slotScopeIds, optimized)
    } else {
      break
    }
    e1--
    e2--
  }

  // 3. 处理剩余情况
  if (i > e1) {
    // 新增节点
    if (i <= e2) {
      const nextPos = e2 + 1
      const anchor = nextPos < l2 ? c2[nextPos].el : parentAnchor
      while (i <= e2) {
        patch(null, c2[i], container, anchor, parentComponent, parentSuspense, isSVG, slotScopeIds, optimized)
        i++
      }
    }
  } else if (i > e2) {
    // 删除节点
    while (i <= e1) {
      unmount(c1[i], parentComponent, parentSuspense, true)
      i++
    }
  } else {
    // 乱序部分，需要移动和patch
    const s1 = i // 旧子节点的开始索引
    const s2 = i // 新子节点的开始索引

    // 3.1 构建key到索引的映射
    const keyToNewIndexMap: Map<string | number | symbol, number> = new Map()
    for (i = s2; i <= e2; i++) {
      const nextChild = c2[i]
      if (nextChild.key != null) {
        keyToNewIndexMap.set(nextChild.key, i)
      }
    }

    // 3.2 找出需要移动的节点
    let j
    let patched = 0
    const toBePatched = e2 - s2 + 1
    let moved = false
    let maxNewIndexSoFar = 0

    const newIndexToOldIndexMap = new Array(toBePatched)
    for (i = 0; i < toBePatched; i++) newIndexToOldIndexMap[i] = 0

    for (i = s1; i <= e1; i++) {
      const prevChild = c1[i]
      if (patched >= toBePatched) {
        // 所有新节点都已patch，剩余的卸载
        unmount(prevChild, parentComponent, parentSuspense, true)
        continue
      }
      let newIndex
      if (prevChild.key != null) {
        newIndex = keyToNewIndexMap.get(prevChild.key)
      } else {
        // 没有key的情况，需要遍历查找
        for (j = s2; j <= e2; j++) {
          if (newIndexToOldIndexMap[j - s2] === 0 && isSameVNodeType(prevChild, c2[j])) {
            newIndex = j
            break
          }
        }
      }
      if (newIndex === undefined) {
        unmount(prevChild, parentComponent, parentSuspense, true)
      } else {
        newIndexToOldIndexMap[newIndex - s2] = i + 1
        if (newIndex >= maxNewIndexSoFar) {
          maxNewIndexSoFar = newIndex
        } else {
          moved = true
        }
        patch(prevChild, c2[newIndex], container, null, parentComponent, parentSuspense, isSVG, slotScopeIds, optimized)
        patched++
      }
    }

    // 3.3 移动和挂载节点
    const increasingNewIndexSequence = moved
      ? getSequence(newIndexToOldIndexMap)
      : EMPTY_ARR
    j = increasingNewIndexSequence.length - 1
    for (i = toBePatched - 1; i >= 0; i--) {
      const nextIndex = s2 + i
      const nextChild = c2[nextIndex]
      const anchor = nextIndex + 1 < l2 ? c2[nextIndex + 1].el : parentAnchor
      if (newIndexToOldIndexMap[i] === 0) {
        // 挂载新节点
        patch(null, nextChild, container, anchor, parentComponent, parentSuspense, isSVG, slotScopeIds, optimized)
      } else if (moved) {
        // 移动节点
        if (j < 0 || i !== increasingNewIndexSequence[j]) {
          move(nextChild, container, anchor, MoveType.REORDER)
        } else {
          j--
        }
      }
    }
  }
}
```

### 3.3 最长递增子序列算法

Vue 3使用**最长递增子序列**算法来最小化移动操作：

```ts
function getSequence(arr: number[]): number[] {
  const p = arr.slice()
  const result = [0]
  let i, j, u, v, c
  const len = arr.length
  for (i = 0; i < len; i++) {
    const arrI = arr[i]
    if (arrI !== 0) {
      j = result[result.length - 1]
      if (arr[j] < arrI) {
        p[i] = j
        result.push(i)
        continue
      }
      u = 0
      v = result.length - 1
      while (u < v) {
        c = (u + v) >> 1
        if (arr[result[c]] < arrI) {
          u = c + 1
        } else {
          v = c
        }
      }
      if (arrI < arr[result[u]]) {
        if (u > 0) {
          p[i] = result[u - 1]
        }
        result[u] = i
      }
    }
  }
  u = result.length
  v = result[u - 1]
  while (u-- > 0) {
    result[u] = v
    v = p[v]
  }
  return result
}
```

## 四、Vue 2 vs Vue 3 Diff算法对比

### 4.1 算法策略对比

| 特性 | Vue 2 | Vue 3 |
|------|-------|-------|
| **算法名称** | 双端比较算法 | 快速Diff算法 |
| **核心思想** | 四个指针双向比较 | 预处理 + 最长递增子序列 |
| **时间复杂度** | O(n) | O(n) |
| **最优情况** | O(min(n,m)) | O(n) |
| **最差情况** | O(n) | O(n) |

### 4.2 性能对比

#### 场景1：头部添加元素
```
旧: [B, C, D]
新: [A, B, C, D]
```

**Vue 2处理：**
- 需要移动B、C、D三个节点
- 3次DOM移动操作

**Vue 3处理：**
- 识别出B、C、D不需要移动
- 只在头部添加A
- 1次DOM插入操作

#### 场景2：复杂序列重排
```
旧: [A, B, C, D, E]
新: [A, C, E, B, D]
```

**Vue 2处理：**
- 双端比较可能产生多次移动
- 需要4次DOM移动操作

**Vue 3处理：**
- 计算最长递增子序列[C, E, D]
- 只移动B到正确位置
- 1次DOM移动操作

### 4.3 源码实现差异

#### 相同节点判断
**Vue 2:**
```js
function sameVnode(a, b) {
  return (
    a.key === b.key &&
    a.tag === b.tag &&
    a.isComment === b.isComment &&
    isDef(a.data) === isDef(b.data) &&
    sameInputType(a, b)
  )
}
```

**Vue 3:**
```ts
function isSameVNodeType(n1: VNode, n2: VNode): boolean {
  return n1.type === n2.type && n1.key === n2.key
}
```

Vue 3的判断更加简洁高效。

### 4.4 移动策略差异

**Vue 2移动策略：**
- 基于双端比较结果直接移动
- 可能产生不必要的中间移动

**Vue 3移动策略：**
- 基于最长递增子序列
- 只移动必要的节点
- 移动次数接近理论最小值

## 五、优化建议

### 5.1 为VNode设置合适的key
```vue
<!-- 好的key -->
<div v-for="item in list" :key="item.id">
  {{ item.name }}
</div>

<!-- 不好的key -->  
<div v-for="(item, index) in list" :key="index">
  {{ item.name }}
</div>
```

### 5.2 避免不必要的重新渲染
- 使用`v-once`静态标记
- 合理使用`shouldComponentUpdate`
- 避免在v-for中使用索引作为key

## 六、总结

Vue 3的Diff算法在Vue 2的基础上进行了重大优化：

1. **算法策略升级**：从双端比较升级为快速Diff算法
2. **移动优化**：引入最长递增子序列最小化DOM移动
3. **性能提升**：在常见场景下性能提升30%-70%
4. **代码简化**：核心逻辑更加清晰简洁