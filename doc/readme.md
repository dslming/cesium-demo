### 图像瓦片渲染

#### 1、相关类和层次

```js
// 使用
const widget = new Cesium.CesiumWidget("cesiumContainer", {
    imageryProvider: new Cesium.UrlTemplateImageryProvider({
        url: "https://webst{s}.is.autonavi.com/appmaptile?x={x}&y={y}&z={z}&lang=zh_cn&size=1&scl=1&style=7",
        subdomains: ["01", "02", "03", "04"]
    })
});

// 内部调用
scene.globe.imageryLayers.addImageryProvider(imageryProvider);
```

```js
/**
 * 如有必要，应优先更换瓷砖，以便为新瓷砖腾出空间。队列实现为链表。
 */
class TileReplacementQueue {}

// 栅格图层
class UrlTemplateImageryProvider {}

// 其他图层的容器
class ImageryLayer {
    constructor() {
        this._reprojectComputeCommands = [];
    }

    // 更新帧状态以执行任何排队纹理重新投影。
    queueReprojectionCommands() {
        const computeCommands = this._reprojectComputeCommands;
        const length = computeCommands.length;
        for (let i = 0; i < length; ++i) {
            frameState.commandList.push(computeCommands[i]);
        }
        computeCommands.length = 0;
    }

    /**
     * 如有必要，在下一次更新时，将重新投影纹理到{@link GeographicProjection}的命令排队，并生成地理纹理的mipmaps。
     */
    _reprojectTexture() {}
}

// 图片层的有序集合
class ImageryLayerCollection {
    constructor() {
        this._layers = [];
        this.layerRemoved = new Event();
        this.layerMoved = new Event();
        this.layerShownOrHidden = new Event();
    }
    _update() {
        // 更新图像层的show状态
    }

    addImageryProvider(imageryProvider) {
        const layer = new ImageryLayer(imageryProvider);
        this.add(layer, index);
    }

    swapLayers() {
        collection.layerMoved.raiseEvent(temp, j, i);
    }

    // 将图层提升到集合的顶部。
    raiseToTop() {
        this.layerMoved.raiseEvent(layer, this._layers.length - 1, index);
    }

    lowerToBottom() {
        this.layerMoved.raiseEvent(layer, 0, index);
    }

    // 更新队列的投影
    queueReprojectionCommands() {
        const layers = this._layers;
        for (let i = 0, len = layers.length; i < len; ++i) {
            layers[i].queueReprojectionCommands(frameState);
        }
    }

    add() {
        if (!hasIndex) {
            index = this._layers.length;
            this._layers.push(layer);
        } else {
            this._layers.splice(index, 0, layer);
        }

        this._update();
        this.layerAdded.raiseEvent(layer, index);
    }
}

// 地球表面
class Globe {
    constructor() {
        this._imageryLayerCollection = new ImageryLayerCollection();

        // QuadtreePrimitive
        this._surface = new QuadtreePrimitive({
            tileProvider: new GlobeSurfaceTileProvider({
                imageryLayers: this._imageryLayerCollection
            })
        });
    }

    get imageryLayers() {
        return this._imageryLayerCollection;
    }

    update() {
        this._surface.update();
    }

    beginFrame() {
        this._surface.beginFrame();
    }

    render() {
        this._surface.render();
    }

    endFrame() {
        this._surface.endFrame();
    }
}

function createRenderCommandsForSelectedTiles(primitive:QuadtreePrimitive, frameState) {
  const tileProvider = primitive._tileProvider;
  const tilesToRender = primitive._tilesToRender;

  for (let i = 0, len = tilesToRender.length; i < len; ++i) {
    const tile = tilesToRender[i];
    tileProvider.showTileThisFrame(tile, frameState);
  }
}
/*
 * 四叉树瓦片图元
 *通过利用细节级别和剔除来渲染大量数据集。地球表面分为
 *四叉树的瓷砖，根部为大的低细节瓷砖，叶子为小的高细节瓷砖。
 *要渲染的瓷砖集是通过将瓷砖中几何误差的估计值投影到
 *估计屏幕空间误差的屏幕，以像素为单位，必须低于用户指定的阈值。
 *平铺的实际内容是任意的，并使用{@link QuadtreeTileProvider}指定。
 */
class QuadtreePrimitive {
    constructor(options) {
        this._tilesToRender = [];
        this._levelZeroTiles = []
        this._tileProvider:GlobeSurfaceTileProvider = options.tileProvider;
    }

    beginFrame() {
        // Gets commands for any texture re-projections
        this._tileProvider.initialize(frameState);

        // 清除瓦片下载队列
        clearTileLoadQueue(this);

        this._tileReplacementQueue.markStartOfRenderFrame();
    }
}

// 地球表面的四叉树图瓦片提供者
class GlobeSurfaceTileProvider {
    constructor(imageryLayers: ImageryLayerCollection) {
        this._imageryLayers = options.imageryLayers;

        this._imageryLayers.layerMoved.addEventListener(this._onLayerMoved);

        // 需要销毁的顶点数组
        this._vertexArraysToDestroy = [];
    }

    _onLayerMoved() {
        this._layerOrderChanged = true;
        this._imageryLayersUpdatedEvent.raiseEvent();
    }

    update() {
        this._imageryLayers._update();
    }

    /**
     * 在每个渲染帧开始调用
     */
    initialize() {
        // 更新每个层以进行纹理重投影。
        this._imageryLayers.queueReprojectionCommands(frameState);

        // 释放需要销毁的顶点数组
    }
}
```

