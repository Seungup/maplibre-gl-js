# Architecture 1 — Custom Raster Source

`addSourceType`으로 등록한 커스텀 소스가 `loadTile`에서 빈 WebGL 텍스처를 할당하고, `prepare()`에서 FBO에 사용자 셰이더로 래스터화하여 `tile.texture`를 채운다. 내장 `raster` 스타일 레이어가 이 텍스처를 drape 한다.

## 적합 시나리오

- 커스텀 셰이더 효과가 필요 (fog of war, 커버리지 그라데이션, 위협 엔벨로프, 힐셰이드 응용)
- 내장 paint property로 표현 불가능한 렌더링
- RTT 참여 필수 (symbols 등 다른 레이어와의 올바른 z-order)
- 커스텀 데이터 포맷 또는 동적 생성 로직

## 핵심 원칙 — loadTile/prepare 분리

`loadTile`은 GL 텍스처 할당 + 렌더 큐잉까지만 수행하고, 모든 FBO 쓰기는 `prepare()`에 집중시킨다. 이유:

1. **결정론적 타이밍** — `prepare()`는 `TileManager.prepare(context)` (`src/tile/tile_manager.ts:221-231`)를 통해 painter draw 직전 호출. `loadTile`은 Promise 기반이라 resolve 타이밍이 painter 중간일 수 있어 GL 상태 충돌 위험.
2. **배치 amortization** — GL 상태 save/restore를 `prepare()` 진입/종료 시 1회만. 타일 N개 = 1 save + N draw + 1 restore.
3. **코드 경로 통합** — "초기 래스터화"와 "연속 zoom 재래스터화"(전략 D)가 동일 루틴 공유.
4. **멱등성** — 같은 타일에 `loadTile` 중복 호출되어도 큐에 플래그만 세팅.
5. **비동기 안전성** — `loadTile`이 painter render 중 resolve되어도 실제 GL draw는 다음 `prepare()`까지 지연.

## 타일 라이프사이클

```
TileManager.loadTile(tile)
  ↓
Source.loadTile(tile):
  gl.createTexture()  →  빈 텍스처 핸들
  Texture 래퍼로 wrap → tile.texture에 부착
  _pendingRender.add(tile.tileID.key)   (큐잉)
  tile.state = 'loaded'
  Promise.resolve()
  ↓
(다음 프레임 시작)
  ↓
TileManager.prepare(context) → Source.prepare():
  GL 상태 백업 [1회]
  for each tile in (_pendingRender ∪ staleZoomTiles):
    framebufferTexture2D(tile.texture.texture)
    viewport, clear, 사용자 셰이더로 draw
    _lastRasterizedZoom.set(key, currentZoom)
  GL 상태 복원 [1회]
  ctx.bindFramebuffer.dirty = true
  _pendingRender.clear()
  ↓
painter.renderLayer(raster layer)
  → drawRaster(tile.texture)  [RTT FBO에 텍스처 쿼드 복사]
```

첫 프레임 blank 노출 문제 없음: render 순서 `style.prepare → painter.render`이므로 loadTile 완료 후 같은 프레임 prepare에서 렌더된다.

