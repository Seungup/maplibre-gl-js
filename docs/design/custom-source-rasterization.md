# Custom Source 기반 Vector 데이터 렌더링 (RTT 호환) 설계

## ★ 아키텍처 선택 — 두 갈래 경로

MapLibre에서 CustomLayer 대신 커스텀 소스로 벡터 데이터를 렌더링하고 싶을 때, 목표에 따라 **두 가지 전혀 다른 아키텍처**가 가능하다. 본 문서는 이 둘을 모두 다룬다.

| | **Architecture 1 — Custom Raster Source** | **Architecture 2 — Custom Vector Bucket Source** | **Architecture 3 — CustomLayer RTT 확장 (코어 수정)** |
|---|---|---|---|
| 핵심 아이디어 | `loadTile`에서 빈 텍스처 할당 + `prepare()`에서 FBO 래스터화 → `tile.texture` | `loadTile`에서 **main-thread로 LineBucket 직접 생성** → `tile.buckets[layerId]` 할당 | `LAYERS_TO_TEXTURES.custom = true` + CustomLayer에 `renderTile(gl, tileID, matrix)` API 추가 |
| 레이어 타입 | `type: 'raster'` 스타일 레이어 | `type: 'line'` (또는 'fill', 'circle') 내장 스타일 레이어 | `type: 'custom'` + 새 `renderToTexture: true` 옵션 |
| 1px 정밀도 | 텍스처에 baked, `prepare()` 재래스터화 필요 | **GPU 셰이더 u_ratio로 매 프레임 자동** — 내장 vector와 동일 | 사용자 셰이더 책임 (u_ratio-style 구현 가능) |
| 연속 zoom 부드러움 | 수동 (전략 D, 프레임당 draw call N개) | 공짜 — 기하 재사용, 유니폼만 바뀜 | 사용자 셰이더 설계에 따라 공짜 가능 |
| 셰이더 자유도 | **완전 — 임의 GLSL** | 내장 paint property (`line-width`, `line-color`, `line-pattern`, `line-gradient`, `line-dasharray`, SDF dash) 한정 | **완전 — 임의 GLSL** |
| Tile LOD 독립성 | **LINEAR 필터 + 텍스처 해상도 상한** → 내장 vector와 완전 동등하지 않음 | **완전 동등** — 내장 line 레이어와 픽셀 단위 일치 | **완전 동등 가능** (사용자 구현에 달림) |
| Worker 활용 | X — 메인 스레드 동기 래스터화 | X — 메인 스레드 tessellation (단, GPU 비용 없음) | 사용자 재량 |
| Feature 쿼리 | 미지원 (수작업 인덱스) | 선택적 (`FeatureIndex` 함께 구성 시 네이티브) | 미지원 (수작업) |
| RTT/terrain drape | O (`raster` RTT 화이트리스트) | O (`line`/`fill` RTT 화이트리스트) | O (`custom` RTT 화이트리스트 추가 필요) |
| 내부 API 의존 | `Texture`, `Context` (경로 import) | `LineBucket`, `FillBucket`, `ProgramConfigurationSet`, `IndexedFeature` 등 다수 (경로 import) | **없음 — 공식 API** |
| 코어 수정 | 불필요 | 불필요 | **필요** (`render_to_texture.ts`, `draw_custom.ts`, `custom_style_layer.ts`) |

### 선택 가이드 (★ 중요)

- **철도·도로·해안선·등고선 등 "표준 line/fill 스타일로 표현 가능한 도메인 벡터"**: **Architecture 2 권장.**  
  내장 `line` 레이어의 `line-pattern`(크로스 타이용 SDF 패턴), `line-dasharray`, `line-gradient`, `line-width` expression을 조합하면 철도 같은 도메인 표현이 대부분 가능하다. Tile LOD와 독립적으로 모든 fractional zoom에서 픽셀 완벽한 선을 얻는 유일한 방법.

- **내장 paint property로 표현 불가능한 특수 셰이더 효과**(예: 파티클, 유체 시뮬레이션 시각화, 커스텀 lighting, 3D 볼륨 효과 등): **Architecture 1.**  
  GPU scaling을 포기하는 대신 셰이더 자유도를 얻는다. 1px 엄격 정밀도는 전략 D 재래스터화로 근사한다.

- **하이브리드 가능**: 같은 커스텀 소스가 `tile.buckets`(표준 line)와 `tile.texture`(커스텀 오버레이)를 동시에 생성하고, 두 레이어(`line` + `raster`)가 같은 소스를 참조. 예: 철도 본선은 vector line, 특수 효과 레이어는 custom raster.

---

## Context

**문제**: MapLibre의 `CustomLayer`는 translucent 패스에서 framebuffer에 직접 그리므로 `RenderToTexture`(RTT) 합성에서 제외된다. 즉 3D terrain, terrain 위 합성 등 RTT 의존 효과가 적용되지 않는다 (`src/webgl/render_to_texture.ts:16-23` 의 `LAYERS_TO_TEXTURES` 화이트리스트에 `custom` 부재).

**필요**: 사용자 정의 벡터 데이터(예: 동적 GIS 지오메트리)를 terrain 위에 정확히 드레이프(drape)해서 그리고 싶다.

**해결 아이디어**: 
- `addSourceType()`으로 **커스텀 소스**를 등록한다
- 소스의 `loadTile(tile)`은 **빈 WebGL 텍스처 할당 + 렌더 큐잉**만 수행한다 (FBO 쓰기 금지)
- 실제 GPU 오프스크린 FBO 래스터화는 **`prepare()`에서 집중 처리** — painter draw 직전의 결정론적 시점에, 배치당 1회 GL 상태 save/restore로 타일들을 순회 렌더
- 결과 텍스처를 `tile.texture`에 할당하면 기존 **`type: 'raster'` 스타일 레이어**가 그대로 렌더한다
- raster 레이어는 RTT 화이트리스트에 포함되어 있으므로 **terrain 합성이 자동**으로 작동한다
- 가시 타일 결정은 `TileManager`가 내부적으로 `coveringTiles`를 호출해주므로, 소스는 `hasTile`/`loadTile`/`prepare`/`hasTransition`/`unloadTile`만 구현하면 된다

이 설계의 장점은 MapLibre 코어를 수정하지 않고 기존 공개 API만으로 RTT 호환 커스텀 렌더링을 달성한다는 점이다.

**추가 요구사항 — Overscale 및 연속 zoom 1px 정밀도**: GeoJSON + line/fill 레이어가 보장하는 "현재 카메라 zoom에서 화면 기준 1px" 정밀도를 raster 경로에서도 유지해야 한다. 이는 두 층위의 문제를 동시에 해결해야 한다.

1. **정수 zoom 경계** — raster의 기본 magnify 동작은 canonical 타일을 확대하여 흐려진다. `reparseOverscaled` 플래그 + `OverscaledTileID` 기반 접근 또는 `maxzoom` 상향으로 해결.
2. **연속 fractional zoom** — 같은 overscaledZ 내에서도 카메라 zoom은 연속적이다. Vector는 draw time 셰이더 uniform으로 stroke를 매 프레임 갱신하지만, raster는 텍스처에 stroke가 구워져 있어 불가능. 해법: `hasTransition() = true` + `prepare()`에서 현재 fractional zoom 기준 **매 프레임 재래스터화** (전략 D).

상세는 "Overscale 지원" 섹션의 전략 A/B/C/D 참조. 권장 조합은 **전략 A + D**.

**산출물 범위**: 본 문서는 설계/PoC 가이드만 제공한다. 코어 코드 변경 없음.

---

## 핵심 발견 사항 (코드 검증 완료)

| 사실 | 위치 | 설명 |
|---|---|---|
| `addSourceType` 공개 API | `src/source/source.ts:193-198` | `Source` 인터페이스만 만족하면 어떤 클래스든 등록 가능 |
| `Source` 인터페이스 계약 | `src/source/source.ts:31-131` | 필수: `loadTile`, `serialize`, `loaded`, `hasTransition` |
| 레이어→소스 타입 검증 없음 | `src/style/style.ts` | `raster` 레이어는 어떤 소스 타입과도 결합 가능 (sourceLayer 검증만 vector/geojson 한정) |
| 렌더 분기는 **레이어 타입** 기준 | `src/render/painter.ts:658-690` | `raster` 레이어 → `drawRaster()` |
| `drawRaster`는 `tile.texture`만 요구 | `src/webgl/draw/draw_raster.ts:39-145` | `Texture` 객체 한 개면 충분 |
| RTT는 `raster` 포함 | `src/webgl/render_to_texture.ts:16-23` | `LAYERS_TO_TEXTURES.raster = true` → terrain 자동 합성 |
| TileManager가 가시 타일 자동 관리 | `src/tile/tile_manager.ts` | `coveringTiles` 결과로 `loadTile`/`unloadTile` 자동 호출, 소스는 `hasTile`로 필터링만 |
| 참조 구현 | `src/source/raster_tile_source.ts:182-217` | `tile.texture = new Texture(context, img, gl.RGBA, …)` 패턴 |

---

## 아키텍처 원칙 (★ 중요)

**`loadTile`과 실제 FBO 렌더링은 분리**한다. `loadTile`은 GL 텍스처 할당 + 렌더 큐잉까지만 수행하고, 모든 FBO 쓰기는 `prepare()`에 집중시킨다. 이유:

1. **결정론적 타이밍** — `prepare()`는 `TileManager.prepare(context)` (`src/tile/tile_manager.ts:221-231`)를 통해 painter draw 직전의 정해진 시점에 호출된다. `loadTile`은 Promise 기반이라 resolve 타이밍이 painter 중간일 수도 있어 GL 상태 충돌 위험이 있다.
2. **배치 amortization** — GL 상태 save/restore를 타일마다 반복하지 않고 `prepare()` 진입/종료 시 1회만 수행. 타일 N개 렌더 비용 = 1 × save + N × draw + 1 × restore.
3. **코드 경로 통합** — "초기 래스터화"와 "연속 zoom 재래스터화" (전략 D)가 동일한 단일 루틴을 공유한다.
4. **멱등성** — 같은 타일에 `loadTile`이 여러 번 호출되어도 큐에 플래그만 세팅되므로 중복 draw call 없음.
5. **비동기 안전성** — `loadTile`이 painter render 중간에 resolve 되어도 실제 GL draw는 다음 `prepare()`까지 지연되므로 painter의 바인딩 상태를 오염시키지 않는다.

