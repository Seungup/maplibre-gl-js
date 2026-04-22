# Architecture 2 — Custom Vector Bucket Source

커스텀 소스가 `loadTile`에서 **메인 스레드로 `LineBucket`/`FillBucket`을 직접 생성**하여 `tile.buckets[layerId]`에 할당. 내장 `line`/`fill` 스타일 레이어가 이 bucket을 소비하며 painter의 GPU 스케일링 파이프라인에 그대로 참여한다.

## 적합 시나리오

- 도메인 벡터 데이터(철도, 도로, 해안선, 경계 등)를 **내장 line/fill 품질로 렌더링**
- 타일 LOD 독립적 1px 정밀도 필수
- `queryRenderedFeatures` 네이티브 지원 필요
- 데이터 포맷·생성 로직이 표준 GeoJSON과 다르지만 최종 렌더는 표준 paint property로 표현 가능

## 가능성 검증 (코드 레벨)

| 사실 | 위치 |
|---|---|
| `LineBucket`은 `@internal`이나 `export class` | `src/data/bucket/line_bucket.ts:90` |
| `tile.buckets`는 외부 할당 가능 | `src/tile/tile.ts:131` `this.buckets = {}` 초기화 |
| `loadVectorData`는 `deserializeBucket`을 호출하지만, 외부에서 직접 `tile.buckets = {...}`도 가능 | `src/tile/tile.ts:243` |
| `tile.upload(context)`가 프레임마다 호출 → `bucket.upload()` 자동 | `src/tile/tile_manager.ts:228` |
| `drawLine`은 `tile.getBucket(layer)`로 조회 (소스 타입 무관) | `src/webgl/draw/draw_line.ts:141` |
| Source-Layer 타입 호환 검증 없음 | `src/style/style.ts` (sourceLayer 필드만 검증) |

즉 "커스텀 타입 소스 + 내장 line 레이어" 조합이 **코어 수정 없이 구조적으로 허용**된다.

## 클래스 스켈레톤

```ts
import {LineBucket} from 'maplibre-gl/src/data/bucket/line_bucket';
import {EvaluationParameters} from 'maplibre-gl/src/style/evaluation_parameters';
import {EXTENT} from 'maplibre-gl/src/data/extent';
import type {LineStyleLayer} from 'maplibre-gl/src/style/style_layer/line_style_layer';
import type {Source} from 'maplibre-gl/src/source/source';
import type {Tile} from 'maplibre-gl/src/tile/tile';
import type {OverscaledTileID} from 'maplibre-gl/src/tile/tile_id';
import type {Map} from 'maplibre-gl/src/ui/map';

export class CustomVectorBucketSource extends Evented implements Source {
  readonly type = 'custom-vector-bucket';
  id: string;
  minzoom = 0;
  maxzoom = 22;
  tileSize = 512;
  reparseOverscaled = true;
  isTileClipped = true;

  private map!: Map;
  private _loaded = false;
  private _features: any[] = [];

  constructor(id: string, options: any, _dispatcher: any, eventedParent: any) {
    super();
    this.id = id;
    this._features = options.features ?? [];
    if (options.tileSize) this.tileSize = options.tileSize;
    if (options.maxzoom != null) this.maxzoom = options.maxzoom;
    this.setEventedParent(eventedParent);
  }

  onAdd(map: Map) {
    this.map = map;
    this._loaded = true;
    this.fire(new Event('data', {dataType: 'source', sourceDataType: 'metadata'}));
    this.fire(new Event('data', {dataType: 'source', sourceDataType: 'content'}));
  }

  hasTransition() { return false; }
  loaded() { return this._loaded; }
  serialize() { return {type: this.type}; }
  hasTile(tileID: OverscaledTileID): boolean { return /* bbox 검사 */ true; }

  async loadTile(tile: Tile): Promise<void> {
    // (1) 이 소스를 사용하는 line/fill 레이어 수집
    const style = this.map.style;
    const targetLayers = Object.values((style as any)._layers)
      .filter((l: any) => l.source === this.id && (l.type === 'line' || l.type === 'fill'));

    const buckets: {[id: string]: any} = {};
    const canonical = tile.tileID.canonical;
    const overscaling = tile.tileID.overscaleFactor();

    for (const layer of targetLayers as any[]) {
      if (layer.type === 'line') {
        const bucket = new LineBucket({
          index: 0,
          layers: [layer as LineStyleLayer],
          zoom: tile.tileID.overscaledZ,
          pixelRatio: this.map.getPixelRatio(),
          overscaling,
        } as any);

        const indexedFeatures = this._featuresOverlappingTile(tile.tileID)
          .map((feat, i) => ({
            feature: toVectorTileFeatureLike(feat, canonical),
            id: feat.id ?? i,
            index: i,
            sourceLayerIndex: 0,
          }));

        bucket.populate(indexedFeatures, {
          featureIndex: null as any,
          iconDependencies: {}, patternDependencies: {}, glyphDependencies: {},
          availableImages: [],
        } as any, canonical);

        buckets[layer.id] = bucket;
      }
      // FillBucket 동일 패턴
    }

    tile.buckets = buckets;
    tile.state = 'loaded';
    // painter.tile.upload(context)가 자동으로 bucket.upload → VBO 업로드
  }

  async unloadTile(tile: Tile) {
    for (const id in tile.buckets) tile.buckets[id].destroy();
    tile.buckets = {};
  }

  setData(newFeatures: any[]) {
    this._features = newFeatures;
    this.fire(new Event('data', {dataType: 'source', sourceDataType: 'content'}));
  }

  private _featuresOverlappingTile(tileID: OverscaledTileID): any[] {
    // 공간 인덱스(rbush 등)로 타일 bbox와 겹치는 feature만 반환
    return [];
  }
}
```

