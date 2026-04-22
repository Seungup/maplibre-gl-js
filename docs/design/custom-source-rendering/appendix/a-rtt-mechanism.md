# Appendix A — RTT Mechanism

MapLibre의 `RenderToTexture` 파이프라인이 per-tile 렌더를 어떻게 수행하는지, 그리고 그 구조가 왜 "per-tile 데이터 컨테이너"를 필요로 하는지 분석한다. 네 Architecture가 이 규약을 어떻게 만족하는지 정리한다.

## RTT per-tile 렌더 루프

핵심 코드 (`src/webgl/render_to_texture.ts:140-205`):

```ts
renderLayer(layer: StyleLayer, renderOptions: RenderOptions): boolean {
  // 스택 누적 단계 생략...
  if (LAYERS_TO_TEXTURES[this._prevType] || (LAYERS_TO_TEXTURES[type] && isLastLayer)) {
    for (const tile of this._renderableTiles) {
      // (1) 타일용 FBO 획득 및 바인딩
      painter.context.bindFramebuffer.set(obj.fbo.framebuffer);
      painter.context.clear({color: Color.transparent, stencil: 0});

      for (const layerId of layers) {
        const layer = painter.style._layers[layerId];
        const coords = layer.source
          ? this._coordsAscending[layer.source][tile.tileID.key]
          : [tile.tileID];

        painter.context.viewport.set([0, 0, obj.fbo.width, obj.fbo.height]);
        painter._renderTileClippingMasks(layer, coords, true);

        // (2) coords 전달하여 per-tile 렌더 수행
        painter.renderLayer(painter, painter.style.tileManagers[layer.source],
                            layer, coords, options);
      }
    }
    drawTerrain(this.painter, this.terrain, this._rttTiles, options);
  }
}
```

핵심 관찰:

1. **RTT는 per-tile 루프** — 각 terrain 타일의 FBO를 바인딩하고, 그 타일이 커버하는 소스 타일(`coords`)을 전달하여 레이어를 그린다.
2. **`coords`는 `OverscaledTileID[]`** — 한 terrain 타일이 여러 소스 타일로부터 합성될 수 있음.
3. **각 draw 함수는 `coords`를 받아 per-tile 렌더를 수행** — `drawLine`, `drawFill`, `drawRaster` 모두 동일한 시그니처.

## 레이어 타입별 렌더 동작 차이

| 레이어 타입 | draw 함수 | `coords` 처리 | per-tile 데이터 |
|---|---|---|---|
| `line` | `drawLine(painter, tm, layer, coords, options)` | `for (const coord of coords) { tile = tm.getTile(coord); bucket = tile.getBucket(layer); renderBucket(bucket) }` | `tile.buckets[layerId]` |
| `fill` | `drawFill(...)` | 동일 패턴 | `tile.buckets[layerId]` |
| `raster` | `drawRaster(...)` | 동일 패턴 | `tile.texture` |
| `custom` | `drawCustom(painter, tm, layer, options)` | **`coords` 파라미터 없음** (`src/webgl/draw/draw_custom.ts:8`) | 없음 |

`drawCustom`은 `implementation.render(gl, customLayerArgs)`를 **한 번**만 호출한다. `customLayerArgs.modelViewProjectionMatrix`는 **전체 맵의 MVP**이지 tile-local 행렬이 아니다.

## 왜 per-tile 데이터 컨테이너가 필요한가

RTT가 타일용 FBO를 바인딩하고 viewport를 타일 크기로 설정한 뒤 레이어를 그릴 때, 레이어는 **"이 타일에 해당하는 지오메트리·픽셀만"** 선택적으로 그려야 한다. 그러려면:

- 타일별로 분리된 데이터(geometry 또는 texture)
- 타일 로컬 좌표계에서 작동하는 변환 규약

이 두 가지를 제공하는 구조가 bucket(vector)과 texture(raster)다. CustomLayer는 둘 다 없으므로 `LAYERS_TO_TEXTURES`에 `custom: true`를 추가하는 것만으로는 올바르게 동작하지 않는다.

### CustomLayer를 단순히 화이트리스트에 추가했을 때의 시뮬레이션