**타일 라이프사이클 (새 구조)**:
```
TileManager.loadTile(tile) 호출
  ↓
Source.loadTile(tile):
  - gl.createTexture() → 빈 텍스처 핸들
  - Texture 래퍼로 wrap, tile.texture에 부착
  - this._pendingRender.add(tile.tileID.key) (큐잉)
  - tile.state = 'loaded' (빈 텍스처지만 등록)
  - Promise.resolve()
  ↓
(다음 프레임 시작)
  ↓
TileManager.prepare(context)
  → Source.prepare():
      gl.getParameter(VIEWPORT / FRAMEBUFFER_BINDING) [1회]
      gl.bindFramebuffer(this._fbo)                    [1회]
      for each tile in (_pendingRender ∪ staleZoomTiles):
          gl.framebufferTexture2D(tile.texture.texture)
          gl.viewport(0, 0, size, size)
          gl.clear(COLOR_BUFFER_BIT)
          this._rasterizeTile(gl, tile, currentZoom)
          _lastRasterizedZoom.set(tile.tileID.key, currentZoom)
      gl.bindFramebuffer(prevFB), gl.viewport(prev)    [1회]
      ctx.bindFramebuffer.dirty = true
      _pendingRender.clear()
  ↓
painter.renderLayer(raster layer) → drawRaster(tile.texture)
```

**`loadTile` resolve와 첫 렌더 사이의 blank 프레임 문제**: `prepare()`는 매 프레임 painter draw 직전에 실행되므로, `loadTile` Promise resolve 후 painter가 호출되기 전에 반드시 `prepare()`가 먼저 실행된다. 따라서 blank 텍스처가 화면에 노출되는 프레임은 없다 (`src/ui/map.ts` 렌더 순서: style.update → style.prepare → painter.render).

---

## 설계 (코드 변경 없음, PoC 가이드)

### 클래스 골격 (★ loadTile/prepare 분리 반영)

```ts
import {Texture} from 'maplibre-gl/src/render/texture';
import {Event, Evented} from 'maplibre-gl/src/util/evented';
import type {Source} from 'maplibre-gl/src/source/source';
import type {Tile} from 'maplibre-gl/src/tile/tile';
import type {OverscaledTileID} from 'maplibre-gl/src/tile/tile_id';
import type {Map} from 'maplibre-gl/src/ui/map';

export class CustomVectorRasterSource extends Evented implements Source {
  readonly type = 'custom-vector-raster';
  id: string;
  minzoom = 0;
  maxzoom = 22;              // 전략 A: overscale 회피
  tileSize = 512;
  roundZoom = true;
  // reparseOverscaled는 기본 false (전략 A + D 권장 조합)

  private map!: Map;
  private _loaded = false;
  private _fbo: WebGLFramebuffer | null = null;
  private _program!: WebGLProgram;                           // 사용자 정의 셰이더
  private _userData: any;

  // 렌더 큐잉 (전략 D와 통합)
  private _pendingRender = new Set<string>();                // loadTile 후 첫 렌더 대기
  private _lastRasterizedZoom = new Map<string, number>();   // 연속 zoom 추적
  private _tilesByKey = new Map<string, Tile>();             // key → Tile 역참조
  private _zoomEpsilon = 0.1;

  constructor(id: string, options: any, _dispatcher: any, eventedParent: any) {
    super();
    this.id = id;
    this._userData = options.data;
    if (options.tileSize) this.tileSize = options.tileSize;
    if (options.minzoom != null) this.minzoom = options.minzoom;
    if (options.maxzoom != null) this.maxzoom = options.maxzoom;
    if (options.zoomEpsilon != null) this._zoomEpsilon = options.zoomEpsilon;
    this.setEventedParent(eventedParent);
  }

  // --- Source 인터페이스 구현 ---

  onAdd(map: Map) {
    this.map = map;
    const gl = map.painter.context.gl;
    this._fbo = gl.createFramebuffer();
    this._program = compileUserShaders(gl);
    this._loaded = true;
    this.fire(new Event('data', {dataType: 'source', sourceDataType: 'metadata'}));
    this.fire(new Event('data', {dataType: 'source', sourceDataType: 'content'}));
  }

  onRemove() {
    const gl = this.map.painter.context.gl;
    if (this._fbo) gl.deleteFramebuffer(this._fbo);
    gl.deleteProgram(this._program);
    // 남은 타일 텍스처 정리
    for (const tile of this._tilesByKey.values()) {
      if (tile.texture) gl.deleteTexture((tile.texture as any).texture);
    }
    this._tilesByKey.clear();
    this._pendingRender.clear();
    this._lastRasterizedZoom.clear();
  }

  loaded() { return this._loaded; }
  serialize() { return {type: this.type, tileSize: this.tileSize}; }

  hasTile(tileID: OverscaledTileID): boolean {
    return isTileOverlappingUserData(tileID, this._userData);
  }

  // ★ 재렌더 필요 여부 판정 → 맵의 프레임 루프 유지
  hasTransition(): boolean {
    if (this._pendingRender.size > 0) return true;
    if (!this.map) return false;
    const z = this.map.getZoom();
    for (const [key, baked] of this._lastRasterizedZoom) {
      if (Math.abs(z - baked) > this._zoomEpsilon) return true;
    }
    return false;
  }

  // ★ loadTile은 "텍스처 할당 + 큐잉"만. FBO 렌더는 하지 않는다.
  async loadTile(tile: Tile): Promise<void> {
    const ctx = this.map.painter.context;
    const gl = ctx.gl;
    const size = this.tileSize;

    const texHandle = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, texHandle);
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, size, size, 0, gl.RGBA, gl.UNSIGNED_BYTE, null);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);

    const tex = new Texture(ctx, {width: size, height: size, data: null} as any, gl.RGBA);
    (tex as any).texture = texHandle;
    (tex as any).size = [size, size];
    tile.texture = tex;

    const key = tile.tileID.key;
    this._tilesByKey.set(key, tile);
    this._pendingRender.add(key);     // prepare()에서 실제 렌더
    tile.state = 'loaded';             // 텍스처는 빈 상태지만 등록
  }

  async unloadTile(tile: Tile) {
    const key = tile.tileID.key;
    this._tilesByKey.delete(key);
    this._pendingRender.delete(key);
    this._lastRasterizedZoom.delete(key);
    if (tile.texture) {
      this.map.painter.context.gl.deleteTexture((tile.texture as any).texture);
      tile.texture = null;
    }
  }

  // ★ 모든 FBO 렌더링이 여기에 집중됨. save/restore는 프레임당 1회.
  prepare(): void {
    if (!this.map || !this._fbo) return;

    const currentZoom = this.map.getZoom();
    const toRender: string[] = [];

    // pending: loadTile 후 아직 한 번도 렌더 안 된 타일
    for (const key of this._pendingRender) toRender.push(key);

    // stale zoom: 마지막 래스터화 시점 zoom과 현재 zoom 차이가 임계값 초과
    for (const [key, baked] of this._lastRasterizedZoom) {
      if (this._pendingRender.has(key)) continue;   // 중복 방지
      if (Math.abs(currentZoom - baked) > this._zoomEpsilon) toRender.push(key);
    }

    if (toRender.length === 0) return;

    const ctx = this.map.painter.context;
    const gl = ctx.gl;
    const size = this.tileSize;

    // === GL 상태 저장 (배치당 1회) ===
    const prevViewport = gl.getParameter(gl.VIEWPORT) as Int32Array;
    const prevFB = gl.getParameter(gl.FRAMEBUFFER_BINDING);
    gl.bindFramebuffer(gl.FRAMEBUFFER, this._fbo);
    gl.viewport(0, 0, size, size);
    gl.useProgram(this._program);
    // blend/depth 등 필요한 상태도 여기서 설정

    try {
      for (const key of toRender) {
        const tile = this._tilesByKey.get(key);
        if (!tile || !tile.texture) continue;
        const texHandle = (tile.texture as any).texture;
        gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, texHandle, 0);
        gl.clearColor(0, 0, 0, 0);
        gl.clear(gl.COLOR_BUFFER_BIT);
        this._rasterizeTile(gl, tile, currentZoom);
        this._lastRasterizedZoom.set(key, currentZoom);
      }
    } catch (e) {
      console.warn(`CustomVectorRasterSource[${this.id}] prepare failed:`, e);
    } finally {
      // === GL 상태 복원 (배치당 1회) ===
      gl.bindFramebuffer(gl.FRAMEBUFFER, prevFB);
      gl.viewport(prevViewport[0], prevViewport[1], prevViewport[2], prevViewport[3]);
      ctx.bindFramebuffer.dirty = true;
      this._pendingRender.clear();
    }
  }

  setData(newData: any) {
    this._userData = newData;
    // 모든 캐시 타일을 pending으로 재돌림 → 다음 prepare에서 재래스터화
    for (const key of this._tilesByKey.keys()) this._pendingRender.add(key);
    this.fire(new Event('data', {dataType: 'source', sourceDataType: 'content'}));
  }

  private _rasterizeTile(gl: WebGL2RenderingContext, tile: Tile, currentZoom: number) {
    // (a) tile.tileID.canonical → mercator bbox
    // (b) currentZoom uniform 전달 (stroke = desiredPx / 2^(currentZoom - canonical.z))
    // (c) this._userData를 tile-local [0..EXTENT]로 변환해 draw
    // EXTENT는 src/data/extent.ts (값 8192)
  }
}
```

### 핵심 설계 포인트 요약

1. **loadTile은 non-blocking "텍스처 예약"만 수행** — GL createTexture + 빈 storage 할당 + 큐잉. 실제 픽셀은 prepare에서.
2. **prepare가 모든 FBO 쓰기의 단일 진입점** — 초기 렌더 + stale zoom 재렌더가 동일 경로.
3. **GL 상태 save/restore 1회/프레임** — bindFramebuffer/viewport/useProgram을 루프 외부에서. 타일 N개 = 1 save + N attach/draw + 1 restore.
4. **`_pendingRender` Set과 `_lastRasterizedZoom` Map 분리** — 초기 렌더(첫 그림) vs 연속 재렌더(zoom 변화)의 의미를 명확히.
5. **첫 프레임 blank 노출 없음** — render 순서 `style.prepare → painter.render` 덕분에 로드된 타일은 같은 프레임 내 prepare에서 그려진 뒤 painter에 전달.
6. **hasTransition은 `_pendingRender 비어있지 않거나 stale 타일 존재`** — 정지 시 false 반환하여 idle 이벤트 발생 가능.

### 등록 및 사용

