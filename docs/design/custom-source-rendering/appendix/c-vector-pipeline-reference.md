# Appendix C — Vector Pipeline Reference

MapLibre의 내장 벡터 렌더링 파이프라인(GeoJSONSource/VectorTileSource → line/fill 레이어)이 어떻게 **타일 LOD와 독립적인 1px 정밀도**를 달성하는지 단계별로 정리한다. 이 지식은 각 Architecture가 무엇을 재활용하고 무엇을 대체하는지 이해하는 기반이다.

## 전체 데이터 흐름

```
GeoJSON / MVT
  │
  ▼
[Worker] geojson-vt.getTile(z, x, y) 또는 MVT parse
  │
  ▼
[Worker] WorkerTile.parse()
  │  for each layer:
  │    layer.createBucket({overscaling, zoom, ...})
  │    bucket.populate(features, options, tileID.canonical)
  │       ├── LineBucket.addLine() — miter/bevel join 생성
  │       └── FillBucket — earcut 삼각분할
  │
  ▼
[Main] tile.loadVectorData(data, painter)
  │  deserializeBucket → tile.buckets
  │
  ▼
[Main] bucket.upload(context)
  │  gl.createVertexBuffer(layoutVertexArray)
  │  gl.createIndexBuffer(indexArray)
  │  programConfigurations.upload() — data-driven paint 속성 VBO
  │
  ▼
[GPU per frame] drawLine(painter, tm, layer, coords, options)
  │  for each coord:
  │    tile = tm.getTile(coord)
  │    bucket = tile.getBucket(layer)
  │    program.draw(bucket.layoutVertexBuffer, ...)
  │
  ▼
[GPU shader] line.vertex.glsl
  │  adjustedThickness = projectLineThickness(pos.y)
  │  projected = projectTile(pos + offset / u_ratio * adjustedThickness)
  │
  ▼
Final screen pixels (1px precision regardless of tile zoom)
```

## 1. 데이터 입수 (Worker 영역)

**GeoJSONSource** (`src/source/geojson_source.ts:587`):
- `loadTile(tile)`이 워커로 메시지 전송
- 워커의 `GeoJSONWorkerSource` (`src/source/geojson_worker_source.ts:88`)가 `geojson-vt.getTile(z, x, y)`로 on-the-fly 타일링

**VectorTileSource** (MVT 형식):
- 이미 서버에서 pre-tiled된 protobuf 데이터를 워커가 받음
- 파싱만 수행 (geojson-vt 불필요)

## 2. Worker Tile Processing (bucket 생성)

**`src/source/worker_tile.ts:48-55`**:

```ts
constructor(params: WorkerTileParameters) {
  this.tileID = new OverscaledTileID(params.tileID.overscaledZ, ...);
  this.zoom = params.zoom;                           // = overscaledZ
  this.overscaling = this.tileID.overscaleFactor();  // 2^(overscaledZ - canonical.z)
  this.tileSize = params.tileSize;                   // tileSize * overscaleFactor
}
```

`WorkerTile.parse()`가 각 레이어 패밀리에 대해:

1. Feature 추출 (`data.layers[sourceLayerId]`)
2. `layer.createBucket({overscaling, zoom, ...})` 호출
3. `bucket.populate(features, options, tileID.canonical)`로 tessellation 트리거

### LineBucket의 CPU tessellation (`src/data/bucket/line_bucket.ts:272`)

- `addLine()`이 polyline을 quad strip으로 변환 (두께 있는 폴리곤)
- miter/bevel/round join을 CPU에서 정점 삽입으로 생성
- `miterLimit = 1.05` (line 306), 샤프 코너에서 bevel로 전환
- `overscaling <= 16` 조건으로 코너 subdivision 조정

### FillBucket의 earcut 삼각분할 (`src/data/bucket/fill_bucket.ts`)

- 폴리곤을 earcut 라이브러리로 삼각분할
- `EARCUT_MAX_RINGS = 500` 제한

## 3. CPU → GPU Upload

**`src/tile/tile.ts:208`** `loadVectorData(data, painter)`:

```ts
this.buckets = deserializeBucket(data.buckets, painter?.style);
```

**`src/data/bucket/line_bucket.ts:229`** `upload(context)`:

```ts
upload(context: Context) {
  if (!this.uploaded) {
    this.layoutVertexBuffer = context.createVertexBuffer(this.layoutVertexArray);
    this.indexBuffer = context.createIndexBuffer(this.indexArray);
  }
  this.programConfigurations.upload(context);  // data-driven paint 속성 VBO
  this.uploaded = true;
}
```

Data-driven paint 속성(예: 피처별 `line-color`)은 **별도 VBO**로 업로드된다.

## 4. Per-frame GPU Draw

**`src/webgl/draw/draw_line.ts:141`** `drawLine()`:

- Program variant 선택 (`line` / `lineSDF` / `lineGradient` / `linePattern`)
- `lineUniformValues(painter, tile, layer, pixelRatio)` 생성 (`src/webgl/program/line_program.ts:127`)
  - `u_ratio = ratioScale / pixelsToTileUnits(tile, 1, transform.zoom)`
  - `u_units_to_pixels`: 카메라 행렬 스케일링
  - `u_device_pixel_ratio`: 안티앨리어싱
- `program.draw(bucket.layoutVertexBuffer2, ...)`

## 5. 1px 정밀도의 본질 — GPU 셰이더

**`src/shaders/glsl/line.vertex.glsl:32`** (간략화):

```glsl
float adjustedThickness = projectLineThickness(pos.y);
vec4 projected = projectTile(
  pos + offset2 / u_ratio * adjustedThickness
       + dist / u_ratio * adjustedThickness
);
```

**핵심**:

- Worker는 `floor(zoom)` 기준으로 두께 있는 quad를 미리 extrude (타일 단위 `[0, EXTENT]`)
- 셰이더가 매 프레임 **현재 fractional zoom**으로 `u_ratio`를 계산
- 같은 VBO로 zoom 10에서도 22에서도 **화면 기준 정확히 1px**
- 기하 재생성 없이 fractional zoom까지 부드럽게 스케일

## 6. Style Expression 평가 (CPU/GPU 분할)

**`src/style/properties.ts`**:

- `DataDrivenProperty<T>`: 피처 종속 (`['get', 'speed']`) — per-feature VBO
- `ZoomDependentProperty`: 줌 전용 (`['interpolate', ['linear'], ['zoom'], 10, 1, 15, 4]`)
  - CPU에서 **floor(zoom)**으로 평가 (layout 시점)
  - GPU에서 fractional zoom을 `projectLineThickness(pos.y)`로 보정

**`src/style/style_layer/line_style_layer.ts:75-77`**:

```ts
(this.paint._values as any)['line-floorwidth'] =
  lineFloorwidthProperty.possiblyEvaluate(
    this._transitioningPaint._values['line-width'].value,
    parameters   // Math.floor(zoom)
  );
```

## 7. Feature Indexing (query 지원)

**`src/data/feature_index.ts`**:

- Worker가 parse 중 `FeatureIndex` 구성
- `featureIndex.bucketLayerIDs[bucketIndex] = family.map(l => l.id)`
- `queryRenderedFeatures()`가 화면 좌표 → 피처 ID 조회에 사용

Arch 2에서 `FeatureIndex`를 함께 구성하면 `queryRenderedFeatures`가 네이티브 동작한다.

## 핵심 관찰 — Arch 2가 재활용하는 부분

| 컴포넌트 | Arch 2에서의 처리 |
|---|---|
| Worker bucket 생성 | 재활용 불가 — 커스텀 소스는 워커에 WebGL 접근 없음 |
| LineBucket CPU tessellation | **메인 스레드에서 직접 실행** — `new LineBucket(...)` + `populate()` |
| `tile.buckets` 할당 | 재활용 (`tile.buckets[layerId] = bucket`) |
| `bucket.upload()` | 재활용 (painter가 자동 호출) |
| `drawLine` 파이프라인 | **완전 재활용** — painter가 동일하게 동작 |
| GPU `u_ratio` 셰이더 | **완전 재활용** — 1px 정밀도 그대로 상속 |
| `FeatureIndex` | 선택적 (수동 구성 필요) |

즉 Arch 2의 작업은 "메인 스레드에서 Worker가 하던 일(bucket populate)을 수행"으로 요약된다. GPU 이후 단계는 전혀 건드리지 않는다.

## 커스텀 raster 경로(Arch 1)가 이 이점을 못 얻는 이유

- raster 텍스처는 **stroke width가 baked** 되어 있음
- `u_ratio` 같은 매 프레임 스케일링 메커니즘 없음
- 대응: Arch 1의 전략 D에서 `prepare()` 재래스터화로 근사 (Appendix D 참조)

## 참고 파일

- `src/source/geojson_worker_source.ts:88` — `geojson-vt` 통합
- `src/source/worker_tile.ts:48-127` — WorkerTile.parse
- `src/data/bucket/line_bucket.ts:90-500` — LineBucket 전체
- `src/data/bucket/fill_bucket.ts` — FillBucket (earcut)
- `src/tile/tile.ts:208-294` — loadVectorData/unloadVectorData
- `src/webgl/draw/draw_line.ts:141-234` — drawLine
- `src/webgl/program/line_program.ts:127-140` — lineUniformValues
- `src/shaders/glsl/line.vertex.glsl` — line vertex 셰이더
- `src/style/properties.ts` — Property 계층
- `src/data/feature_index.ts` — FeatureIndex

---

**See also**: [README](../README.md) · [Arch 2](../architectures/arch-2-custom-vector-bucket-source.md) · [Appendix D — Overscale](d-overscale-strategies.md)