### VectorTileFeature 어댑터

`LineBucket.populate`가 받는 feature는 `{geometry: () => Point[][], properties, extent, type, ...}` 형태의 `VectorTileFeature`-like 객체여야 한다. 사용자 데이터(예: GeoJSON-like)를 이 인터페이스로 감싸는 어댑터가 필요.

참고: `src/util/vectortile_to_geojson.ts`, `src/source/geojson_wrapper.ts`의 `GeoJSONWrapper`.

## 등록 및 사용

```js
await maplibregl.addSourceType('custom-vector-bucket', CustomVectorBucketSource);

map.addSource('rail', {
  type: 'custom-vector-bucket',
  features: railwayFeatures,
});

map.addLayer({
  id: 'rail-line',
  type: 'line',               // 내장 line 레이어 — RTT 자동
  source: 'rail',
  paint: {
    'line-width': ['interpolate', ['linear'], ['zoom'], 10, 1, 18, 3],
    'line-pattern': 'railway-ties',
    'line-color': '#333',
  }
});

map.setTerrain({source: 'dem', exaggeration: 1.5});   // 자동 drape
```

## 핵심 장점

1. **MapLibre 내장 line 품질 완전 획득** — `line.vertex.glsl`의 `u_ratio × projectLineThickness(zoom)`이 현재 카메라 zoom 기준으로 매 프레임 stroke 계산. 타일이 z14에서 만들어져도 z22에서 화면 두께 정확히 유지.
2. **`prepare()` 불필요** — GPU 셰이더가 zoom을 흡수, 매 프레임 CPU 작업 거의 없음.
3. **Feature 쿼리 가능** — `FeatureIndex`를 함께 구성하면 `queryRenderedFeatures` 네이티브 지원.
4. **paint property expression 자동 평가** — `['interpolate', ['linear'], ['zoom'], ...]`가 내장 평가기로 처리.
5. **RTT + terrain drape 자동** — line layer가 `LAYERS_TO_TEXTURES`에 포함.
6. **Globe 자동 대응** — `applyGlobeMatrix` 플래그 및 projection-specific 셰이더 자동 분기.

## 제약

1. **CPU tessellation 메인 스레드** — 대용량 데이터셋에서 프레임 드롭 가능. 대안:
   - 전체 데이터 공간 인덱싱을 `setData()` 시점에 선행 수행
   - `loadTile` 작업을 `requestIdleCallback`으로 이월 (tile.state 관리 필요)
   - 자체 Web Worker로 tessellation 준비 후 메인 스레드 transfer