```js
await maplibregl.addSourceType('custom-vector-raster', CustomVectorRasterSource);

map.addSource('my-vec', {
  type: 'custom-vector-raster',
  data: { /* GeoJSON 등 사용자 데이터 */ },
  tileSize: 512,
});

map.addLayer({
  id: 'my-vec-layer',
  type: 'raster',                 // ★ raster 레이어로 선언 → 자동 RTT
  source: 'my-vec',
  paint: { 'raster-opacity': 1.0 }
});

// terrain 활성화 — 커스텀 소스 텍스처가 elevation 위로 정확히 드레이프됨
map.setTerrain({source: 'dem', exaggeration: 1.5});
```

---

## 주요 결정 사항 (사용자 선택 반영)

1. **산출물**: 코어 코드 변경 없음. 본 문서 + PoC 스니펫이 가이드 역할.
2. **래스터화 방식**: GPU 오프스크린 FBO. `map.painter.context`의 WebGL 컨텍스트를 공유하여 zero-copy로 텍스처를 painter에 넘긴다.
3. **coveringTiles 활용**: TileManager에 위임. 소스는 `hasTile()`로 사용자 데이터와 겹치는 타일만 필터링.
4. **loadTile / FBO 렌더 분리 (★ 핵심)**: `loadTile`은 빈 텍스처 할당 + 큐잉만 수행. 모든 FBO 쓰기는 `prepare()`에 집중 — 결정론적 타이밍, 배치 amortization, 초기/재렌더 코드 경로 통합.
5. **Overscale / 연속 zoom 정밀도**: 전략 A (`maxzoom` 상향) + 전략 D (`hasTransition() + prepare()` 기반 동적 재래스터화) 조합. 매 프레임 현재 fractional zoom 기준으로 가시 타일을 재래스터화하여 vector의 1px 선 두께와 동등한 정밀도를 유지.

---

## Overscale (Overzoom) 지원 — 1px 정밀도

### 문제 제기

raster 소스 기본 동작은 텍스처 확대(magnify). 카메라 줌이 소스의 `maxzoom`을 넘으면 기존 텍스처가 bilinear/LINEAR로 늘어나며 흐려진다. 반면 vector(GeoJSON/Vector tile + line/fill 레이어)는 `reparseOverscaled = true`로 overscaled 줌마다 bucket 재조립 → 화면에서 정확히 1px 선을 얻는다. 본 설계를 그대로 두면 vector가 보장하던 1px 정밀도를 잃는다.

### 메커니즘 확인

| 요소 | 위치 | 작동 |
|---|---|---|
| 플래그 정의 | `src/source/source.ts:65` | `reparseOverscaled?: boolean` |
| GeoJSONSource 활성화 | `src/source/geojson_source.ts:167` | `this.reparseOverscaled = true` |
| VectorTileSource 활성화 | `src/source/vector_tile_source.ts:98` | 동일 |
| coveringTiles에서 적용 | `src/geo/projection/covering_tiles.ts:272` | `const overscaledZ = options.reparseOverscaled ? Math.max(it.zoom, thisTileDesiredZ) : it.zoom;` |
| TileManager 호출 | `src/tile/tile_manager.ts:518` | `reparseOverscaled: this._source.reparseOverscaled` 전달 |
| 타일 크기 확장 | TileManager가 `new Tile(tileID, tileSize * tileID.overscaleFactor())` | overscale 배수만큼 타일 크기 자동 증가 |
| overscaleFactor | `src/tile/tile_id.ts:212-214` | `Math.pow(2, overscaledZ - canonical.z)` |
| Vector의 1px 구현 | `src/source/vector_tile_source.ts:210-211` | `zoom: overscaledZ`, `tileSize: tileSize * overscaleFactor()` → worker가 overscaledZ 기준으로 재 layout |
| VectorTile worker 사용 | `src/source/worker_tile.ts:48-55` | `this.overscaling = this.tileID.overscaleFactor()` → bucket 생성 시 stroke/심볼 크기 보정 |
| Terrain 무관 | `src/tile/terrain_tile_manager.ts:101` | DEM 전용 하드코딩, **소스 레이어 타일 요청에는 영향 없음** |

### 전략 선택 (우선순위)

권장 전략은 두 가지이며, 사용자 데이터 특성에 따라 선택/혼합한다.

#### 전략 A — `maxzoom`을 충분히 크게 설정 (가장 단순)

```ts
constructor(...) {
  // reparseOverscaled 미설정 (false 유지)
  this.maxzoom = 24;   // 실사용 최대 줌보다 높게
}
```

- 동작: 모든 카메라 줌에서 canonical 타일이 새로 요청된다. overscale 자체가 발생하지 않는다.
- `loadTile` 내부에서는 `tile.tileID.canonical.z`만 참조해 해당 줌 기준으로 1px 선을 직접 그리면 끝.
- FBO 크기는 항상 `this.tileSize`(512). 메모리 안정적.
- 단점: 모든 줌 레벨에서 fresh rasterize 발생 → 고정 벡터 데이터라면 저줌 캐시 재활용 불가.

#### 전략 B — `reparseOverscaled = true` + overscaled 기반 래스터화

```ts
constructor(...) {
  this.reparseOverscaled = true;
  this.maxzoom = 16;   // 실제 데이터 소스의 자연스러운 상한
}

async loadTile(tile: Tile) {
  const ctx = this.map.painter.context;
  const gl = ctx.gl;
  const overscaleFactor = tile.tileID.overscaleFactor();   // 2^(overscaledZ - canonical.z)
  const OVERSCALE_CAP = 4;                                 // 메모리 방어
  const effectiveFactor = Math.min(overscaleFactor, OVERSCALE_CAP);
  const fboSize = this.tileSize * effectiveFactor;         // 1px 정밀도용 확대 FBO

  // (FBO/텍스처 할당은 fboSize 기준)
  // ...

  // 셰이더에 uniform으로 overscaledZ 전달 — stroke width 등은 overscaledZ 기준으로 계산
  gl.uniform1f(uOverscaledZ, tile.tileID.overscaledZ);
  gl.uniform1f(uOverscaleFactor, overscaleFactor);
  this.rasterizeTile(gl, tile.tileID);
}
```

- 동작: canonical 타일 하나가 overscaled 줌마다 별도 `loadTile` 호출을 받는다. overscaled 줌을 알고 있으므로 stroke 폭 등을 해당 줌 기준 1px로 계산 가능.
- FBO를 `tileSize * effectiveFactor`로 할당하면 drawRaster의 LINEAR 샘플링에도 alias 최소화.
- **주의점**:
  - `overscaleFactor`가 2^n으로 증가하므로 상한 캡(예: 4 또는 8) 필수. 캡 없으면 GPU 메모리 폭발.
  - TileManager는 `new Tile(tileID, tileSize * tileID.overscaleFactor())`로 타일의 논리 크기를 이미 확장했지만, 이는 feature query/좌표 계산용일 뿐 우리가 FBO 크기를 반드시 그 값으로 만들 필요는 없다. FBO는 우리 재량.
  - `Texture.size` 필드는 drawRaster의 uniform 계산에 사용되지 않음(샘플링이 0..1 UV이므로). 안전.

#### 전략 C — 혼합 (권장 기본값)

- 기본 `maxzoom = 22`, `reparseOverscaled = false`로 시작 (전략 A).
- 대용량 벡터 데이터로 매 줌마다 rasterize 비용이 크다면 사용자가 `maxzoom` 낮추고 `reparseOverscaled = true`로 전환.
- 생성자에서 옵션으로 노출:
  ```ts
  new CustomVectorRasterSource(id, {
    data: geojson,
    maxzoom: 22,
    reparseOverscaled: false,
    overscaleCap: 4,   // 전략 B일 때만 사용
    tileSize: 512,
  });
  ```

#### 전략 D — `hasTransition() + prepare()` 기반 연속 zoom 재래스터화 (★ 권장)

전략 A/B는 **정수 줌 변경** 시에만 `loadTile`을 재호출한다. 반면 카메라 줌은 연속적(예: 14.37)이므로 같은 canonical/overscaled 타일 내에서 fractional zoom이 변해도 텍스처는 그대로다 → 선 두께가 화면에서 미세하게 스케일되어 1px 엄밀성은 정수 경계에서만 유지된다.

Vector 경로는 이 문제를 **draw time 셰이더에서 해결**한다 (line-width uniform이 매 프레임 현재 zoom으로 평가됨). Raster 경로는 draw time에 stroke를 변경할 수 없으므로, **매 프레임 `prepare()`에서 현재 카메라 zoom 기준으로 FBO 재래스터화**해야 연속 1px 정밀도를 얻는다.

**호출 흐름 (코드 검증됨)**

- `Source.hasTransition()` → `TileManager.hasTransition()` (`src/tile/tile_manager.ts:923-929`) → `Style.hasTransitions()` (`src/style/style.ts:687-711`) → `map.ts:3703` `_styleDirty = true` → `map.ts:3720` `triggerRepaint()` → 다음 프레임 진행
- 각 프레임 painter draw 전에 `TileManager.prepare(context)` (`src/tile/tile_manager.ts:221-231`)가 호출되고 내부에서 `this._source.prepare()` 실행 (`tile_manager.ts:222-224`)
- `prepare()` 시점은 `tile.texture`가 painter의 `drawRaster`로 읽히기 **이전**이므로, 여기서 FBO를 갱신하면 해당 프레임 렌더에 즉시 반영됨

**구현은 앞서 "클래스 골격" 섹션의 `prepare()` / `hasTransition()` / `_pendingRender` / `_lastRasterizedZoom` 조합을 그대로 사용한다** (loadTile/prepare 분리 원칙과 자연스럽게 통합됨).

**핵심 설계 포인트**

1. **`hasTransition()`은 정적 `true`가 아니라 조건부** — `_pendingRender` 비어있지 않거나 stale 타일 존재 시에만 true. 카메라 정지 + 모든 타일 동기화 시 false → `idle` 이벤트 정상 발생.
2. **`_zoomEpsilon`으로 재래스터화 빈도 제어** — 0.0이면 매 프레임 재래스터화(고비용), 1.0이면 정수 줌 변경 시에만. 권장 초기값 0.1 ≈ zoom 0.1 차이(선 두께 약 7%).
3. **FBO 텍스처 핸들 재사용** — loadTile에서 할당한 `tile.texture.texture` 핸들을 prepare()에서 framebufferTexture2D로 반복 바인딩. GPU 메모리 재할당 비용 없음.
4. **전략 A/B와 직교** — 전략 D는 "연속 zoom에서의 선 두께 보정", 전략 A/B는 "특정 카메라 zoom에서의 충분한 해상도". 실사용 시 **전략 A + D 조합**이 가장 자연스러움: `maxzoom=22`로 overscale 회피 + `prepare()`로 연속 zoom 보정.
5. **`_tilesByKey` 역참조로 TileManager private 접근 회피** — `loadTile`/`unloadTile`/`setData` 훅에서 자기 자신이 tile 레퍼런스를 관리하므로 `map.style.tileManagers[id]` 같은 내부 속성 접근 불필요.
6. **타일이 많을 때 성능** — 가시 타일 N개 × 매 프레임 재래스터화는 draw call N개 추가를 의미. 임계값(`_zoomEpsilon`)으로 완화하되, 타일당 rasterize가 무거운 데이터셋에서는 `requestIdleCallback` 또는 2-프레임 나눠 처리 등 쓰로틀 고려.