## 클래스 스켈레톤

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

  private map!: Map;
  private _loaded = false;
  private _fbo: WebGLFramebuffer | null = null;
  private _program!: WebGLProgram;
  private _userData: any;

  private _pendingRender = new Set<string>();
  private _lastRasterizedZoom = new Map<string, number>();
  private _tilesByKey = new Map<string, Tile>();
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

  hasTransition(): boolean {
    if (this._pendingRender.size > 0) return true;
    if (!this.map) return false;
    const z = this.map.getZoom();
    for (const [, baked] of this._lastRasterizedZoom) {
      if (Math.abs(z - baked) > this._zoomEpsilon) return true;
    }
    return false;
  }

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
    this._pendingRender.add(key);
    tile.state = 'loaded';
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

  prepare(): void {
    if (!this.map || !this._fbo) return;

    const currentZoom = this.map.getZoom();
    const toRender: string[] = [];

    for (const key of this._pendingRender) toRender.push(key);
    for (const [key, baked] of this._lastRasterizedZoom) {
      if (this._pendingRender.has(key)) continue;
      if (Math.abs(currentZoom - baked) > this._zoomEpsilon) toRender.push(key);
    }

    if (toRender.length === 0) return;

    const ctx = this.map.painter.context;
    const gl = ctx.gl;
    const size = this.tileSize;

    const prevViewport = gl.getParameter(gl.VIEWPORT) as Int32Array;
    const prevFB = gl.getParameter(gl.FRAMEBUFFER_BINDING);
    gl.bindFramebuffer(gl.FRAMEBUFFER, this._fbo);
    gl.viewport(0, 0, size, size);
    gl.useProgram(this._program);

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
      gl.bindFramebuffer(gl.FRAMEBUFFER, prevFB);
      gl.viewport(prevViewport[0], prevViewport[1], prevViewport[2], prevViewport[3]);
      ctx.bindFramebuffer.dirty = true;
      this._pendingRender.clear();
    }
  }

  setData(newData: any) {
    this._userData = newData;
    for (const key of this._tilesByKey.keys()) this._pendingRender.add(key);
    this.fire(new Event('data', {dataType: 'source', sourceDataType: 'content'}));
  }

  private _rasterizeTile(gl: WebGL2RenderingContext, tile: Tile, currentZoom: number) {
    // (a) tile.tileID.canonical → mercator bbox
    // (b) currentZoom uniform 전달 (stroke = desiredPx / 2^(currentZoom - canonical.z))
    // (c) this._userData를 tile-local [0..EXTENT]로 변환해 draw
  }
}
```

## 등록 및 사용

```js
await maplibregl.addSourceType('custom-vector-raster', CustomVectorRasterSource);

map.addSource('my-vec', {
  type: 'custom-vector-raster',
  data: { /* GeoJSON 등 */ },
  tileSize: 512,
});

map.addLayer({
  id: 'my-vec-layer',
  type: 'raster',           // RTT 자동 참여
  source: 'my-vec',
  paint: { 'raster-opacity': 1.0 }
});