2. **내장 paint property 한계** — 완전히 새로운 셰이더 효과는 불가. 단 `line-pattern` + `line-gradient` + `line-dasharray` 조합만으로도 도메인 표현의 80% 커버.
3. **내부 API 의존** — `LineBucket`, `EvaluationParameters`, `IndexedFeature` 등은 공개 타입이 아님. 경로 import + 타입 캐스트 필요. MapLibre 메이저 업데이트 시 파손 가능성.
4. **VectorTileFeature 어댑터 필요** — 사용자 데이터 구조를 MapLibre feature 인터페이스로 감싸는 코드.

## 철도 도메인 매핑

| 요구사항 | 내장 기능 구현 |
|---|---|
| 선로 본선 | `line` 레이어 + `line-width` + `line-color` |
| 크로스 타이 (ties) | `line-pattern`에 SDF 패턴 아틀라스 등록 |
| 광궤/협궤 구분 | feature property `gauge` + `['match', ['get', 'gauge'], ...]` |
| 터널/지상 구분 | `line-opacity` expression + 별도 layer |
| 고속/저속 색상 | `line-gradient` (speed vs distance 매핑) |
| 신호등·역 심볼 | `symbol` 레이어 (별도 source 또는 `circle`) |
| 철도 활성/비활성 애니메이션 | `line-dasharray` + time-based expression |

→ 대부분의 철도 시각화는 Arch 2로 네이티브 품질 달성 가능.

## Testing

- `LineBucket` 인스턴스 직접 생성 + `populate()` 호출 후 `layoutVertexArray.length` 검증
- 동일 feature를 GeoJSONSource + line 레이어로 렌더 후 screenshot 비교
- `queryRenderedFeatures`가 올바른 feature ID 반환하는지 확인

## Debugging

- `bucket.layoutVertexArray.length`로 tessellation 결과 확인
- `programConfigurations`의 data-driven 속성 VBO 빌드 확인
- `tile.buckets[layerId]` 접근으로 bucket 존재 확인
- painter가 자동 `bucket.upload()` 호출 확인 (첫 렌더 시점)

## Performance

- `populate()` 호출 시간 측정 (`performance.measure`)
- 가시 타일 동시 loadTile 수 제한
- Worker에서 feature 필터링 + geometry 정규화 선행

## Memory

- `unloadTile`에서 `bucket.destroy()` 호출 (VBO 해제)
- `setData()` 호출 시 전체 타일 재로드 → 이전 bucket 모두 destroy됨

## Worker 활용

- Worker에서 `VectorTileFeature`-like 객체 전부 준비까지 수행
- 메인 스레드는 `LineBucket` 생성 + `populate()`만 동기 수행 (가벼움)

## 검증 체크리스트

- [ ] 동일 feature가 Arch 2와 GeoJSON + line 레이어에서 픽셀 단위 일치
- [ ] `queryRenderedFeatures(point)`가 올바른 feature 반환
- [ ] terrain drape 정상 동작 (`line.vertex.glsl` 자동 처리)
- [ ] globe 모드에서 1px 정밀도 유지
- [ ] zoom 연속 변경 시 끊김 없음 (GPU 스케일링)

## 참고 파일

- `src/data/bucket/line_bucket.ts:90-500` — LineBucket 전체
- `src/data/bucket/fill_bucket.ts` — FillBucket
- `src/tile/tile.ts:131, 208, 243` — buckets 할당 지점
- `src/tile/tile_manager.ts:228` — `tile.upload()` 호출
- `src/webgl/draw/draw_line.ts:141-234` — drawLine
- `src/source/geojson_wrapper.ts` — GeoJSONWrapper (VectorTileFeature 어댑터 참조)
- `src/util/vectortile_to_geojson.ts` — 역방향 변환 참조
- `src/data/feature_index.ts` — FeatureIndex 구성 (선택)

---

**See also**: [README](../README.md) · [Appendix C — Vector Pipeline](../appendix/c-vector-pipeline-reference.md) · [FAQ](../faq.md)