### 전략 B를 사용할 때의 텍스처 크기 (loadTile/prepare 분리 반영)

전략 B(`reparseOverscaled=true`)에서는 overscale 배율에 따라 FBO 크기를 키워야 한다. loadTile/prepare 분리 구조에서는 **texture 크기 결정은 loadTile**에서(할당 시점), **실제 그리기는 prepare**에서 수행한다.

```ts
// loadTile 내부 — 텍스처 크기만 overscale 반영
async loadTile(tile: Tile) {
  const overscaleFactor = tile.tileID.overscaleFactor();
  const effectiveFactor = this.reparseOverscaled
    ? Math.min(overscaleFactor, this._overscaleCap)
    : 1;
  const size = this.tileSize * effectiveFactor;   // 타일별로 달라질 수 있음

  // texImage2D(..., size, size, ...) 로 할당
  // 이 타일의 size를 _tileSizes Map에 기록 (prepare에서 viewport 설정에 사용)
  this._tileSizes.set(tile.tileID.key, size);
  // ... (나머지 할당/큐잉 로직)
}

// prepare 내부 — 타일별 viewport 조정
prepare() {
  // ...
  for (const key of toRender) {
    const size = this._tileSizes.get(key) ?? this.tileSize;
    gl.viewport(0, 0, size, size);           // 타일마다 viewport 갱신
    gl.framebufferTexture2D(...);
    gl.clear(gl.COLOR_BUFFER_BIT);
    this._rasterizeTile(gl, tile, currentZoom, {
      canonicalZ: tile.tileID.canonical.z,
      overscaledZ: tile.tileID.overscaledZ,
      overscaleFactor: tile.tileID.overscaleFactor(),
    });
  }
}
```

전략 A(권장)에서는 모든 타일이 동일한 `this.tileSize`이므로 `viewport`는 prepare 진입 시 1회 설정이면 충분하고 `_tileSizes` Map은 불필요하다.

### Overscale 검증 체크리스트 (기존 검증에 추가)

1. `reparseOverscaled = false` (전략 A) 시나리오: 카메라 줌 10 → 22 연속 이동 시 각 줌에서 fresh loadTile 호출 확인, 타일 경계 선 두께 일정(1px) 확인.
2. `reparseOverscaled = true` (전략 B) 시나리오: maxzoom=14로 설정 후 줌 22로 이동 → `tile.tileID.canonical.z === 14`, `tile.tileID.overscaledZ === 22`, `overscaleFactor === 256` 되는지 `console.log`로 확인. overscaleCap=4 적용되어 FBO 크기가 512*4=2048로 클램프되는지 확인.
3. 1px 선 렌더링 비교: 동일 벡터 데이터를 `type: 'line'` + GeoJSON 소스로 같이 렌더해 overscale 극단 줌에서 두 결과의 선 두께가 일치(±0.5px)하는지 비교.
4. terrain 활성화 시 overscale과 drape 동시 동작 확인: `setTerrain()` + 고줌 이동에서 텍스처가 elevation을 따라 휘면서도 선이 1px로 유지되는지.
5. 전략 D 연속 zoom 검증:
   - 카메라 zoom을 14.0 → 14.5 → 15.0으로 천천히 드래그. GeoJSON + line 레이어의 1px 선과 비교해 육안 두께가 항상 일치하는지.
   - `hasTransition()`이 이동 중에는 `true`, 정지 후 1프레임 내에 `false`로 돌아오는지 확인. `map.on('idle', ...)` 이벤트 발생 확인.
   - `prepare()` 내 rasterize 호출 횟수 측정 (console.count). `_zoomEpsilon` 조정에 따라 감소하는지.
   - GPU 메모리 안정성: 1분간 zoom in/out 반복 후 Chrome DevTools Memory → GPU 증가 없어야 함 (텍스처 핸들 재사용 확인).

---

## 위험 요소와 주의점

| 항목 | 주의 |
|---|---|
| painter GL 상태 오염 | `prepare()` 진입 시 viewport/FBO/program/blend/depth 모두 백업, 종료 시 복원. 배치당 1회이므로 비용 부담 없음. |
| `ctx.bindFramebuffer.dirty` | Context 래퍼가 캐시한 바인딩 상태를 무효화하지 않으면 다음 painter draw가 우리 FBO에 그려질 수 있음. `prepare()` 종료 시 반드시 `ctx.bindFramebuffer.dirty = true`. |
| `loadTile`에서 FBO 렌더 금지 | 본 설계의 핵심 원칙. loadTile은 gl.createTexture + texImage2D로 빈 storage만 할당. 실제 그리기는 prepare에서만. |
| 첫 프레임 blank 리스크 | 실제로는 발생하지 않음. render 순서 `style.prepare → painter.render`에서 prepare가 먼저 실행되므로 로드 직후 프레임에 렌더 완료. |
| 텍스처 메모리 누수 | `unloadTile`/`onRemove`에서 반드시 `gl.deleteTexture()` 호출. RasterTileSource의 `painter.saveTileTexture()` 풀은 사용 금지 (FBO 백킹 텍스처 재사용 부적합). |
| 타일 좌표 변환 | 재사용 가능한 utility 없음. `EXTENT`(`src/data/extent.ts`, 값 8192)와 `tileID.canonical`로 직접 계산. `MercatorCoordinate.fromLngLat` 활용. |
| TypeScript 타입 | `addSource`의 specification 유니온에 커스텀 타입이 없음. 사용자 코드에서 `as any` 캐스트 또는 모듈 augmentation으로 우회 (코어 수정 없이 가능). |
| 워커 미사용 | maplibre-gl-js의 worker는 WebGL 접근 불가. GPU 래스터화는 메인 스레드 `prepare()`에서 동기적으로 수행. 타일당 draw call이 가벼워야 프레임 드롭 회피. |
| overscale 메모리 | 전략 B에서 `overscaleFactor` 상한 없으면 텍스처 크기가 2^n으로 폭증 (factor 16 → 8192×8192 ≈ 256MB RGBA). 반드시 `overscaleCap` 지정. |
| TileManager 논리 타일 크기 | TileManager는 overscale 시 `tile.tileSize = tileSize * overscaleFactor()`로 설정하지만 이는 feature query 용이며, 우리 텍스처 크기는 별도 결정 가능. 혼동 주의. |
| `hasTransition()` 과도 활성화 | 정적 `true` 반환 시 `idle` 이벤트 미발생 + 배터리/CPU 소모. 반드시 `_pendingRender 비어있지 않거나 stale 타일 존재 시에만` true 반환. |
| `prepare()` 내 예외 처리 | 매 프레임 호출되므로 rasterize 중 에러 발생 시 즉시 프레임 드롭. try/catch로 감싸고 실패 시 기존 텍스처 유지 + console.warn. **중요**: catch 블록에서도 GL 상태 복원은 반드시 실행되어야 함 (finally 사용). |
| `_tilesByKey` 일관성 | loadTile/unloadTile이 Map 항목을 정확히 추가/삭제해야 메모리 누수 없음. `onRemove`에서 모든 항목 정리 필수. |
| prepare 성능 | draw call 수 = `_pendingRender.size + staleCount`. 수백 개 타일이 동시에 stale될 수 있음(예: 큰 zoom 점프). 필요 시 프레임당 최대 N개로 제한하여 다음 프레임으로 이월. |

---

## 검증 방법

1. **개발 서버 기동**: `npm run start` (포트 9966).
2. **PoC HTML 작성** (선택): `test/examples/add-a-custom-vector-raster-source.html`에 위 스켈레톤 적용. 단순 사각형 폴리곤을 빨강으로 그리는 셰이더로 시작.
3. **기본 렌더 확인**: terrain 없이 팬/줌. 타일 경계에 seam 없어야 함 (`CLAMP_TO_EDGE` 덕분).
4. **RTT/terrain 검증** (핵심):
   - DEM 소스 추가 후 `map.setTerrain({source: 'dem', exaggeration: 1.5})`
   - 빨간색 폴리곤이 elevation을 따라 휘어져야 함
   - `exaggeration` 값을 0 → 2 로 슬라이더 조작 시 베이스맵 raster와 함께 왜곡되어야 함 → RTT 합성 성공
5. **GL 에러 모니터링**: `loadTile` 끝에 `console.assert(gl.getError() === 0)`.
6. **타일 라이프사이클**: `loadTile`/`unloadTile`에 `console.log` 삽입. 팬으로 화면 밖으로 보낸 타일이 unload되는지 확인. Chrome DevTools → Memory → GPU 메모리 증가 없어야 함.
7. **데이터 갱신**: `source.setData(newGeo)` 호출 후 1-2 프레임 내 모든 가시 타일이 재래스터화되는지 확인.

---

## 참고 파일 (변경 없음, 참조용)

- `src/source/source.ts:31-131` — `Source` 인터페이스 정의
- `src/source/source.ts:193-198` — `addSourceType`
- `src/source/raster_tile_source.ts:182-217` — `loadTile` 참조 패턴
- `src/source/raster_tile_source.ts:226-230` — `unloadTile` 참조 패턴
- `src/render/texture.ts` — `Texture` 클래스 (생성자 시그니처 확인 후 핸들 주입 방식 검토)
- `src/webgl/draw/draw_raster.ts:39-145` — `tile.texture` 사용 방식
- `src/webgl/render_to_texture.ts:16-23` — RTT 화이트리스트 (`raster: true` 확인)
- `src/tile/tile_manager.ts:221-231` — `prepare(context)`에서 `this._source.prepare()` 호출 (★ 전략 D 핵심)
- `src/tile/tile_manager.ts:923-929` — `hasTransition()` 위임 (★ 전략 D 핵심)
- `src/style/style.ts:687-711` — `Style.hasTransitions()` 집계
- `src/ui/map.ts:3703, 3720` — `_styleDirty` + `triggerRepaint` 흐름
- `src/tile/tile_manager.ts` — `hasTile`/`loadTile` 호출 흐름, overscale 시 `tileSize * overscaleFactor()` 적용 지점
- `src/tile/tile_manager.ts:518` — `reparseOverscaled`를 coveringTiles에 전달
- `src/tile/tile_id.ts:89-138` — `OverscaledTileID`, `overscaledZ`, `canonical`
- `src/tile/tile_id.ts:212-214` — `overscaleFactor() = 2^(overscaledZ - canonical.z)`
- `src/geo/projection/covering_tiles.ts:272` — overscale 분기 로직
- `src/source/geojson_source.ts:167` / `src/source/vector_tile_source.ts:98` — `reparseOverscaled = true` 참조 구현
- `src/source/worker_tile.ts:48-55` — vector가 `overscaling`을 layout에 쓰는 방식 (참고용, 우리는 셰이더 uniform으로 대체)
- `src/tile/terrain_tile_manager.ts:101` — DEM 전용 하드코딩 (본 설계와 무관, 혼동 방지용 참고)
- `src/data/extent.ts` — `EXTENT` 상수 (타일 로컬 좌표 변환에 사용)

