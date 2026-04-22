# Appendix B — Globe Projection + Terrain Interaction

MapLibre는 globe projection과 terrain을 **layered rendering** 구조로 결합한다. 표면적으로 두 모드가 상호 배타적으로 보이는 코드 조각들이 있으나, 실제 흐름은 두 모드가 자연스럽게 공존한다.

## Layered Rendering의 핵심 두 단계

### Stage 1 — RTT pass (mercator 평면 합성)

RTT 중에는 globe 행렬이 적용되지 않는다:

```ts
// src/webgl/draw/draw_line.ts:206
applyGlobeMatrix: !isRenderingToTexture,
applyTerrainMatrix: true

// src/webgl/draw/draw_fill.ts:112 — 동일
```

- line/fill 레이어가 per-tile FBO에 **mercator 투영**으로 기록됨
- 결과: 타일 FBO에 평면 mercator-투영된 벡터 픽셀
- 이 시점에는 globe도 elevation도 적용되지 않음 — 순수 mercator 타일 텍스처

### Stage 2 — `drawTerrain`이 terrain mesh를 globe + elevation으로 투영

```ts
// src/webgl/draw/draw_terrain.ts:93
const projectionData = tr.getProjectionData({
  overscaledTileID: tile.tileID,
  applyTerrainMatrix: false,    // terrain mesh가 이미 elevation 포함
  applyGlobeMatrix: true,        // terrain mesh를 globe 좌표계로 투영
});
program.draw(context, gl.TRIANGLES, depthMode, StencilMode.disabled, colorMode,
             CullFaceMode.backCCW, uniformValues, terrainData, projectionData,
             'terrain', mesh.vertexBuffer, mesh.indexBuffer, mesh.segments);
```

- terrain mesh 정점이 DEM elevation을 가짐
- **globe projection**으로 mesh를 구 표면에 투영
- Stage 1의 RTT 텍스처를 **mesh에 sampling** → 구면 + 고도 표면에 벡터 drape

## 왜 이 구조가 자연스러운가

- 타일 텍스처는 본질적으로 평면 `[0..tileSize]² ` 픽셀 격자 — 어떤 projection이든 타일 단위로 잘게 나뉘면 각 타일 내부는 거의 평면으로 근사 가능
- RTT는 그 평면 텍스처를 만든다 (globe 여부와 무관)
- 최종 합성 단계에서 mesh 투영 방식에 따라 평면 텍스처가 평면/구면에 휘감긴다

"**per-tile 평면 텍스처 + projection-aware mesh**"의 분리가 globe + terrain 공존을 가능하게 한다.

## 오해되기 쉬운 코드의 실제 의미

### `painter.ts:583-585`

```ts
// Render the globe sphere into the depth buffer - but only if globe is enabled and terrain is disabled.
// There should be no need for explicitly writing tile depths when terrain is enabled.
if (renderOptions.isRenderingGlobe && !this.style.map.terrain) {
  this._renderTilesDepthBuffer();
}
```

- globe 단독 모드는 far-side z-clipping을 위해 globe sphere를 depth buffer에 그려야 함
- terrain이 활성이면 **terrain mesh가 이미 depth 정보를 제공**하므로 sphere depth 별도 기록 불필요
- 두 모드의 비활성화가 아니라 **depth 책임의 이양**

### `vertical_perspective_transform.ts:341`

```ts
// elevation is assumed to be zero - globe rendering must be separate from terrain rendering anyway
```

- 카메라 → globe 중심 거리(수평선 클리핑 plane 결정용) 계산 블록 내부 주석
- 이 특정 수학에서만 elevation=0으로 가정
- 전체 시스템 차원의 제약이 아님

### `draw_line.ts:206`, `draw_fill.ts:112`의 `applyGlobeMatrix: !isRenderingToTexture`

- RTT 중에는 globe 행렬 미적용 (→ mercator 평면 FBO 생성)
- `drawTerrain`에서 globe 행렬 적용 (→ mesh를 구면에 투영)
- 두 단계의 역할 분리

## GlobeProjection facade 구조

`src/geo/projection/globe_projection.ts`의 `GlobeProjection`은 `transitionState`(0~1)에 따라 `MercatorProjection`과 `VerticalPerspectiveProjection`을 위임한다.

- `transitionState = 0`: 순수 mercator
- `transitionState = 1`: 순수 vertical perspective (구면 투영)
- `0 < transitionState < 1`: 두 행렬을 보간 (전환 애니메이션)

`drawTerrain`의 `applyGlobeMatrix: true`는 결과적으로 이 vertical perspective transform 행렬을 mesh 정점에 적용한다.

## 네 Architecture의 globe + terrain 동작

| 시나리오 | Arch 1 | Arch 2 | Arch 3 | Arch 4 |
|---|---|---|---|---|
| Mercator + Terrain | 예 | 예 | 예 (fork 후) | 예 |
| Mercator 단독 | 예 | 예 | 예 | 예 |
| Globe 단독 | 예¹ | 예 | 예 | 예 |
| **Globe + Terrain** | **예** | **예** | **예 (fork 후)** | **예** |