1. RTT가 타일 FBO 바인딩, viewport `(0, 0, tileSize, tileSize)`.
2. `drawCustom` 호출. `coords`는 무시됨.
3. `implementation.render`가 **전체 맵 MVP**로 draw.
4. 결과: 전체 세계가 타일 크기 FBO에 압축되어 기록됨 → 왜곡된 픽셀.

이것이 `LAYERS_TO_TEXTURES`에 `custom`이 빠져있는 구조적 이유다.

## 네 Architecture의 per-tile 규약 충족 방식

| Arch | per-tile 데이터 | per-tile 변환 | RTT 호환성 |
|---|---|---|---|
| 1 | `tile.texture` (prepare에서 FBO로 래스터화) | UV [0..1], mesh quad | 예 — `drawRaster`가 coords iterate |
| 2 | `tile.buckets[id]` (loadTile에서 생성) | `u_ratio`, `u_matrix` uniform | 예 — `drawLine`이 coords iterate |
| 3 | 없음 (사용자가 `renderTile` 훅으로 주입) | 사용자가 `tileMatrix` 파라미터로 받음 | 예 (fork 후) — `drawCustom`이 coords iterate로 수정됨 |
| 4 | 없음 (터레인의 mesh + DEM 차용) | `get_elevation`으로 per-vertex elevation | **아니오** — translucent 패스에서 덧그리기 |

## Arch 1의 `prepare()`가 벡터 RTT 렌더와 등가인 이유

벡터 경로:

```
Frame:
  RenderToTexture.renderLayer(line_layer)
    for (each terrain tile):
      bind FBO_terrainTile
      for (each source coord):
        drawLine → bucket VBO → FBO_terrainTile에 line 픽셀 기록
    drawTerrain → FBO_terrainTile을 terrain mesh에 drape
```

커스텀 raster 경로:

```
Frame:
  TileManager.prepare()
    Source.prepare()
      bind FBO_custom (우리 FBO)
      for (each pending/stale tile):
        framebufferTexture2D(tile.texture)
        사용자 셰이더로 draw → tile.texture에 기록
  painter.render()
    RenderToTexture.renderLayer(raster_layer)
      for (each terrain tile):
        bind FBO_terrainTile
        for (each source coord):
          drawRaster → tile.texture 바인딩 → 텍스처 쿼드 → FBO_terrainTile에 복사
      drawTerrain → drape
```

두 경로의 본질적 차이: **1-pass (벡터) vs 2-pass (커스텀 raster)**. 추가되는 패스는 타일당 쿼드 1개 복사(6 vertex)로, GPU 비용은 microsecond 단위이다. 교환으로 얻는 이점:

- `prepare()` 타이밍은 painter 진입 전으로 결정적
- `tile.texture` 라이프사이클을 소스가 직접 관리 (RTT pool 재활용과 독립)
- zoom 변화 없으면 `tile.texture`를 여러 프레임 재사용

## 레이어 순서와 RTT 스택

`RenderToTexture.renderLayer`는 인접한 RTT-대상 레이어들을 하나의 "스택"에 누적한 뒤 함께 그린다 (`src/webgl/render_to_texture.ts:148-157`). 비-RTT 레이어(예: `symbol`)가 끼면 스택이 분리된다.

```
Style order: [base-raster, fill-1, line-1, symbol-1, fill-2]
            └── Stack A ──────┘ └──── Stack B
```

사용자가 style 순서로 합성 순서를 제어 가능. Arch 1/2 레이어 여러 개를 같은 스택에 묶어 한 번에 terrain으로 drape할 수 있다.

## 참고 파일

- `src/webgl/render_to_texture.ts` — 전체 RTT 구현
- `src/webgl/draw/draw_line.ts` — line 렌더 (coords 활용)
- `src/webgl/draw/draw_raster.ts` — raster 렌더
- `src/webgl/draw/draw_custom.ts` — custom 렌더 (coords 미수신)
- `src/render/painter.ts:658-690` — renderLayer dispatch

---

**See also**: [README](../README.md) · [Glossary](../glossary.md) · [Appendix B — Globe + Terrain](b-globe-terrain-interaction.md)