---

## 실제 벡터 렌더링 파이프라인과 본 설계의 차이

MapLibre의 GeoJSON/Vector 소스가 어떻게 line/fill 레이어를 "모든 줌에서 픽셀 완벽한 1px"으로 그리는지, 그리고 본 설계가 그와 어떻게 다른지를 비교한다. **핵심**: vector는 "CPU layout + GPU per-frame scaling"의 2단 구조로 1px 정밀도를 달성하지만, raster는 draw time에 stroke 제어 수단이 없다.

### 벡터 파이프라인 (GeoJSON + line 레이어 기준)

```
[Worker]                                          [Main]                         [GPU per-frame]
GeoJSON
  → geojson-vt.getTile(z,x,y)                    tile.loadVectorData
  → WorkerTile.parse()                             ↓                             drawLine()
  → layer.createBucket({overscaling, zoom})        deserializeBucket              → lineUniformValues(zoom, pixelRatio)
  → LineBucket.addLine()                           ↓                             → program.draw(bucket.VBOs, zoom, paint)
    · miter/bevel join 생성 (CPU)                  bucket.upload(context)
    · floor(zoom) 기준 width로 quad extrude         → gl.createVertexBuffer          [Shader: line.vertex.glsl]
  → layoutVertexArray + indexArray                  → gl.createIndexBuffer          projectLineThickness(pos.y) × u_ratio
  → data-driven paint는 별도 VBO로                  → programConfigurations.upload   → 최종 screen 픽셀 stroke
```

**주요 코드 지점**:
- `src/source/geojson_worker_source.ts:88` — `geojson-vt.getTile(z,x,y)`로 on-the-fly 타일링
- `src/source/worker_tile.ts:63` — `WorkerTile.parse()` 진입점
- `src/data/bucket/line_bucket.ts:272` — `addLine()` CPU 테셀레이션
- `src/data/bucket/fill_bucket.ts` — earcut 기반 fill 삼각분할 (최대 500 rings)
- `src/tile/tile.ts:208,243` — `loadVectorData`, `deserializeBucket`
- `src/data/bucket/line_bucket.ts:229` — `upload(context)`에서 VBO 생성
- `src/webgl/draw/draw_line.ts:141,212` — `drawLine`, `lineUniformValues`
- `src/webgl/program/line_program.ts:127` — `u_ratio = ratioScale / pixelsToTileUnits(tile, 1, zoom)`
- `src/shaders/glsl/line.vertex.glsl:32` — `projectLineThickness(pos.y)`로 현재 줌 기반 두께 보정
- `src/style/style_layer/line_style_layer.ts:75-77` — layout 시 `Math.floor(zoom)` 적용 (`lineFloorwidthProperty`)

### 본 설계 (Custom Vector-Raster Source)

```
[Main — loadTile or prepare()]                                   [GPU per-frame]
사용자 벡터 데이터
  → 커스텀 셰이더로 FBO에 래스터화                               drawRaster()
    · tile.tileID 기반 mercator bbox 계산                         → tile.texture.bind()
    · viewport = tileSize × overscaleFactor                       → rasterUniformValues
    · gl.clear + 사용자 draw (임의 셰이더/스타일)                   → program.draw(MESH, TRIANGLES)
  → tile.texture = FBO 백킹 WebGL 텍스처
  → (전략 D) prepare()에서 zoom 변화 시 동일 핸들에 재그리기       [Shader: raster.vertex/fragment]
                                                                    → UV 0..1 bilinear 샘플 — stroke 제어 없음
```

### 단계별 핵심 차이

| 단계 | Vector (line/fill) | 본 설계 (custom raster) |
|---|---|---|
| 데이터 → 타일 | 워커에서 `geojson-vt`로 동적 타일링 | 메인 스레드에서 `loadTile`이 타일 영역 직접 래스터화 |
| Tessellation | 워커 CPU — miter/bevel/earcut | 없음. 사용자 셰이더가 원하는 방식으로 픽셀 생성 |
| 데이터 구조 | `layoutVertexArray` + `indexArray` + paint별 VBO | `WebGLTexture` 1장 |
| 메모리 | 기하 복잡도 선형 (edges×4) | 고정 (`tileSize² × 4B`) |
| 업로드 | 워커→메인 transfer + `gl.createVertexBuffer` | 없음 (FBO가 곧 결과) |
| Draw-time 비용 | 작음 — 셰이더에 `u_ratio`, `zoom`, paint 유니폼만 전달 | 매우 작음 — 텍스처 쿼드 1회 |
| 1px 정밀도 원리 | **GPU 셰이더가 `pos × u_ratio × projectLineThickness(zoom)`** 로 매 프레임 계산 | **CPU/GPU 래스터화가 `stroke = px / 2^(zoom-canonical.z)`** 로 FBO에 굽는다 (전략 D로 매 프레임 갱신) |
| 연속 zoom 부드러움 | 공짜 — draw-time 유니폼만 바뀜, 기하는 재사용 | 수동 — `prepare()` 재래스터화 필요 (draw call N개 추가) |
| 정수 zoom 경계 | 워커가 `floor(zoom)` 기준 재layout (reparseOverscaled=true) | `reparseOverscaled=true` 또는 `maxzoom` 상향 (전략 A/B) |
| Data-driven paint | 피처별 속성을 별도 VBO로 → 셰이더에서 직접 평가 | 사용자가 셰이더에 직접 uniform/attribute 제공 책임 |
| Feature 쿼리 | 네이티브 (`FeatureIndex`, `src/data/feature_index.ts`) + `queryRenderedFeatures` | **미지원** — 필요 시 소스가 별도 공간 인덱스 유지해야 함 |
| Style expression | `Property`/`PossiblyEvaluated`/`Transitioning` 인프라가 자동 평가 (`src/style/properties.ts:45-494`) | 사용자가 수작업 |
| Worker 병렬화 | O — 타일 파싱/테셀레이션이 메인 스레드 막지 않음 | X — WebGL 워커 접근 불가, 모두 메인 스레드 동기 처리 |
| RTT(terrain) | O — line/fill이 `LAYERS_TO_TEXTURES`에 포함 | O — raster 레이어로 포함 |
| 임의 셰이더 효과 | 제한적 — 내장 `line-*`, `fill-*` paint property만 | **자유** — 사용자 GLSL 전체 통제 (본 설계의 주된 이점) |

### 왜 벡터는 "공짜로" 1px 정밀도인가

핵심은 `line.vertex.glsl`의 이 한 줄:
```glsl
vec4 projected = projectTile(pos + offset / u_ratio * projectLineThickness(pos.y) + ...);
```
- `pos`는 worker가 타일 단위 [0..EXTENT]로 만든 고정 정점
- `u_ratio = ratioScale / pixelsToTileUnits(tile, 1, camera.zoom)` — 현재 카메라 줌에 따라 매 프레임 바뀜
- `projectLineThickness(pos.y)`는 paint expression을 현재 zoom으로 평가한 값
- 결과: **동일한 VBO로 zoom 10에서도 22에서도 화면 기준 1px**. 기하 재생성 없이 fractional zoom까지 부드럽게 스케일.

본 설계는 이 GPU scaling을 `u_ratio` 대신 래스터화 시점의 상수로 대체한다 → fractional zoom 변경 시 텍스처를 다시 구워야 함 → **전략 D의 존재 이유**.

### 결론: 언제 본 설계를 선택해야 하는가

| 상황 | 권장 |
|---|---|
| 표준 line/fill/circle/symbol + terrain drape | **GeoJSON + line/fill 레이어** (기본 경로가 충분). 본 설계 불필요 |
| CustomLayer로 그리던 특수 셰이더 효과를 terrain drape까지 확장 | **본 설계 (custom source → raster 레이어 → RTT)** |
| 동적 데이터셋이지만 표준 paint property로 표현 가능 | GeoJSONSource + `setData()` 사용. 본 설계보다 Worker 활용 우수 |
| queryRenderedFeatures 필수 | 본 설계 부적합. 별도 인덱스 구축 필요 |
| Fractional zoom 1px 엄격 요구 | 본 설계 + **전략 D 필수**. 비용 용인 안 되면 GeoJSON + line 레이어가 최선 |

즉, 본 설계는 **"CustomLayer로만 가능한 특수 렌더링을 terrain-drape까지 확장"** 이라는 좁은 목표에 최적화된 구조이며, 표준 line/fill로 표현 가능한 것은 기존 벡터 경로가 항상 더 효율적이다.

---

## Architecture 1이 RTT의 "per-tile 렌더" 역할을 대체하는가 (★ 중요)

**질문**: "벡터 소스 + line 레이어 + terrain은 결국 RTT를 통해 per-tile FBO에 line geometry가 렌더된 뒤 drape된다. 커스텀 소스도 `prepare()`에서 per-tile FBO에 렌더해두면 동일한 결과를 얻는 것 아닌가?"

**답**: **그렇다. Architecture 1의 `prepare()`가 정확히 그 역할을 한다.** 다만 그 구조가 두 가지 방식으로 "같은 일"을 하고 있으므로 세부 차이를 명확히 할 필요가 있다.

### 두 경로의 렌더 흐름 비교

**벡터 경로 (GeoJSON + line 레이어 + terrain)**:

```
Frame N:
  (1) TileManager.prepare(context)                             [source는 특별한 일 없음]
  (2) painter.render()
      (2a) RenderToTexture.renderLayer(line_layer)
           for (각 terrain 타일):
             bind FBO_terrainTile                              ← RTT가 만든 per-tile FBO
             viewport = tile-local
             for (각 source 타일 coord):
               drawLine(tile = tileManager.getTile(coord))
                 → bucket.layoutVertexBuffer 바인딩
                 → line.vertex.glsl (u_ratio × projectLineThickness(zoom))
                 → FBO_terrainTile에 라인 픽셀 기록     ← per-tile 렌더 "완성"
           drawTerrain(...)                                    ← drape
```

**커스텀 소스 경로 (Architecture 1 + raster 레이어 + terrain)**:

```
Frame N:
  (1) TileManager.prepare(context)
      → Source.prepare()
        bind FBO_customSource                                  ← 우리가 만든 per-tile FBO
        viewport = this.tileSize
        for (각 pending/stale 타일):
          framebufferTexture2D(tile.texture.texture)          ← tile.texture가 attachment
          clear + 사용자 셰이더로 draw                         ← per-tile 렌더 "완성"
  (2) painter.render()
      (2a) RenderToTexture.renderLayer(raster_layer)
           for (각 terrain 타일):
             bind FBO_terrainTile
             for (각 source 타일 coord):
               drawRaster(tile = tileManager.getTile(coord))
                 → tile.texture 바인딩 (=우리가 prepare에서 그린 것)
                 → 텍스처 쿼드 1회 draw → FBO_terrainTile에 복사
           drawTerrain(...)                                    ← drape
```

### 결론: 사용자 직관은 정확하다

- **벡터 경로의 "RTT 루프 안에서 일어나는 per-tile 렌더"**와 **커스텀 소스 경로의 `prepare()` 안에서 일어나는 per-tile 렌더**는 **구조적으로 완전히 동일한 일**이다. 둘 다 "per-tile FBO에 사용자/내장 셰이더로 geometry를 그린다"라는 동일 작업이다.
- 차이는 **그 렌더가 일어나는 시점과 장소**:
  - 벡터: RTT가 직접 만든 FBO에 line 셰이더로 바로 기록 (원-패스)
  - 커스텀 소스: 우리가 만든 FBO(=`tile.texture`)에 먼저 기록 → 이후 drawRaster가 텍스처 쿼드로 RTT FBO에 복사 (투-패스)

### 투-패스의 오버헤드가 실제로 문제인가

**거의 문제 아니다.** `drawRaster`의 추가 비용은 타일당 쿼드 1개(2 triangles, 6 vertex)의 텍스처 샘플링이며, GPU 관점에서 microsecond 단위이다. 그 대가로 얻는 이점:

1. **`prepare()` 타이밍 결정론성** — painter 진입 전에 실행되므로 RTT 내부 타이밍과 얽히지 않음
2. **RTT pool과 독립적인 텍스처 라이프사이클** — RTT FBO pool이 재활용되어도 `tile.texture`는 소스가 직접 관리
3. **텍스처 캐싱 이득** — 같은 `tile.texture`가 zoom 변화가 없으면 여러 프레임 재사용 (RTT는 fingerprint 기반 재사용, `src/webgl/render_to_texture.ts:104-126`)
4. **코어 수정 불필요**

### "원-패스로 만들 수 있는가" (이론상의 Architecture 4)

원칙적으로는 가능하다. `drawRaster`가 `tile.texture`를 샘플링하는 대신 **소스가 제공한 `renderToTile(gl, tile)` 훅을 직접 호출**하도록 바꾸면 된다. 그러면 RTT FBO 위에서 사용자 셰이더가 직접 그리게 된다.

- 필요 변경: `drawRaster`에 분기 추가 또는 커스텀 source 타입을 인식하는 새 레이어 타입 도입
- 단점: 본 PoC "코어 수정 없음" 원칙 위반 / 추가 복잡성 / 얻는 이점은 미미
- 권장하지 않음 (Architecture 1의 투-패스 구조가 실용적 최적)

### 사용자 질문에 대한 최종 응답

"`prepare()`에서 동일하게 렌더링하면 되지 않느냐" — **그렇다. 정확히 그 접근이 Architecture 1의 설계 의도이며, 벡터 소스가 RTT에서 수행하는 per-tile 렌더와 기능적으로 동등하다.** 단 한 가지 차이는 "우리 FBO → tile.texture → RTT FBO로 한 번 더 블릿되는 투-패스 구조"뿐이며, 이는 실질적 비용이 무시할 수준이고 타이밍/캐싱/코어 수정 회피 측면에서 오히려 이점이 있다.

---

## (참고) Architecture 3 — CustomLayer 자체를 RTT에 넣는 가설적 경로

위 분석과는 별개로, "CustomLayer 자체를 RTT 대상으로 만들면 어떻게 되는가"를 탐구한 결과는 다음과 같다 (본 PoC 스코프 외, 참고용).

"벡터 레이어는 terrain에서 RTT로 렌더링된다. 그렇다면 bucket 없이도 동일하게 가능하지 않은가?"라는 자연스러운 의문이 든다. 답은 **"부분적으로 맞지만 근본적인 구조 이유 때문에 불가능"** 이다.

### RTT의 실제 동작 (코드 레벨 재확인)

`src/webgl/render_to_texture.ts:140-205`의 `renderLayer` 루프:

```
for (각 terrain 타일) {
  painter.context.bindFramebuffer.set(obj.fbo.framebuffer)     // 타일용 FBO 바인딩
  painter.context.viewport.set([0, 0, tileSize, tileSize])     // tile-local viewport
  for (스택의 각 layer) {
    coords = _coordsAscending[layer.source][tileKey]            // 이 FBO 타일에 해당하는 소스 타일 IDs
    painter.renderLayer(painter, tm, layer, coords, options)    // ← 여기에 coords 전달
  }
}
```

- line/fill: `drawLine(..., coords, ...)`가 `coords`를 받아 `for (coord of coords) { tile = tm.getTile(coord); bucket = tile.getBucket(layer); renderBucket(bucket) }`. **bucket이 per-tile geometry 컨테이너 역할.**
- raster: `drawRaster(..., coords, ...)`도 동일 패턴, `tile.texture`가 per-tile 데이터 역할.
- custom: `drawCustom(painter, tm, layer, options)` — **`coords` 파라미터 자체가 시그니처에 없음** (`src/webgl/draw/draw_custom.ts:8`). 그냥 `implementation.render(gl, {modelViewProjectionMatrix, …})`를 한 번 호출.

### CustomLayer + RTT = 왜 구조적으로 깨지는가

`LAYERS_TO_TEXTURES`에 `custom: true`를 추가하면 어떻게 될지 시뮬레이션:

1. RTT가 타일 FBO를 바인딩하고 viewport를 `(0,0, tileSize, tileSize)`로 설정
2. `drawCustom`이 호출되지만 `coords`는 무시됨
3. CustomLayer의 `render(gl, customLayerArgs)`가 **전체 맵의 `modelViewProjectionMatrix`** 로 그림
4. 전체 세계 좌표 → 전체 맵 NDC로 투영된 결과가 **작은 타일 FBO에 압축되어 기록됨** → 왜곡된 픽셀 데이터
5. 이 왜곡된 타일 텍스처가 terrain 위에 drape됨 → 시각적으로 완전히 부정확

즉 RTT는 **"per-tile 로컬 좌표계에서 그릴 수 있는 무언가"** 가 필요한데, CustomLayer의 `render` 계약은 "full-map 좌표계에서 한 번 그린다"에 고정되어 있어 두 모델이 맞지 않는다.

### 왜 bucket/texture는 이 문제를 해결하는가

| | per-tile 데이터 | per-tile 변환 | RTT 호환성 |
|---|---|---|---|
| line/fill (bucket) | `tile.buckets[layerId]` — 타일 로컬 [0..EXTENT] 좌표의 geometry VBO | `u_ratio`, `u_matrix` uniform이 `coord` 정보로 계산 | O — `drawLine`이 coords를 iterate하며 tile별 uniform 갱신 |
| raster (texture) | `tile.texture` — 타일 로컬로 이미 rasterize된 픽셀 | UV [0..1] 평면 + mesh quad | O — `drawRaster`가 coords로 tile 찾아 텍스처 바인딩 |
| custom (none) | 없음 — 전역 render 호출 1회 | 전체 맵 MVP만 제공 | X — per-tile 좌표계 없음 |

**bucket이나 texture의 본질은 "per-tile 데이터 + per-tile 변환 규약"이다.** 이 규약 없이 RTT만 도입해도 painter가 "이 FBO에 무엇을 어떻게 그려야 하는지"를 알 수 없다.

### 결론: 세 가지 선택지

| 접근 | per-tile 데이터 | 핵심 변경 | 품질 |
|---|---|---|---|
| **Architecture 1** | `tile.texture` (Custom Source의 `prepare()` FBO 결과) | 코어 수정 없음 | 텍스처 해상도 제약 있음 |
| **Architecture 2** | `tile.buckets[id]` (Custom Source의 `loadTile()` bucket 생성) | 코어 수정 없음 | 내장 vector 완전 동등 |
| **Architecture 3** (가설) | CustomLayer에 per-tile 렌더 계약 추가 | 코어 API 확장 필수 | 완전 자유 + 품질 동등 |

**Architecture 3 (코어 확장안)** 의 최소 변경:

1. `LAYERS_TO_TEXTURES.custom = true` 추가
2. `CustomLayerInterface`에 `renderToTexture?: boolean` 플래그와 `renderTile?(gl, tileID, tileMatrix): void` 메서드 추가 (또는 기존 `render`에 `tileMatrix` 전달)
3. `drawCustom`이 `coords` 파라미터를 받고, RTT 모드일 때 각 coord마다 tile-local 매트릭스를 계산해 `renderTile`을 호출하도록 분기
4. painter의 stencil/clipping/viewport 처리도 tile 단위로 반복

이 경로는 본질적으로 "CustomLayer에 bucket/texture 같은 per-tile 개념을 도입"하는 것이며, **사용자의 직관이 가리키는 진짜 종착지**다. 다만 본 문서의 "코어 수정 없음" 제약을 벗어나므로, Architecture 1/2로 회피하는 것이 단기 전략이다.

### 요약 답변 (사용자 질문에 대한 직접 응답)

