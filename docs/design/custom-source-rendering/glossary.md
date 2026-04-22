# Glossary

본 문서 세트에서 사용되는 용어와 MapLibre 내부 상수·API의 간단 정의. 각 항목은 소스 위치를 포함한다.

## 타일 좌표계

**EXTENT**
타일 로컬 좌표계의 해상도 상수. 값 `8192`. 모든 타일 내부 좌표는 `[0, EXTENT]² ` 범위의 정수로 표현된다. 정의: `src/data/extent.ts`.

**CanonicalTileID**
표준 타일 좌표. `{z, x, y}` 세 정수로 구성. mercator 격자의 고정된 위치. 정의: `src/tile/tile_id.ts`.

**OverscaledTileID**
카메라 줌이 소스 maxzoom을 넘을 때 사용되는 확장 타일 ID. `canonical` 필드(CanonicalTileID)와 `overscaledZ`(요청된 줌 레벨) 조합. 정의: `src/tile/tile_id.ts:89-138`.

**overscaleFactor()**
`OverscaledTileID`의 메서드. `Math.pow(2, overscaledZ - canonical.z)` 반환. overscale 시 타일이 시각적으로 몇 배 확대되었는지를 나타낸다. 정의: `src/tile/tile_id.ts:212-214`.

## 렌더링 파이프라인

**RTT (Render-To-Texture)**
레이어를 per-tile FBO에 먼저 그린 뒤 terrain mesh에 텍스처로 매핑하여 drape 효과를 만드는 2단계 렌더링. MapLibre의 `RenderToTexture` 클래스가 관리. 정의: `src/webgl/render_to_texture.ts`.

**FBO (Framebuffer Object)**
오프스크린 렌더 대상. 여러 텍스처를 color/depth attachment로 바인딩하여 GL 그리기의 결과를 텍스처에 기록.

**LAYERS_TO_TEXTURES**
RTT 파이프라인에 참여하는 레이어 타입 화이트리스트. 현재 `{background, fill, line, raster, hillshade, color-relief}`. `custom`은 미포함. 정의: `src/webgl/render_to_texture.ts:16-23`.

**ctx.bindFramebuffer.dirty**
MapLibre `Context` 래퍼가 캐시한 framebuffer 바인딩 상태를 무효화하는 플래그. 외부 코드가 `gl.bindFramebuffer`를 직접 호출한 뒤에는 `true`로 설정해야 다음 painter draw가 올바른 FBO에 기록된다. 정의: `src/webgl/context.ts`.

## 셰이더 / 프로젝션

**u_ratio**
line 셰이더의 uniform. `ratioScale / pixelsToTileUnits(tile, 1, camera.zoom)` 공식. 타일 좌표계 단위를 화면 픽셀로 변환하는 스케일. 매 프레임 현재 카메라 줌 기반으로 업데이트되어 1px 정밀도를 보장. 정의: `src/webgl/program/line_program.ts:127`.

**projectTile(vec2)**
projection-specific 셰이더 함수. 타일 로컬 좌표 → clip-space NDC. mercator와 globe가 각각 구현. 정의: `src/shaders/glsl/_projection_mercator.vertex.glsl`, `src/shaders/glsl/_projection_globe.vertex.glsl`.

**projectLineThickness(tileY)**
line 두께 보정 함수. mercator에서는 상수 `1.0`, globe에서는 `1.0 / cos(sphericalLatitude)`. 극지방에서의 자연 수축을 보상. 정의: 각 projection 셰이더.

**get_elevation(vec2 pos)**
DEM 텍스처에서 elevation을 샘플링하는 셰이더 함수. `u_terrain_matrix`로 변환한 좌표에서 bilinear sampling 후 `u_terrain_exaggeration` 적용. 정의: `src/shaders/glsl/_prelude.vertex.glsl:146-166`.

**projectTileFor3D(vec2 pos, float ele)**
elevation을 포함한 3D 타일 투영. terrain 렌더링과 globe 모드에서 사용. 정의: `_prelude.vertex.glsl` + projection별 구현.

## 프로젝션