¹ Arch 1의 Globe 단독: `tile.texture`는 mercator 평면이므로 `drawRaster`가 globe 셰이더로 구면에 매핑 시 극지방에서 LINEAR 필터 왜곡 누적. 저위도 무손실, 고위도 해상도 손실.

### Arch 1의 Globe + Terrain 흐름

1. `Source.prepare()`: mercator 타일 좌표계에서 FBO에 사용자 벡터 래스터화 → `tile.texture`
2. `drawRaster` (RTT 중): `applyGlobeMatrix: false`로 mercator RTT FBO에 텍스처 쿼드 복사
3. `drawTerrain`: `applyGlobeMatrix: true` + DEM elevation으로 terrain mesh 투영, RTT FBO를 텍스처로 샘플링
4. 결과: 구면 elevation 표면에 사용자 벡터가 정확히 drape

### Arch 2의 Globe + Terrain 흐름

1. `Source.loadTile()`: `tile.buckets`에 LineBucket 생성 (mercator tile-local `[0..EXTENT]`)
2. `drawLine` (RTT 중): `applyGlobeMatrix: false`로 mercator 셰이더로 RTT FBO에 라인 기록
3. `drawTerrain`: 동일
4. 결과: globe + terrain 위에 픽셀 완벽한 1px 라인

### Arch 4의 Globe + Terrain 흐름 (mesh 조건부 선택)

1. `CustomLayer.render(gl, args)`에서 `map.coveringTiles()` 또는 자체 enumeration으로 가시 타일 목록 획득
2. Mesh 선택 — **terrain 활성 여부에 따라 분기**:
   - **Terrain ON**: `map.terrain.getTerrainMesh(tileID)` 사용 (`@internal`, 129×129=16,641 vertex, elevation 샘플링에 적합)
   - **Terrain OFF**: `maplibregl.createTileMesh({granularity, extendToNorthPole, extendToSouthPole}, '16bit')` 사용 (공개 API, projection 곡률만 고려)
     - granularity는 `map.style.projection.subdivisionGranularity.tile.getGranularityForZoomLevel(z)`
3. `map.transform.getProjectionData({overscaledTileID, applyGlobeMatrix: true, applyTerrainMatrix: false})`로 projection uniform 세트 획득 (`mainMatrix`, `fallbackMatrix`, `clippingPlane`, `projectionTransition`, `tileMercatorCoords`)
4. 셰이더는 `args.shaderData.vertexShaderPrelude`가 자동 제공하는 `projectTile(a_pos)` 또는 `projectTileFor3D(a_pos, ele)` 호출 — mercator/globe 분기 불필요
5. Terrain drape 필요 시 `map.terrain.getTerrainData(tileID)`(@internal)로 DEM 텍스처/행렬 획득, 셰이더에서 `get_elevation(a_pos)` 호출 (함수는 `_prelude.vertex.glsl:146-166`에서 수동 복제)
6. 결과: 구면/평면 + (선택적) elevation 위에 사용자 도메인 렌더

**왜 terrain 활성 시 `createTileMesh`만 쓰면 안 되는가**: `subdivisionGranularity.tile`은 projection 곡률용이지 DEM 해상도용이 아니다. Mercator는 2×2=4 vertex, globe z≥3는 33×33=1,089 vertex만 생성. Elevation은 per-vertex interpolation이므로 타일 내부 변화가 평탄화된다. Terrain 활성 시에는 terrain 내부 meshSize=128 mesh(129×129=16,641 vertex)를 재사용해야 DEM 해상도 재현 가능.

### Globe 전환 애니메이션 (transitionState 중간값)

- 두 행렬 보간이 자동 수행됨
- 커스텀 소스/레이어는 특별 처리 불필요 — `getProjectionData` 결과가 자동으로 보간된 행렬 반환
- 애니메이션 중 타일 재로드 없음 (mercator tile 좌표계 유지)

## Vertical Perspective Projection이란?

MapLibre의 "globe" 모드 내부 구현체. 세계 좌표를 구면 각도(spherical angle)로 변환한 뒤 구 표면에 투영하여 스크린 NDC로 매핑. 정의:

- `src/geo/projection/vertical_perspective_projection.ts`
- `src/geo/projection/vertical_perspective_transform.ts`
- 셰이더: `src/shaders/glsl/_projection_globe.vertex.glsl`

Mercator 셰이더와 동일한 함수 시그니처(`projectTile`, `projectLineThickness`, `projectTileWithElevation`)를 구현하여 상위 레이어 셰이더가 projection-agnostic하게 작성될 수 있다.

## 참고 파일

- `src/webgl/draw/draw_terrain.ts:93` — terrain mesh에 `applyGlobeMatrix: true`
- `src/webgl/draw/draw_line.ts:206`, `draw_fill.ts:112` — RTT 중 `applyGlobeMatrix: false`
- `src/render/painter.ts:583-585` — globe sphere depth vs terrain depth
- `src/geo/projection/globe_projection.ts` — facade
- `src/geo/projection/vertical_perspective_transform.ts:335-360` — 카메라 계산 주석

---

**See also**: [README](../README.md) · [Glossary](../glossary.md) · [Appendix A — RTT Mechanism](a-rtt-mechanism.md)