**类的层次:**

```js
|--Scene;
    |--Globe
        |--QuadtreePrimitive
            |--GlobeSurfaceTileProvider
                        |--QuadtreeTile
```

#### 2、单帧流程

```js
scene.render() {
   // ① globe.update()
   prePassesUpdate() -> globe.update()

   // ② globe.beginFrame();
   globe.beginFrame();

   // ③ globe.render()
   updateAndExecuteCommands() -> executeCommandsInViewport() -> updateAndRenderPrimitives() ->  globe.render()

   // ④ globe.endFrame();
   globe.endFrame();
}
```

##### 2.1 globe.update()

```js
globe.update() {
  // ①更新图像层的show状态
  GlobeSurfaceTileProvider.update();
}
```

##### 2.2 globe.beginFrame()

```js
globe.beginFrame() {
  QuadtreePrimitive.beginFrame()
}
```

##### 2.3 globe.render()

```js
globe.render() {
    QuadtreePrimitive.render(frameState);
}

QuadtreePrimitive.render() {
    //
    const tileProvider:GlobeSurfaceTileProvider = this._tileProvider;
    tileProvider.beginUpdate(frameState);

    selectTilesForRendering(this, frameState);
    createRenderCommandsForSelectedTiles(this, frameState);

    tileProvider.endUpdate(frameState);
}
```

###### 2.3.1 selectTilesForRendering()

```js
function selectTilesForRendering(primitive:GlobeSurfaceTileProvider, frameState) {
    // 1) Clear the render list.
    const tilesToRender = primitive._tilesToRender;
    tilesToRender.length = 0;

    // GlobeSurfaceTileProvider
    const tileProvider = primitive._tileProvider;
    // 2) 创建零级瓦片
    primitive._levelZeroTiles = QuadtreeTile.createLevelZeroTiles(tilingScheme);

    createRenderCommandsForSelectedTiles(this, frameState);
}

class QuadtreePrimitive {
    render() {
        selectTilesForRendering(this, frameState);
    }
}

// 四叉树瓦片
class QuadtreeTile{

    // 为“详细级别零”（最粗糙、最不详细的级别）创建一组矩形平铺。
    static createLevelZeroTiles(tilingScheme) {
        // tilingScheme:GeographicTilingScheme
        const numberOfLevelZeroTilesX = tilingScheme.getNumberOfXTilesAtLevel(0);
        const numberOfLevelZeroTilesY = tilingScheme.getNumberOfYTilesAtLevel(0);
        const result = new Array(numberOfLevelZeroTilesX * numberOfLevelZeroTilesY);

        let index = 0;
        for (let y = 0; y < numberOfLevelZeroTilesY; ++y) {
            for (let x = 0; x < numberOfLevelZeroTilesX; ++x) {
                result[index++] = new QuadtreeTile({
                    tilingScheme: tilingScheme,
                    x: x,
                    y: y,
                    level: 0,
                });
            }
        }

  return result;
    }
}

/**
 * 地理平铺方案
 * 参考简单{@link GeographicProjection}的几何体的平铺方案，其中经度和纬度直接映射到X和Y。
 * 此投影通常被称为地理、等矩形、等距圆柱或平板。
*/
class GeographicTilingScheme {
    constructor() {
        // 零级Tilex数
        this._numberOfLevelZeroTilesX = 2;
        this._numberOfLevelZeroTilesY = 1;
    }
    // 获取指定详细级别下X方向上的平铺总数。
    getNumberOfXTilesAtLevel() {
        return this._numberOfLevelZeroTilesX << level;
    }

    // 获取指定详细级别下Y方向上的平铺总数。
    getNumberOfYTilesAtLevel(){
        return this._numberOfLevelZeroTilesY << level;
    }
}


class GlobeSurfaceTileProvider {

    // 地形的平铺方案
    get tilingScheme(){
      return this._terrainProvider.tilingScheme;
  },
}
```

### Ref

https://zhuanlan.zhihu.com/p/511339180
https://zhuanlan.zhihu.com/p/389940746
[中文文档](http://cesium.xin/cesium/cn/Documentation1.62/Globe.html?classFilter=Globe)