**VerticalPerspectiveProjection / VerticalPerspectiveTransform**
MapLibre의 globe 모드 구현체. 세계 좌표를 구면 각도로 변환 후 구 표면에 투영. 정의: `src/geo/projection/vertical_perspective_projection.ts`, `src/geo/projection/vertical_perspective_transform.ts`.

**GlobeProjection**
`MercatorProjection`과 `VerticalPerspectiveProjection` 사이를 위임하는 facade. `transitionState`에 따라 분기. 정의: `src/geo/projection/globe_projection.ts`.

**transitionState**
globe ↔ mercator 전환 애니메이션의 현재 상태. `0`이면 순수 mercator, `1`이면 순수 vertical perspective, 0~1 사이는 행렬 보간. 정의: `GlobeProjection`.

## 공식 Custom Layer 헬퍼

**createTileMesh**
타일 mesh 생성 공개 helper. 시그니처: `createTileMesh(options: CreateTileMeshOptions, forceIndicesSize?: '16bit'|'32bit') → {vertices, indices, uses32bitIndices}`. Int16 vertex 배열(`a_pos` SHORT × 2) + Uint16/Uint32 index 배열 반환. 정의: `src/util/create_tile_mesh.ts:116`, export: `src/index.ts:382`. 공식 예제: `test/examples/add-a-custom-layer-with-tiles-to-a-globe.html`.

**CreateTileMeshOptions**
`{granularity?: number, generateBorders?: boolean, extendToNorthPole?: boolean, extendToSouthPole?: boolean}`. `generateBorders: true`는 타일 경계 seam 방지용 border mesh 추가. pole 플래그는 globe 모드에서 구의 극점 커버용. 정의: `src/util/create_tile_mesh.ts`.

**subdivisionGranularity**
Projection interface의 공개 getter. `map.style.projection.subdivisionGranularity.tile.getGranularityForZoomLevel(z)` 호출로 줌 레벨별 적정 mesh 분할 횟수 획득. mercator는 저 granularity, globe는 고 granularity 반환. 정의: `src/geo/projection/projection.ts:91`.

**vertexShaderPrelude**
CustomLayer의 `render(gl, args)`에서 `args.shaderData.vertexShaderPrelude`로 접근. 현재 projection에 맞는 `projectTile`/`projectTileFor3D`/`projectLineThickness` 등 함수를 셰이더에 자동 제공. mercator/globe 분기 불필요. 생성 지점: `src/webgl/draw/draw_custom.ts:26`.

**projectTileFor3D**
셰이더 prelude 함수. `(vec2 posInTile, float elevation) → vec4 clipSpace`. mercator에서는 `projectTileWithElevation`과 동일, globe에서는 구면 투영 + elevation 오프셋 적용. Terrain drape 렌더링의 표준 함수.

## 빌드·타입

**@internal**
JSDoc 태그. "공개 API가 아님"을 표시. 접근은 가능하지만 마이너 버전 업그레이드에서 파손 가능. 본 문서에서 `@internal`로 표시된 API를 사용하는 경로(Arch 2, Arch 4)는 MapLibre 버전마다 검증 필요.

**saveTileTexture pool**
`painter.saveTileTexture(texture)` / `getTileTexture(size)`. 동일 크기 텍스처를 재사용하기 위한 풀. 일반 raster tile에서 사용. FBO attachment 이력이 있는 텍스처의 재사용은 권장하지 않음. 정의: `src/render/painter.ts`.

## 군사 도메인 (use-cases/military-rtt-layers.md에서 사용)

**MIL-STD-2525**
미군 합동 군사 심볼 표준. 단위·무기·활동·의도를 기호화. 맵 위 심볼 레이어로 렌더링.

**APP-6**
NATO 군사 심볼 표준. MIL-STD-2525와 상호 호환 가능한 구조.

**MGRS (Military Grid Reference System)**
군사용 격자 좌표 표현. 지도에 오버레이로 표시되는 경우 있음.

**Fog of War**
미탐색·미관측 영역을 가리는 반투명 마스크. 도메인 데이터에 따라 유동적으로 갱신됨.

**Threat Envelope**
무기(예: SAM) 사거리 범위를 지면 footprint 또는 3D 볼륨으로 표현.