- **"벡터 렌더링도 terrain에서 RTT를 수행하나요?"** → 예, line/fill/raster 모두 RTT 대상이며 terrain에 drape됩니다 (`LAYERS_TO_TEXTURES`).
- **"bucket 없이도 동일하게 가능하지 않을까?"** → 원칙적으로는 CustomLayer를 RTT에 넣을 수 있지만, CustomLayer는 per-tile 좌표계 개념이 없어 현재 API로는 왜곡된 결과가 나옵니다. **bucket(또는 texture)은 "per-tile 데이터 + 변환 규약"의 역할**이며 RTT가 그 규약 위에서 동작합니다.
- **단기 해법**: Architecture 1(`tile.texture`) 또는 Architecture 2(`tile.buckets`)가 이 규약을 구현하는 두 가지 방법.
- **장기 이상향**: Architecture 3 (CustomLayer + per-tile API + RTT 화이트리스트 추가) — 코어 수정 동의 시 가장 깔끔.

---

## Architecture 2 — Custom Vector Bucket Source (Tile LOD 독립 1px 정밀도)

내장 `line`/`fill` 레이어의 렌더링 품질(= MapLibre의 네이티브 vector 품질)을 유지하면서 데이터 소스만 커스터마이즈하려는 경우에 사용하는 대안 경로다. `LineBucket`/`FillBucket`을 **메인 스레드에서 직접 구성**하여 `tile.buckets`에 할당하는 방식.

### 가능성 검증 (코드 확인 완료)

- `src/data/bucket/line_bucket.ts:90` — `export class LineBucket implements Bucket` (`@internal` JSDoc 태그지만 실제 export됨 → 경로 import 가능)
- `src/tile/tile.ts:131` — `this.buckets = {}` 초기 할당, 외부에서 직접 할당 가능한 구조
- `src/tile/tile.ts:208-294` — `loadVectorData` / `unloadVectorData` — `tile.buckets`를 설정/해제하는 내부 경로. 커스텀 소스는 이를 우회하여 `tile.buckets = {...}` 직접 대입으로 충분
- `src/tile/tile_manager.ts:228` — `tile.upload(context)`가 프레임마다 호출되어 bucket의 VBO를 GPU에 업로드 (첫 프레임 자동)
- `src/webgl/draw/draw_line.ts:141` — `drawLine`이 `tile.getBucket(layer)`로 bucket을 조회하여 렌더 (소스 타입 무관)
- `src/style/style.ts` — source-type → layer-type 호환성 검증 없음 (sourceLayer 필드만 vector/geojson에서 검증)

즉 "커스텀 타입 소스 + 내장 line 레이어" 조합이 **코어 수정 없이 구조적으로 허용**되며, 소스가 적절한 형식의 bucket을 채워주기만 하면 painter가 정상 렌더한다.

### 구현 개요

```ts
import {LineBucket} from 'maplibre-gl/src/data/bucket/line_bucket';
import {EvaluationParameters} from 'maplibre-gl/src/style/evaluation_parameters';
import {EXTENT} from 'maplibre-gl/src/data/extent';
import type {LineStyleLayer} from 'maplibre-gl/src/style/style_layer/line_style_layer';

export class CustomVectorBucketSource extends Evented implements Source {
  readonly type = 'custom-vector-bucket';
  minzoom = 0; maxzoom = 22; tileSize = 512;
  reparseOverscaled = true;      // vector 전통적 경로 따름
  isTileClipped = true;

  private map!: Map;
  private _loaded = false;
  private _features: IndexedFeature[] = [];     // 사용자 데이터 (전역)

  // ...

  onAdd(map: Map) {
    this.map = map;
    this._loaded = true;
    this.fire(new Event('data', {dataType: 'source', sourceDataType: 'metadata'}));
    this.fire(new Event('data', {dataType: 'source', sourceDataType: 'content'}));
  }

  hasTransition() { return false; }   // vector는 draw-time scaling으로 연속 zoom 대응 → prepare 불필요
  loaded() { return this._loaded; }

  async loadTile(tile: Tile): Promise<void> {
    // (1) 이 소스를 사용하는 모든 line/fill 레이어 수집
    const style = this.map.style;
    const targetLayers = Object.values(style._layers)
      .filter(l => (l as any).source === this.id && (l.type === 'line' || l.type === 'fill'));

    const buckets: {[id: string]: Bucket} = {};
    const canonical = tile.tileID.canonical;
    const overscaling = tile.tileID.overscaleFactor();

    // (2) 레이어 유형별로 Bucket 생성
    for (const layer of targetLayers) {
      if (layer.type === 'line') {
        const bucket = new LineBucket({
          index: 0,
          layers: [layer as LineStyleLayer],
          zoom: tile.tileID.overscaledZ,
          pixelRatio: this.map.getPixelRatio(),
          overscaling,
        });

        // (3) 사용자 벡터 → 타일 로컬 좌표 [0..EXTENT]로 변환된 VectorTileFeature-like 객체로 감싸기
        const indexedFeatures = this._featuresOverlappingTile(tile.tileID)
          .map((feat, i) => ({
            feature: toVectorTileFeatureLike(feat, canonical),    // 사용자 구현
            id: feat.id ?? i,
            index: i,
            sourceLayerIndex: 0,
          }));

        // (4) CPU tessellation 실행 — miter/bevel/cap 생성, layoutVertexArray/indexArray 채움
        bucket.populate(indexedFeatures, {
          featureIndex: /* 필요 시 FeatureIndex 인스턴스 */ null as any,
          iconDependencies: {}, patternDependencies: {}, glyphDependencies: {},
          availableImages: [],
        }, canonical);

        buckets[layer.id] = bucket;
      } else if (layer.type === 'fill') {
        // FillBucket 동일 패턴
      }
    }

    // (5) Tile에 직접 할당 — painter가 다음 프레임부터 자동으로 upload & draw
    tile.buckets = buckets;
    tile.state = 'loaded';

    // 참고: painter.tile.upload(context) 가 첫 draw 전에 자동 호출되어
    //       bucket.upload() → gl.createVertexBuffer → VBO 업로드 수행
  }

  async unloadTile(tile: Tile) {
    for (const id in tile.buckets) tile.buckets[id].destroy();
    tile.buckets = {};
  }

  setData(newFeatures: any[]) {
    this._features = newFeatures;
    this.fire(new Event('data', {dataType: 'source', sourceDataType: 'content'}));   // 모든 타일 reload
  }

  serialize() { return {type: this.type}; }
  hasTile(tileID: OverscaledTileID): boolean { return /* bbox 검사 */ true; }

  private _featuresOverlappingTile(tileID: OverscaledTileID): MyFeature[] { /* ... */ return []; }
}
```

### 사용

```js
await maplibregl.addSourceType('custom-vector-bucket', CustomVectorBucketSource);

map.addSource('rail', { type: 'custom-vector-bucket', /* ... */ });

// 내장 line 레이어 — 모든 내장 paint property 사용 가능
map.addLayer({
  id: 'rail-line',
  type: 'line',
  source: 'rail',
  paint: {
    'line-width': ['interpolate', ['linear'], ['zoom'], 10, 1, 18, 3],
    'line-pattern': 'railway-ties',     // SDF 패턴으로 크로스 타이 표현
    'line-color': '#333',
  }
});

map.setTerrain({source: 'dem', exaggeration: 1.5});   // 자동 drape
```

### Architecture 2의 핵심 장점

1. **MapLibre 내장 라인 렌더링 품질을 그대로 획득**. `line.vertex.glsl`의 `u_ratio * projectLineThickness(zoom)`이 현재 카메라 zoom 기준으로 매 프레임 stroke를 계산하므로, 타일이 z14에서 만들어져도 z22에서 화면상 선 두께가 정확히 유지된다. Architecture 1으로는 prepare 재래스터화를 아무리 정교하게 해도 달성 불가능한 수준.
2. **`prepare()` 불필요**. GPU 셰이더가 zoom을 흡수하므로 매 프레임 CPU 작업이 거의 없다.
3. **Feature 쿼리 가능**. `FeatureIndex`를 함께 구성하면 `queryRenderedFeatures` 네이티브 지원.
4. **paint property expression 자동 적용**. `['interpolate', ['linear'], ['zoom'], ...]`이 내장 평가기로 처리됨.

### Architecture 2의 제약

1. **CPU tessellation을 메인 스레드에서 수행**. 대용량 데이터셋에서는 프레임 드롭 가능. 대안:
   - `setData()` 호출 시점에 미리 전체 데이터 공간 인덱싱 (예: `kdbush`, `rbush`)
   - `loadTile` 내부 작업을 `requestIdleCallback`으로 이월 (단, tile.state 관리 필요)
   - 자체 Web Worker로 tessellation 분리 후 메인 스레드에 transfer
2. **내장 paint property 한계**. 완전히 새로운 셰이더 효과는 불가. 단, `line-pattern` + `line-gradient` + `line-dasharray` 조합만으로도 도메인 표현의 80%는 커버된다.
3. **내부 API 의존**. `LineBucket`, `EvaluationParameters`, `IndexedFeature` 등은 공개 타입이 아니므로 경로 import + 타입 캐스트 필요. MapLibre 메이저 버전 업데이트 시 파손 가능성 있음.
4. **VectorTileFeature-shaped wrapping 필요**. `LineBucket.populate`가 받는 feature는 `{geometry: () => Point[][]}` 형태의 객체여야 하므로 사용자 데이터를 이 인터페이스로 감싸는 어댑터 작성 필요 (참고: `src/util/vectortile_to_geojson.ts`, `src/source/geojson_wrapper.ts`의 `GeoJSONWrapper`).

### 철도 도메인 예시 — 내장 기능만으로 어디까지 가능한가

| 요구사항 | 내장 기능으로 구현 |
|---|---|
| 선로 본선 | `line` 레이어 + `line-width` + `line-color` |
| 크로스 타이 (ties) | `line-pattern`에 SDF 패턴 아틀라스 등록 |
| 광궤/협궤 구분 | feature property `gauge` + `line-width: ['match', ['get', 'gauge'], ...]` |
| 터널/지상 구분 | `line-opacity` expression + 별도 layer |
| 고속/저속 구분 색 | `line-gradient` (다만 speed vs distance 매핑) |
| 신호등, 역 심볼 | `symbol` 레이어 (소스는 `circle` 또는 별도 point source) |
| 철도 활성/비활성 애니메이션 | `line-dasharray` + time-based paint expression |

→ **대부분의 철도 시각화는 Architecture 2로 네이티브 수준 달성 가능**. Architecture 1이 필요한 경우는 "진짜로 내장이 지원하지 않는 쉐이더 효과"(예: 전선의 sag physics 시각화, 열차 실시간 위치에 따른 dynamic glow 등)에 한정됨.

### 요약: 어느 경로를 선택할 것인가

