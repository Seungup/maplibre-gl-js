# Appendix D — Overscale Strategies

raster 소스 기반의 Arch 1에서 "카메라 줌이 소스 maxzoom을 넘었을 때 1px 정밀도를 유지하는" 문제를 다룬다. 네 가지 전략(A, B, C, D)과 조합 방법.

## 문제 제기

raster 소스의 기본 동작은 **텍스처 확대(magnify)**. 카메라 줌이 소스 `maxzoom`을 넘으면 기존 텍스처가 LINEAR 필터로 늘어나며 흐려진다.

반면 vector 경로(GeoJSON/VectorTile + line/fill)는 `reparseOverscaled = true`로 overscaled 줌마다 bucket을 재조립하고, 셰이더의 `u_ratio`로 fractional zoom까지 대응하여 화면 기준 정확히 1px 선을 유지한다.

Arch 1은 raster 형태이므로 이 정밀도를 자동으로 얻지 못한다. 아래 전략으로 근사한다.

## 메커니즘 확인

| 요소 | 위치 | 역할 |
|---|---|---|
| `reparseOverscaled` 플래그 | `src/source/source.ts:65` | `true`면 각 overscaled 줌마다 별도 loadTile 호출 |
| GeoJSONSource 기본값 | `src/source/geojson_source.ts:167` | `this.reparseOverscaled = true` |
| VectorTileSource 기본값 | `src/source/vector_tile_source.ts:98` | `this.reparseOverscaled = true` |
| RasterTileSource | — | `false` (기본값) |
| coveringTiles 분기 | `src/geo/projection/covering_tiles.ts:272` | `overscaledZ = options.reparseOverscaled ? Math.max(it.zoom, thisTileDesiredZ) : it.zoom` |
| TileManager 전달 | `src/tile/tile_manager.ts:518` | `reparseOverscaled: this._source.reparseOverscaled` |
| 타일 크기 확장 | TileManager 내부 | `new Tile(tileID, tileSize * tileID.overscaleFactor())` |
| `overscaleFactor()` | `src/tile/tile_id.ts:212-214` | `Math.pow(2, overscaledZ - canonical.z)` |

## 전략 A — `maxzoom` 상향 (가장 단순)

```ts
constructor(...) {
  // reparseOverscaled는 기본값(false) 유지
  this.maxzoom = 24;   // 실사용 최대 줌보다 높게
}
```

- 모든 카메라 줌에서 canonical 타일이 새로 요청됨
- overscale 자체가 발생하지 않음
- `loadTile` 내부에서 `tile.tileID.canonical.z`만 참조해 해당 줌 기준으로 1px 선을 그리면 됨
- FBO 크기는 항상 `this.tileSize` — 메모리 안정적
- 단점: 모든 줌 레벨에서 fresh rasterize 발생, 저줌 캐시 재활용 불가

## 전략 B — `reparseOverscaled = true` + overscaled 기반

```ts
constructor(...) {
  this.reparseOverscaled = true;
  this.maxzoom = 16;        // 실제 데이터 소스의 자연 상한
  this._overscaleCap = 4;   // 메모리 방어
}

// loadTile 내부 (Arch 1의 loadTile/prepare 분리 구조 기준)
async loadTile(tile: Tile) {
  const overscaleFactor = tile.tileID.overscaleFactor();
  const effectiveFactor = Math.min(overscaleFactor, this._overscaleCap);
  const size = this.tileSize * effectiveFactor;

  // size × size 텍스처 할당 + 렌더 큐잉
  this._tileSizes.set(tile.tileID.key, size);
  // ...
}

// prepare 내부
prepare() {
  for (const key of toRender) {
    const size = this._tileSizes.get(key) ?? this.tileSize;
    gl.viewport(0, 0, size, size);
    // 셰이더에 overscaledZ uniform 전달 → stroke = desiredPx / overscaleFactor
  }
}
```

- canonical 타일 하나가 overscaled 줌마다 별도 loadTile 호출 수신
- 셰이더 uniform으로 `overscaledZ` 전달하여 해당 줌 기준 1px 계산
- FBO를 `tileSize * effectiveFactor`로 확대하여 LINEAR 샘플링 aliasing 완화

**주의점**:
- `overscaleFactor`가 2^n으로 증가하므로 **상한 캡 필수** (factor 16 → 8192×8192 ≈ 256MB RGBA)
- `_overscaleCap`으로 제한 (권장 4 또는 8)
- TileManager가 `tile.tileSize = tileSize * overscaleFactor()`로 타일 논리 크기를 확장하지만 이는 feature query 용. FBO 크기는 별도 결정 가능.

## 전략 C — 혼합 (권장 기본값)

- 기본 `maxzoom = 22`, `reparseOverscaled = false` (전략 A)
- 대용량 벡터 데이터로 매 줌마다 rasterize 비용이 크면 `maxzoom` 낮추고 `reparseOverscaled = true` (전략 B)
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

## 전략 D — `hasTransition() + prepare()` 기반 연속 zoom 재래스터화 (권장)

전략 A/B는 **정수 줌 변경** 시에만 loadTile을 재호출. 반면 카메라 줌은 연속적(예: 14.37)이므로 같은 타일 내에서 fractional zoom이 변해도 텍스처는 그대로 → 선 두께가 미세하게 스케일되어 1px 엄밀성은 정수 경계에서만 유지.