map.setTerrain({source: 'dem', exaggeration: 1.5});   // 자동 drape
```

## Overscale 지원

상세 전략은 [Appendix D](../appendix/d-overscale-strategies.md) 참조. 권장 조합은 **전략 A + D** (`maxzoom=22` + `prepare()` 기반 연속 zoom 재래스터화).

## 위험 요소와 주의점

| 항목 | 주의 |
|---|---|
| painter GL 상태 오염 | `prepare()` 진입 시 viewport/FBO/program/blend/depth 모두 백업, 종료 시 복원 |
| `ctx.bindFramebuffer.dirty` | Context 래퍼가 캐시한 바인딩 상태 무효화. `prepare()` 종료 시 필수 |
| `loadTile`에서 FBO 렌더 금지 | loadTile은 `createTexture` + `texImage2D`만. FBO 쓰기는 prepare에서만 |
| 첫 프레임 blank 리스크 | 실제로는 발생 안 함. render 순서가 prepare → painter이므로 |
| 텍스처 메모리 누수 | `unloadTile`/`onRemove`에서 `gl.deleteTexture()`. painter pool 미사용 |
| 타일 좌표 변환 | 재사용 utility 없음. `EXTENT`와 `tileID.canonical`로 직접 계산 |
| TypeScript 타입 | `addSource` specification 유니온에 커스텀 타입 없음. `as any` 또는 모듈 augmentation |
| Worker 미사용 | WebGL은 메인 스레드 한정. 데이터 전처리는 Worker 가능 |
| overscale 메모리 (전략 B) | `overscaleCap` 없으면 텍스처 크기 2^n 폭증. 반드시 cap 지정 |
| TileManager 논리 타일 크기 | overscale 시 TileManager는 `tile.tileSize * overscaleFactor()` 설정. 우리 FBO 크기는 별도 결정 |
| `hasTransition()` 과도 활성화 | 정적 `true` 반환 시 `idle` 이벤트 미발생 + 배터리 소모. 조건부 반환 필수 |
| `prepare()` 예외 처리 | try/catch + finally로 GL 상태 복원 보장 |
| `_tilesByKey` 일관성 | loadTile/unloadTile이 정확히 추가/삭제, onRemove에서 전체 정리 |
| prepare 성능 | draw call = `_pendingRender.size + staleCount`. 프레임당 max N개 제한으로 완화 |

## Testing

- `map.painter.context` 모킹: headless-gl 또는 mock WebGL 컨텍스트
- `map.terrain` null/non-null 양쪽 경로 테스트
- FBO `readPixels`로 출력 스냅샷 검증
- `loadTile` → `prepare` 수동 트리거 harness

## Debugging

- Spector.js로 frame capture, FBO 바인딩 추적
- `gl.getError()`를 `prepare` 말미에 `console.assert(gl.getError() === 0)`
- `_pendingRender.size` 로깅으로 재렌더 빈도 추적
- FBO 내용 시각화: 화면 모서리에 임시 쿼드로 디버그 오버레이

## Performance Profiling

- Chrome DevTools GPU Timeline
- `performance.mark('prepare-start')` / `performance.measure(...)`
- 프레임 예산: 60fps 기준 16.67ms, 전술 디스플레이는 5-8ms 권장
- 가시 타일 N × 타일당 draw call 추정

## Memory Management

- `painter.saveTileTexture` pool **사용 금지** (FBO attachment 이력 텍스처는 재사용 부적합)
- `onRemove`에서 모든 텍스처/FBO/program 해제
- Leak 감지: 타일 생성/해제 카운터 유지

## Worker 활용

- Worker에서 사용자 데이터 파싱, 공간 인덱싱(`kdbush`/`rbush`), feature 필터링 수행 가능
- Worker → main transfer는 `ArrayBuffer` transferable 활용
- `loadTile`에서 Worker에 `postMessage({tileID})`, 응답받은 데이터를 내부 상태에 저장
- 실제 FBO 쓰기는 메인 스레드 `prepare()`에서만

## 검증 체크리스트

- [ ] 개발 서버 기동 (`npm run start`)
- [ ] PoC HTML에서 단순 폴리곤 렌더 확인
- [ ] terrain OFF, 팬/줌 — 타일 경계 seam 없음
- [ ] terrain ON + `setTerrain({source: 'dem', exaggeration: 1.5})` — elevation 따라 휨
- [ ] `gl.getError() === 0` 확인
- [ ] unloadTile 호출 확인 (팬으로 화면 밖으로)
- [ ] GPU 메모리 증가 없음 (Chrome DevTools Memory → GPU)
- [ ] `source.setData(newGeo)` 후 1-2 프레임 내 재래스터화
- [ ] 전략 D: zoom 14.0 → 15.0 연속 드래그 시 GeoJSON + line 레이어와 선 두께 일치

## 참고 파일

- `src/source/source.ts:31-198` — `Source` 인터페이스, `addSourceType`
- `src/source/raster_tile_source.ts:182-230` — `loadTile`/`unloadTile` 참조 패턴
- `src/render/texture.ts` — `Texture` 클래스
- `src/webgl/draw/draw_raster.ts:39-145` — `tile.texture` 사용 방식
- `src/webgl/render_to_texture.ts:16-23` — RTT 화이트리스트
- `src/tile/tile_manager.ts:221-231, 923-929` — `prepare()`/`hasTransition()` 흐름
- `src/style/style.ts:687-711` — `Style.hasTransitions()` 집계
- `src/ui/map.ts:3703, 3720` — `_styleDirty` + `triggerRepaint`
- `src/data/extent.ts` — `EXTENT = 8192`

---

**See also**: [README](../README.md) · [Appendix A — RTT](../appendix/a-rtt-mechanism.md) · [Appendix D — Overscale](../appendix/d-overscale-strategies.md) · [FAQ](../faq.md)