```
사용자 요구 분석
  │
  ├─ 내장 line/fill/circle/symbol paint property로 표현 가능한가?
  │     │
  │     ├─ 예 → Architecture 2 (Custom Vector Bucket)
  │     │        - 내장 vector 품질 완전 획득
  │     │        - Tile LOD 독립 1px 정밀도
  │     │        - terrain drape
  │     │
  │     └─ 아니오 (임의 GLSL 필요) → Architecture 1 (Custom Raster)
  │              - 셰이더 자유도 최대
  │              - 전략 A + D로 1px 정밀도 근사
  │              - terrain drape
  │
  └─ 둘 다 필요 → 하이브리드 소스 (tile.buckets + tile.texture 동시 제공)
```

철도 같은 "타일 LOD와 독립적 정확도가 필수"인 도메인이라면 **Architecture 2가 정답**이며, Architecture 1의 전략 D 재래스터화는 여기에 대한 근사 해법에 불과함을 명시한다.

---

## 글로브 프로젝션 + 지형 — MapLibre의 처리 방식 (★ 중요)

"globe projection 환경에서 terrain이 함께 활성화되면 벡터 데이터는 어떻게 렌더되는가?"라는 질문의 답은 예상 밖이다: **MapLibre는 이 조합을 근본적으로 지원하지 않는다.** 두 렌더링 모드는 **상호 배타적(mutually exclusive)**이며, 이는 벡터든 raster든 본 설계의 Architecture 1/2 모두에 동일하게 적용되는 코어 제약이다.

### 코드 레벨 증거

| 위치 | 내용 |
|---|---|
| `src/render/painter.ts:488` | `isRenderingGlobe: style.projection?.transitionState > 0` — globe 트랜지션 진행 중일 때만 globe 렌더 활성화 |
| `src/render/painter.ts:583-585` | `if (renderOptions.isRenderingGlobe && !this.style.map.terrain) { …render globe depth… }` — **globe sphere depth는 terrain이 비활성일 때만 기록됨** |
| `src/render/painter.ts:584` (주석) | `// There should be no need for explicitly writing tile depths when terrain is enabled.` |
| `src/geo/projection/vertical_perspective_transform.ts:341` (주석) | `// elevation is assumed to be zero - globe rendering must be separate from terrain rendering anyway` |
| `src/webgl/draw/draw_line.ts:206` | `applyGlobeMatrix: !isRenderingToTexture` — RTT 중에는 globe 행렬 적용 불가 |
| `src/webgl/draw/draw_fill.ts:112` | 동일 (`applyGlobeMatrix: !isRenderingToTexture`) |

### 왜 공존이 불가능한가 — 기하학적 모순

- **Globe projection의 가정**: 세계 좌표를 구 표면으로 투영 (`_projection_globe.vertex.glsl:46-76`). `projectTile()`이 mercator(x,y) → 구면각 → 3D 구 표면 좌표 → 스크린 NDC로 변환. **elevation = 0 가정**.
- **Terrain의 가정**: 평면 mercator 평면에 DEM texture 샘플링으로 z축 변위. Per-fragment elevation lookup. **구면이 아닌 평면 기반**.

두 모델이 서로 다른 기하학적 전제에서 출발하므로 한쪽이 활성화되면 다른 쪽 수학이 깨진다. 예: terrain은 "이 (x,y) 위치의 고도" 개념이지만 globe에서 (x,y)는 이미 구면각이라 "고도"의 의미가 달라진다.

### 현재의 렌더링 동작 (globe 활성 시)

`applyGlobeMatrix: !isRenderingToTexture`의 의미를 사례별로 해석:

1. **Terrain OFF + Globe OFF** (표준 mercator):
   - RTT 발생하지 않음 → `isRenderingToTexture = false`
   - `applyGlobeMatrix = true`이지만 globe도 비활성이라 무의미
   - → 일반 mercator 경로

2. **Terrain ON + Globe OFF** (mercator + 3D terrain):
   - line/fill/raster 레이어가 RTT FBO에 기록 → `isRenderingToTexture = true`
   - `applyGlobeMatrix = false` → mercator 행렬로 RTT 그림
   - drawTerrain이 RTT 텍스처를 elevation에 drape
   - → **본 설계(Architecture 1/2)가 정상 작동하는 표준 시나리오**

3. **Terrain OFF + Globe ON** (구 위의 flat vectors):
   - RTT 없음 → `isRenderingToTexture = false`
   - `applyGlobeMatrix = true` → globe 행렬로 직접 draw
   - shader의 `projectTile()`이 구면 투영, `projectLineThickness()`가 위도 보정
   - → 벡터 정상 렌더, 단 elevation 없음

4. **Terrain ON + Globe ON** (모순):
   - `painter.ts:585`의 guard: `if (isRenderingGlobe && !terrain)` — globe depth 기록 생략
   - 실질적으로 terrain이 우선하고 globe 수학은 부분 비활성화
   - **지원되지 않는 상태 — 시각적 결과 미정의**

### Globe 단독 모드에서 1px 정밀도 유지 방식

터미널 모드에서 globe만 활성화된 경우(시나리오 3), 벡터 렌더링 품질은 어떻게 유지되는가? 이는 Architecture 2의 GPU 셰이더 매커니즘이 projection-agnostic하게 설계된 덕분이다.

- **Projection 인터페이스**: `projectTile(vec2)`, `projectLineThickness(float)`, `projectTileWithElevation(vec3)` 시그니처가 mercator/globe 셰이더에서 공통 (`_projection_mercator.vertex.glsl`, `_projection_globe.vertex.glsl`)
- **Line thickness 위도 보정**: globe 셰이더의 `projectLineThickness(tileY) = 1.0 / cos(sphericalLatitude)` — 극지방에 가까울수록 자연스러운 수축을 보상하여 화면상 1px 유지
- **Pole seam 방지**: 특수 Y 값으로 극점 vertex 마킹
  - `rawPos.y < -32767.5` → 북극
  - `rawPos.y > 32766.5` → 남극  
  - Shader가 이를 감지해 구의 극점에 스냅 → 타일 경계 seam 제거
- **LineBucket은 projection-독립**: `layoutVertexArray`가 [0..EXTENT] tile-local 좌표를 저장하므로 buckets는 projection이 바뀌어도 재생성 불필요. 셰이더에서만 분기.

결과: Architecture 2의 bucket은 mercator든 globe든 동일하게 쓸 수 있고, line 셰이더가 projection별로 `projectTile`만 바꿔 호출한다. Tile LOD 독립 1px 정밀도는 두 projection 모두에서 보장된다.

### Architecture 1/2에 대한 함의

| 시나리오 | Architecture 1 (Custom Raster) | Architecture 2 (Custom Vector Bucket) |
|---|---|---|
| Mercator + Terrain | O — `prepare()` FBO → raster RTT → drape | O — bucket + line RTT → drape |
| Mercator 단독 | O | O |
| Globe 단독 | △ — `prepare()`에서 FBO는 mercator 타일 단위로 생성 가능하지만, raster 레이어의 globe 샘플링이 왜곡될 수 있음. `drawRaster`의 globe 처리 확인 필요 | O — 내장 line 레이어가 globe 셰이더 분기를 처리하므로 자동 |
| Globe + Terrain | **X (MapLibre 전체 제약)** | **X (MapLibre 전체 제약)** |

**결론**: 본 설계의 두 Architecture 모두 "globe + terrain 동시"를 극복할 수 없다 — 이는 MapLibre 코어의 근본 제약이기 때문이다. 벡터 소스가 해결한 것은 **"mercator + terrain"** 조합에서의 정확한 1px drape뿐이며, **"globe + terrain"은 MapLibre 자체가 미지원** 영역이다.

### Globe 단독 시 Architecture 1의 추가 고려사항

시나리오 3(terrain OFF + globe ON)에서 Architecture 1을 사용할 때 체크포인트:

- `prepare()`가 mercator 타일 좌표계에서 FBO를 만드는 것은 문제없음 — tile.texture는 본질적으로 tile-local [0..tileSize] 평면 텍스처
- `drawRaster`가 globe 렌더 시 `applyGlobeMatrix: true`로 동작하므로 텍스처 쿼드가 구면에 투영됨
- **단, 텍스처 mapping이 구의 곡면에 펴질 때 LINEAR 필터링의 pole-area distortion이 발생**. mercator→구면 변환은 극지방에서 심하게 왜곡되므로 Architecture 1의 raster 텍스처는 저위도 대비 고위도에서 해상도 손실이 커진다.
- Architecture 2(bucket)는 geometry 자체를 globe 셰이더에서 re-project하므로 이 문제가 없음.

### 글로브 + 고도 효과가 꼭 필요하다면 (워크어라운드)

MapLibre 코어 수정 없이 globe에서 "고도 있는 것처럼 보이는" 시각화가 필요하다면:

1. **CustomLayer + 자체 3D 구 + 자체 elevation sampling** (RTT 미사용) — CustomLayer가 globe projection matrix를 직접 받아 구 위에 3D geometry를 그리되 terrain drape는 자체 구현. 단 MapLibre 터레인 DEM 활용 불가.
2. **Terrain OFF 유지 + 색/셰이딩으로 고도 표현** — 벡터 데이터에 고도 속성을 attribute로 심고 line-color를 고도 함수로 표현. 실제 3D 변위는 없지만 시각적으로 고도감 표현.
3. **Projection 전환** — UX에서 "flat 모드(mercator+terrain)" / "globe 모드(terrain 없음)" 두 모드를 제공하고 사용자가 선택.

이상의 워크어라운드는 본 설계의 스코프를 벗어나므로 참고용 언급만 한다.

---

## 후속 작업 가능성 (스코프 외)

- Architecture 1 승격: `src/source/custom_raster_source_base.ts` 추상 베이스 클래스 (FBO/texture 라이프사이클 + prepare 배치 렌더 + strategy D 자동 관리, 사용자는 `rasterize(gl, tile, zoom)` 만 override)
- Architecture 2 승격: `src/source/custom_vector_bucket_source_base.ts` 추상 베이스 클래스 (bucket 생성 + VectorTileFeature 어댑터 + featureIndex 구성 자동화, 사용자는 `getFeaturesForTile(tileID)` 만 override)
- `CustomLayer`에 `renderToTexture: true` 옵션 추가로 RTT 파이프라인에 직접 합류 (painter 패스 로직 수정 필요 — 이 경우 Architecture 1 자체가 불필요해짐)
- 공식 예제: `test/examples/add-a-custom-vector-bucket-source.html` + `add-a-custom-raster-source.html` 두 가지 산출
- 타입 공개: `LineBucket`, `FillBucket` 등을 `@public` 또는 `/** @experimental */` 태그로 재분류하여 Architecture 2 구현을 안정화된 API로 지원