Vector 경로는 draw-time 셰이더에서 해결 (line-width uniform이 매 프레임 현재 zoom으로 평가). Raster 경로는 draw-time에 stroke를 변경할 수 없으므로 **매 프레임 `prepare()`에서 현재 카메라 zoom 기준으로 FBO 재래스터화**.

### 호출 흐름

1. `Source.hasTransition()` → `TileManager.hasTransition()` (`src/tile/tile_manager.ts:923-929`) → `Style.hasTransitions()` (`src/style/style.ts:687-711`)
2. → `map.ts:3703` `_styleDirty = true` → `map.ts:3720` `triggerRepaint()`
3. 다음 프레임: painter draw 전에 `TileManager.prepare(context)` (`src/tile/tile_manager.ts:221-231`)가 `this._source.prepare()` 실행
4. `prepare()` 시점은 painter의 `drawRaster`로 `tile.texture`가 읽히기 **이전**

### 구현 개요

```ts
class CustomVectorRasterSource implements Source {
  private _lastRasterizedZoom = new Map<string, number>();
  private _zoomEpsilon = 0.1;

  hasTransition(): boolean {
    if (!this.map) return false;
    if (this._pendingRender.size > 0) return true;
    const z = this.map.getZoom();
    for (const [key, baked] of this._lastRasterizedZoom) {
      if (Math.abs(z - baked) > this._zoomEpsilon) return true;
    }
    return false;
  }

  prepare() {
    const currentZoom = this.map.getZoom();
    const toRender: string[] = [];
    for (const key of this._pendingRender) toRender.push(key);
    for (const [key, baked] of this._lastRasterizedZoom) {
      if (this._pendingRender.has(key)) continue;
      if (Math.abs(currentZoom - baked) > this._zoomEpsilon) toRender.push(key);
    }
    // ... 배치 FBO 렌더 ...
    for (const key of toRender) this._lastRasterizedZoom.set(key, currentZoom);
  }
}
```

### 핵심 포인트

1. **조건부 `hasTransition()`** — `_pendingRender` 비어있지 않거나 stale 타일 존재 시에만 `true`. 모든 타일 동기화 시 `false` → `idle` 이벤트 정상 발생.
2. **`_zoomEpsilon` 임계값** — `0.0`이면 매 프레임 재래스터화(고비용), `1.0`이면 정수 줌 변경 시에만. 권장 초기값 `0.1` (zoom 0.1 차이 = 선 두께 약 7%).
3. **FBO 텍스처 핸들 재사용** — `loadTile`에서 할당한 핸들을 `prepare`에서 framebufferTexture2D로 반복 바인딩. GPU 메모리 재할당 없음.
4. **전략 A/B와 직교** — 전략 D는 "연속 zoom에서 선 두께 보정", A/B는 "특정 카메라 zoom에서 충분한 해상도". 권장 조합: **A + D** (`maxzoom=22`로 overscale 회피 + `prepare`로 연속 zoom 보정).
5. **타일 많을 때 성능** — 가시 타일 N개 × 매 프레임 재래스터화 = draw call N개 추가. `_zoomEpsilon` + 프레임당 최대 처리 타일 수 제한으로 완화.

## 전략별 메모리/성능 프로파일

| 전략 | 메모리 | 연속 zoom 품질 | 정수 zoom 전환 비용 | 복잡도 |
|---|---|---|---|---|
| A | 일정 (tileSize²) | 정수 경계 흐림 | 매 줌 재로드 | 낮음 |
| B | `tileSize² × cap²` | 정수 경계 샤프 | overscale 분리 재로드 | 중 |
| C | A 또는 B 선택 | — | — | 낮음 |
| D | 일정 (tileSize²) | 완전 샤프 | 불필요 | 높음 |
| A + D (권장) | 일정 | 완전 샤프 | 매 줌 재로드 (단순) | 중 |

## 검증 체크리스트

- [ ] 전략 A: 카메라 줌 10 → 22 연속 이동 시 각 줌에서 fresh loadTile 확인, 타일 경계 선 두께 일정
- [ ] 전략 B: `maxzoom=14` 설정 후 줌 22 → `canonical.z=14`, `overscaledZ=22`, `overscaleFactor=256`, `overscaleCap=4` 적용되어 FBO 크기 2048로 클램프
- [ ] 동일 데이터를 GeoJSON + line으로 비교 렌더 → 선 두께 ±0.5px 일치
- [ ] terrain 활성화 시 overscale + drape 동시 동작
- [ ] 전략 D: zoom 14.0 → 15.0 연속 드래그 시 GeoJSON 대비 두께 일치
- [ ] `hasTransition()`이 이동 중 `true`, 정지 후 1프레임 내 `false` 복귀, `map.on('idle')` 발생
- [ ] GPU 메모리 안정성: 1분간 zoom in/out 반복 후 증가 없음

## 참고 파일

- `src/source/source.ts:65` — `reparseOverscaled` 플래그
- `src/source/geojson_source.ts:167`, `src/source/vector_tile_source.ts:98` — 참조 구현
- `src/geo/projection/covering_tiles.ts:272` — overscale 분기
- `src/tile/tile_manager.ts:518` — reparseOverscaled 전달
- `src/tile/tile_manager.ts:221-231, 923-929` — prepare/hasTransition
- `src/tile/tile_id.ts:89-138, 212-214` — OverscaledTileID, overscaleFactor
- `src/source/worker_tile.ts:48-55` — vector overscaling layout (참고용)

---

**See also**: [README](../README.md) · [Arch 1](../architectures/arch-1-custom-raster-source.md) · [Appendix C — Vector Pipeline](c-vector-pipeline-reference.md)
