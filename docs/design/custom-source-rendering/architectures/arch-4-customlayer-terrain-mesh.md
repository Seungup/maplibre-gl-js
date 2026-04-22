# Architecture 4 — CustomLayer + Tile Mesh (Official Pattern)

`CustomLayer`의 `render()` 안에서 **`maplibregl.createTileMesh()`** (공개 API)로 per-tile 메시를 생성하고, projection-agnostic 셰이더 프렐루드(`args.shaderData.vertexShaderPrelude`)로 mercator/globe를 자동 처리하여 per-tile 렌더링. 공식 예제 `add-a-custom-layer-with-tiles-to-a-globe.html`이 이 패턴의 reference이다.

## 적합 시나리오

- CustomLayer로 이미 구현된 렌더 로직을 globe + terrain까지 확장
- 임의 GLSL 셰이더 필수
- Arch 3의 코어 fork 비용 회피
- Z-order 제약 수용 가능 (symbols 등이 위에 겹쳐짐)
- RTT 참여는 포기하고 translucent pass에서 per-tile 렌더

## 공식 API (공개, 안정)

| API | 위치 | 용도 |
|---|---|---|
| `maplibregl.createTileMesh(opts, '16bit')` | `src/util/create_tile_mesh.ts:116`, `src/index.ts:382` export | 타일 메시 생성 (공개 안정 API) |
| `CreateTileMeshOptions` | `src/util/create_tile_mesh.ts` | `{granularity, generateBorders, extendToNorthPole, extendToSouthPole}` |
| `map.style.projection.subdivisionGranularity.tile.getGranularityForZoomLevel(z)` | `src/geo/projection/projection.ts:91` (public interface) | 줌 레벨별 적정 subdivision |
| `map.coveringTiles(options)` | `src/ui/map.ts:980` | 가시 타일 enumeration |
| `map.transform.getProjectionData({overscaledTileID, applyGlobeMatrix: true})` | `src/geo/transform.ts` | projection uniform 패키지 (globe 자동) |
| `args.shaderData.vertexShaderPrelude` | `src/webgl/draw/draw_custom.ts:26` | `projectTile`/`projectTileFor3D`/`projectLineThickness` 함수 제공 |
| `args.shaderData.variantName` | 동상 | `'globe'` 또는 `'mercator'` — 현재 projection 식별 |
| `args.shaderData.define` | 동상 | 셰이더 #define (projection 분기용) |

## 표준 uniform 세트

CustomLayer가 `projectTile` 등을 호출하려면 다음 uniform을 바인딩해야 함:

- `u_projection_matrix` — `projectionData.mainMatrix`
- `u_projection_fallback_matrix` — `projectionData.fallbackMatrix` (globe transition 보조)
- `u_projection_clipping_plane` — `projectionData.clippingPlane` (globe far-side clip)
- `u_projection_transition` — `projectionData.projectionTransition` (0..1)
- `u_projection_tile_mercator_coords` — `projectionData.tileMercatorCoords` (tile bbox mercator)

## Mesh 선택 가이드

terrain 활성화 여부에 따라 적절한 mesh 생성 방식이 다르다. 이유: elevation 샘플링은 per-vertex interpolation이므로 충분한 vertex 밀도가 없으면 elevation 왜곡이 발생한다.

### Vertex 밀도 비교

| 방식 | 소스 | vertex per tile (대표값) |
|---|---|---|
| `terrain.getTerrainMesh(tileID)` | `meshSize = 128` 고정 (`src/render/terrain.ts:146`) | **129×129 = 16,641** |
| `createTileMesh` + `subdivisionGranularity.tile` (globe, z=0) | `granularity = 128` | 129×129 = 16,641 |
| `createTileMesh` + `subdivisionGranularity.tile` (globe, z≥3) | `granularity = 32` (min clamp) | 33×33 = **1,089** |
| `createTileMesh` + `subdivisionGranularity.tile` (mercator) | `noSubdivision → granularity = 1` | 2×2 = **4** ← terrain에 불충분 |
| `createTileMesh` + 명시적 `{granularity: 128}` | 호출자 결정 | 129×129 = 16,641 |

`subdivisionGranularity` 공식 (`src/render/subdivision_granularity_settings.ts:36-39`):
```
getGranularityForZoomLevel(z) = max(floor(baseZoomGranularity / (1 << z)), minGranularity, 1)
```

Globe `tile`은 `base=128, min=32`이므로 z≥3에서 32로 고정. Mercator는 모두 `(0, 0)`이므로 항상 1.

### 권장 조합

| 시나리오 | 권장 mesh 소스 | 근거 |
|---|---|---|
| **Terrain ON (임의 projection)** | **`terrain.getTerrainMesh(tileID)`** | elevation 샘플링에 충분한 129×129 격자. 공유 캐싱 자동 |
| Terrain OFF + Globe | `createTileMesh` + `subdivisionGranularity.tile` | 구면 곡률 재현에 32+ 충분, 공개 API 안정성 |
| Terrain OFF + Mercator | `createTileMesh` + `{granularity: 1}` 또는 단순 쿼드 | 평면이므로 2×2로 충분 |
| Terrain OFF, 그러나 per-fragment 효과(gradient 등) 필요 | `createTileMesh` + 명시적 granularity (16~32) | 프래그먼트 정확도 확보 |
| Terrain ON + 자체 mesh 관리 원함 | `createTileMesh` + `{granularity: 128}` | `terrain.getTerrainMesh`와 동등 밀도, 공개 API |

**핵심 원칙**: "terrain이 있으면 terrain이 제공하는 mesh를 그대로 쓰는 것이 가장 안전". `createTileMesh`는 지형이 없을 때 또는 명시적으로 granularity=128로 호출할 때만 사용.

## 두 가지 구현 패턴

Mesh 선택 가이드에 따라 구현은 두 가지 명확한 경로로 분리된다. 두 경로를 **한 Layer 안에 섞지 않고** 각 시나리오를 별개 패턴으로 제시한다. 하이브리드 혼합 코드는 vertex 포맷/stride 차이 때문에 유지보수가 어렵다.

### 패턴 1 — `createTileMesh` 단일 경로 (권장 기본)

Terrain 활성 여부에 관계없이 **항상 `createTileMesh`로 mesh 생성**. granularity만 조건부 결정. 단일 코드 경로, 단일 vertex 포맷, 공개 API만 의존.

```js
const EXTENT = 8192;

const layer = {
  id: 'my-custom-tiles',
  type: 'custom',
  shaderMap: new Map(),
  meshMap: new Map(),

  onAdd(map, gl) { this.map = map; },

  getShader(gl, shaderDescription) {
    if (this.shaderMap.has(shaderDescription.variantName)) {
      return this.shaderMap.get(shaderDescription.variantName);
    }
    const vertexSource = `#version 300 es
      ${shaderDescription.vertexShaderPrelude}
      ${shaderDescription.define}
      in vec2 a_pos;
      out mediump vec2 v_pos;
      void main() {
        gl_Position = projectTile(a_pos);
        v_pos = a_pos / float(${EXTENT});
      }`;
    const fragmentSource = `#version 300 es
      precision mediump float;
      in vec2 v_pos;
      out highp vec4 fragColor;
      void main() { fragColor = vec4(v_pos, 0.0, 0.5); }`;
    const {program, aPos, locations} = compileProgram(gl, vertexSource, fragmentSource);
    const result = {program, aPos, locations};
    this.shaderMap.set(shaderDescription.variantName, result);
    return result;
  },

  getMesh(gl, x, y, z) {
    // granularity 결정:
    // - terrain 활성: 128 (terrain meshSize와 동등, elevation 샘플링 밀도 확보)
    // - terrain 비활성: projection의 subdivisionGranularity 사용 (곡률 재현에 충분)
    const granularity = this.map.terrain
      ? 128
      : this.map.style.projection.subdivisionGranularity.tile.getGranularityForZoomLevel(z);

    const north = y === 0;
    const south = y === (1 << z) - 1;
    const key = `${granularity}_${north}_${south}`;
    if (this.meshMap.has(key)) return this.meshMap.get(key);

    const buffers = maplibregl.createTileMesh({
      granularity,
      generateBorders: false,
      extendToNorthPole: north,
      extendToSouthPole: south,
    }, '16bit');

    const vbo = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, vbo);
    gl.bufferData(gl.ARRAY_BUFFER, buffers.vertices, gl.STATIC_DRAW);
    const ibo = gl.createBuffer();
    gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, ibo);
    gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, buffers.indices, gl.STATIC_DRAW);

    const mesh = {vbo, ibo, indexCount: buffers.indices.byteLength / 2};
    this.meshMap.set(key, mesh);
    return mesh;
  },

  render(gl, args) {
    const {program, aPos, locations} = this.getShader(gl, args.shaderData);
    const isGlobe = args.shaderData.variantName === 'globe';

    gl.useProgram(program);
    gl.enable(gl.BLEND);
    gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA);

    const tiles = this.map.coveringTiles({tileSize: 512});

    for (const tileID of tiles) {
      // `map.coveringTiles()`는 globe 모드에서 antimeridian 타일에
      // 의도적으로 wrap=±1을 할당한다 (GlobeCoveringTilesDetailsProvider.getWrap).
      // 필터링하면 날짜 변경선 영역 타일이 누락되므로 wrap !== 0 필터를 걸지 말 것.

      const proj = this.map.transform.getProjectionData({
        overscaledTileID: tileID,
        applyTerrainMatrix: false,
        applyGlobeMatrix: true,
      });
      gl.uniformMatrix4fv(locations.u_projection_matrix, false, proj.mainMatrix);
      gl.uniformMatrix4fv(locations.u_projection_fallback_matrix, false, proj.fallbackMatrix);
      gl.uniform4f(locations.u_projection_clipping_plane, ...proj.clippingPlane);
      gl.uniform1f(locations.u_projection_transition, proj.projectionTransition);
      gl.uniform4f(locations.u_projection_tile_mercator_coords, ...proj.tileMercatorCoords);

      // 단일 vertex 포맷: Int16 × 2, stride=0 (기본 packed)
      const mesh = this.getMesh(gl, tileID.canonical.x, tileID.canonical.y, tileID.canonical.z);
      gl.bindBuffer(gl.ARRAY_BUFFER, mesh.vbo);
      gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, mesh.ibo);
      gl.enableVertexAttribArray(aPos);
      gl.vertexAttribPointer(aPos, 2, gl.SHORT, false, 0, 0);
      gl.drawElements(gl.TRIANGLES, mesh.indexCount, gl.UNSIGNED_SHORT, 0);
    }
  },
};
```

**장점**:
- 단일 코드 경로, 단일 vertex 포맷(Int16 × 2, stride=0)
- 공개 API(`createTileMesh`) 만 의존. MapLibre 업그레이드 안정
- terrain 유무 분기가 granularity 결정 한 줄에만 존재
- shader·draw call 로직 동일

**단점**:
- terrain 활성 시 terrain이 이미 보유한 mesh와 중복 생성 (메모리 ~100KB per unique key × granularity 수; 실무상 무시 가능)

### 패턴 2 — terrain mesh 직접 재활용 (구조와 주의사항)

`terrain.getTerrainMesh()`가 반환하는 mesh는 **terrain 전용으로 특수 구성된 mesh**라서 단순한 정규 격자가 아니다. 재활용하려면 그 구조를 이해하고 셰이더에서 동일하게 처리해야 한다.

#### 실제 mesh 구조 (`src/render/terrain.ts:433-496` 기준)

Vertex 포맷: `Pos3dArray` = `{a_pos3d: Int16 × 3}` (정의: `src/data/pos3d_attributes.ts`)

3개 섹션이 하나의 VBO에 순차 배치:

1. **Main grid** (129×129 = 16,641 vertex)
   - 좌표: `(x * delta, y * delta, 0)` — z=0
   - delta = EXTENT / 128 = 64
   - 주 렌더 표면

2. **Top/bottom frame** — 타일 경계 stitching용 2×129 = 258 vertex
   - `northY = northPole ? NORTH_POLE_Y(-32768) : 0`, `northZ = northPole ? 0 : 1`
   - `southY = southPole ? SOUTH_POLE_Y(+32767) : EXTENT`, `southZ = southPole ? 0 : 1`
   - **z=1이면 frame vertex** (pole이 아닐 때)

3. **Left/right frame** — 수직 "벽" 516 vertex
   - `for (x of [0, 1]) for (y of [0..128]) for (z of [0, 1]) → (x*EXTENT, y*delta, z)`
   - 같은 (x, y) 좌표에 z=0(메인)과 z=1(frame) 쌍으로 ribbon 생성

전체 vertex 수: 16,641 + 258 + 516 ≈ **17,415**. **SegmentVector는 하나**(`SegmentVector.simpleSegment(0, 0, vertexArray.length, indexArray.length)`)로 메인+프레임을 한 번의 draw call로 처리.

#### z (frame-bit)의 역할 — 필수 이해 포인트

Terrain vertex shader (`src/shaders/glsl/terrain.vertex.glsl`) 핵심:

```glsl
in vec3 a_pos3d;
uniform float u_ele_delta;

void main() {
  float ele = get_elevation(a_pos3d.xy);
  float ele_delta = a_pos3d.z == 1.0 ? u_ele_delta : 0.0;
  gl_Position = projectTileFor3D(a_pos3d.xy, ele - ele_delta);
  // ...
}
```

- `a_pos3d.z == 1.0`인 frame vertex는 elevation을 `u_ele_delta` 만큼 **아래로 내림**
- `u_ele_delta = terrain.getMeshFrameDelta(zoom) = 2πR / 2^zoom / 5` (`terrain.ts:505-508`)
- 이 수직 offset이 **서로 다른 zoom 레벨의 인접 타일 간 elevation seam을 가림** (zoom mismatch stitching)
- Pole frame은 z=0으로 예외 처리 (이미 pole 좌표 자체가 투영됨)

**따라서 terrain mesh를 custom layer에서 재활용하면서 elevation을 사용한다면, frame-bit 처리를 셰이더에서 동일하게 수행해야 한다**. 그렇지 않으면 zoom 전환 시 cross-tile seam이 발생.

#### 두 가지 사용 시나리오

##### 시나리오 2A — 2D projection only (elevation 사용 안 함)

단순 tile 격자로만 활용. z=1 frame vertex는 (x, y) 위치가 메인 grid와 동일하므로 `projectTile`이 같은 픽셀로 투영 → 화면상 degenerate 효과, 시각적 아티팩트 없음.

```glsl
// Vertex shader
in vec3 a_pos3d;   // 3 컴포넌트로 받되 xy만 사용
out vec2 v_pos;
void main() {
  gl_Position = projectTile(a_pos3d.xy);
  v_pos = a_pos3d.xy / 8192.0;
}
```

```js
// Binding
gl.bindBuffer(gl.ARRAY_BUFFER, mesh.vertexBuffer.buffer);
gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, mesh.indexBuffer.buffer);
gl.enableVertexAttribArray(aPos3d);
gl.vertexAttribPointer(aPos3d, 3, gl.SHORT, false, 0, 0); // Int16 × 3, stride packed
gl.drawElements(
  gl.TRIANGLES,
  mesh.segments.get()[0].primitiveLength * 3,   // primitiveLength = 삼각형 수
  gl.UNSIGNED_SHORT, 0
);
```

> Pos3dArray stride는 `createLayout`의 4-byte 정렬에 따라 6 또는 8 바이트. 안전을 위해 `stride=0`으로 두면 GPU가 attribute size(3 × 2 = 6)로 자동 추론.

##### 시나리오 2B — elevation 적용 drape (terrain 표면과 정렬)

Terrain 표면과 완전히 동일한 elevation + seam 처리가 필요한 경우. 터레인 셰이더 패턴을 그대로 따라야 함.

```glsl
in vec3 a_pos3d;
uniform float u_ele_delta;       // = terrain.getMeshFrameDelta(zoom)
uniform sampler2D u_terrain;     // DEM 텍스처 (terrain.getTerrainData)
uniform mat4 u_terrain_matrix;
uniform vec4 u_terrain_unpack;
uniform float u_terrain_dim;
uniform float u_terrain_exaggeration;

// _prelude.vertex.glsl:146-166에서 복제한 get_elevation
float get_elevation(vec2 pos) { /* ... */ }

void main() {
  float ele = get_elevation(a_pos3d.xy);
  float ele_delta = a_pos3d.z == 1.0 ? u_ele_delta : 0.0;
  gl_Position = projectTileFor3D(a_pos3d.xy, ele - ele_delta);
}
```

```js
// render() 내부
const td = terrain.getTerrainData(tileID);
const eleDelta = terrain.getMeshFrameDelta(this.map.getZoom());
gl.uniform1f(locations.u_ele_delta, eleDelta);
gl.activeTexture(gl.TEXTURE0 + 2);
gl.bindTexture(gl.TEXTURE_2D, td.texture.texture);
gl.uniform1i(locations.u_terrain, 2);
gl.uniformMatrix4fv(locations.u_terrain_matrix, false, td.u_terrain_matrix);
gl.uniform1f(locations.u_terrain_dim, td.u_terrain_dim);
gl.uniform4fv(locations.u_terrain_unpack, td.u_terrain_unpack);
gl.uniform1f(locations.u_terrain_exaggeration, td.u_terrain_exaggeration);
// ... projection uniforms + 바인딩/드로우
```

**결과**: 커스텀 layer가 terrain 표면과 **완전히 같은 geometry**로 렌더됨. cross-tile seam 처리까지 terrain과 일치.

#### 패턴 2 전체 render 스켈레톤 (시나리오 2B 기준)

```js
render(gl, args) {
  const {program, aPos3d, locations} = this.getShader(gl, args.shaderData);
  const terrain = this.map.terrain;
  if (!terrain) { this._renderFlat(gl, args); return; }   // 폴백

  const isGlobe = args.shaderData.variantName === 'globe';
  const eleDelta = terrain.getMeshFrameDelta(this.map.getZoom());

  gl.useProgram(program);
  gl.enable(gl.DEPTH_TEST);
  gl.enable(gl.BLEND);
  gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA);
  gl.uniform1f(locations.u_ele_delta, eleDelta);

  for (const tile of terrain.tileManager.getRenderableTiles()) {
    const tileID = tile.tileID;
    // `terrain.tileManager.getRenderableTiles()`는 내부적으로 `coveringTiles`를
    // 호출하며, globe 모드에서 antimeridian 타일에 wrap=±1을 할당하여 반환한다.
    // 필터링하면 날짜 변경선 영역이 누락되므로 wrap !== 0 필터를 걸지 말 것.

    const mesh = terrain.getTerrainMesh(tileID);
    const td = terrain.getTerrainData(tileID);
    const proj = this.map.transform.getProjectionData({
      overscaledTileID: tileID,
      applyTerrainMatrix: false,     // 우리가 직접 get_elevation 사용
      applyGlobeMatrix: true,
    });

    // Projection uniforms
    gl.uniformMatrix4fv(locations.u_projection_matrix, false, proj.mainMatrix);
    gl.uniformMatrix4fv(locations.u_projection_fallback_matrix, false, proj.fallbackMatrix);
    gl.uniform4f(locations.u_projection_clipping_plane, ...proj.clippingPlane);
    gl.uniform1f(locations.u_projection_transition, proj.projectionTransition);
    gl.uniform4f(locations.u_projection_tile_mercator_coords, ...proj.tileMercatorCoords);

    // Terrain DEM uniforms
    gl.activeTexture(gl.TEXTURE0 + 2);
    gl.bindTexture(gl.TEXTURE_2D, td.texture.texture);
    gl.uniform1i(locations.u_terrain, 2);
    gl.uniformMatrix4fv(locations.u_terrain_matrix, false, td.u_terrain_matrix);
    gl.uniform1f(locations.u_terrain_dim, td.u_terrain_dim);
    gl.uniform4fv(locations.u_terrain_unpack, td.u_terrain_unpack);
    gl.uniform1f(locations.u_terrain_exaggeration, td.u_terrain_exaggeration);

    // Mesh 바인딩 — @internal API
    gl.bindBuffer(gl.ARRAY_BUFFER, mesh.vertexBuffer.buffer);
    gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, mesh.indexBuffer.buffer);
    gl.enableVertexAttribArray(aPos3d);
    gl.vertexAttribPointer(aPos3d, 3, gl.SHORT, false, 0, 0);   // Int16 × 3, packed
    gl.drawElements(
      gl.TRIANGLES,
      mesh.segments.get()[0].primitiveLength * 3,
      gl.UNSIGNED_SHORT, 0
    );
  }
}
```

#### 패턴 2 정리

**언제 쓰나**:
- Custom layer 결과를 terrain 표면과 **픽셀 단위로 정확히 정렬** 필요 (공동 렌더링, overlay mask 등)
- 메모리 중복 없음 (terrain이 이미 만든 mesh 재사용)
- Cross-tile seam 처리까지 terrain과 일치시켜야 할 때

**비용**:
- `@internal` API 4개 의존: `getTerrainMesh`, `getTerrainData`, `getMeshFrameDelta`, `mesh.vertexBuffer.buffer`/`indexBuffer.buffer`/`segments.get()`
- 셰이더가 terrain 패턴(`a_pos3d.z == 1.0 ? u_ele_delta : 0.0`)을 정확히 복제해야 함
- MapLibre 마이너 업그레이드 시 이 5가지 지점 모두 검증

**terrain OFF 폴백**: 패턴 1의 `createTileMesh` 경로로 전환.

### 패턴 선택 기준

- **패턴 1 (기본)**: API 안정성·유지보수 우선. 대부분의 실무.
- **패턴 2**: terrain elevation 정렬 완벽성·메모리 최소화가 필수일 때. 군사/임베디드 등.

**필수 uniform 리스트** (두 패턴 공통):
```js
const uniforms = [
  'u_projection_matrix', 'u_projection_fallback_matrix',
  'u_projection_clipping_plane', 'u_projection_transition',
  'u_projection_tile_mercator_coords',
];
```

## Globe Antimeridian Wrap 처리

Globe 모드에서 `map.coveringTiles()` 및 `terrain.tileManager.getRenderableTiles()`는 **antimeridian 근처 타일에 `wrap=±1`을 의도적으로 할당**하여 반환한다. 이는 버그가 아니라 정상 동작이며, 사용자 render 루프에서 임의로 필터링하면 날짜 변경선 영역이 렌더되지 않아 "구멍"이 생긴다.

### 메커니즘

`GlobeCoveringTilesDetailsProvider` (`src/geo/projection/globe_covering_tiles_details_provider.ts`)의 두 특성이 조합되어 이 동작을 만든다.

**`allowWorldCopies(): false`** (line 104-106):

```ts
allowWorldCopies(): boolean {
    return false;
}
```

`covering_tiles.ts:218-226`의 인공적 world copy 루프(`wrap ∈ {-3..+3}`)가 **실행되지 않는다**. Globe traversal은 단일 `newRootTile(0)`에서만 시작. Mercator의 `renderWorldCopies` 패턴과 다름.

**`getWrap(centerCoord, tileID, _parentWrap)`** (line 83-98):

```ts
const distanceCurrent = distanceToTileSimple(centerCoord.x, tileX, tileMercatorSize);
const distanceLeft    = distanceToTileSimple(centerCoord.x, tileX - 1.0, tileMercatorSize);
const distanceRight   = distanceToTileSimple(centerCoord.x, tileX + 1.0, tileMercatorSize);
const distanceSmallest = Math.min(distanceCurrent, distanceLeft, distanceRight);
if (distanceSmallest === distanceRight) return 1;
if (distanceSmallest === distanceLeft) return -1;
return 0;
```

각 canonical 타일에 대해 **camera center 기준 최근접 wrap** (0, -1, +1 중 하나)을 반환. `covering_tiles.ts:262` `it.wrap = detailsProvider.getWrap(...)`로 OverscaledTileID에 반영된다 (`_parentWrap`은 globe에서 무시).

### 시나리오 예시

카메라가 lng=170° 부근, 카메라 center mercator x≈0.944:

- Canonical tile `{z: 2, x: 0}` (lng ≈ -180° 근처, antimeridian 동쪽에 위치)
- `distanceCurrent = 0.694`, `distanceLeft = 1.694`, **`distanceRight = 0.056`**
- `getWrap` 반환 = `+1` → `OverscaledTileID(z=2, wrap=+1, x=0, ...)`
- TerrainTileManager는 이를 `tileID.key`(wrap 포함)로 저장. `getRenderableTiles()`도 이 wrap=+1 타일을 포함해 반환.

만약 사용자 render 루프가 `if (isGlobe && tileID.wrap !== 0) continue`로 필터링하면 이 타일이 skip되어 lng≈±180° 경계 영역의 픽셀이 비게 된다.

### Globe 셰이더의 wrap 불변성

`_projection_globe.vertex.glsl:47-54`:

```glsl
vec2 mercator_pos = u_projection_tile_mercator_coords.xy
                  + u_projection_tile_mercator_coords.zw * translatedPos;
vec2 spherical;
spherical.x = mercator_pos.x * PI * 2.0 + PI;
```

`spherical.x`는 2π 주기 삼각함수로 들어가므로 mercator x=0과 x=1은 동일 sphere 위치(antimeridian)로 투영된다. **같은 canonical 타일을 wrap=0 또는 wrap=+1로 그려도 최종 픽셀은 동일하다.** 단 render 자체가 빠지면 해당 canonical 영역은 채워지지 않는다.

### 공식 예제와의 차이

`test/examples/add-a-custom-layer-with-tiles-to-a-globe.html:72-76`는 `coveringTiles` API를 쓰지 않고 **static하게** 3-copy(wrap={-1,0,+1})를 직접 생성한다:

```js
for (let i = -1; i <= 1; i++) {
    generateTileList(tilesToRender, {x: 0, y: 0, z: 0, wrap: i});
}
```

이 static list 맥락에서는 mercator는 3 copy 모두, globe는 중복이므로 `wrap !== 0` skip이 옳다.

**그러나 `coveringTiles` 또는 `terrain.tileManager.getRenderableTiles()` API의 반환값에는 이 필터가 오동작을 일으킨다.** API가 이미 globe 중복 방지(`allowWorldCopies: false`)를 끝낸 뒤 wrap을 의도적으로 할당한 것이므로 추가 필터링은 잘못된 제거가 된다.

### 선택적 중복 방지 (canonical 단위 dedup)

`coveringTiles`와 `getRenderableTiles`는 이미 canonical 단위 중복을 제거하므로 일반적인 경우 안전 장치는 불필요하다. 방어적 코드가 필요하다면 canonical 단위로 dedup한다:

```js
const seen = new Set();
for (const tileID of tiles) {
  const canonKey = `${tileID.canonical.z}/${tileID.canonical.x}/${tileID.canonical.y}`;
  if (seen.has(canonKey)) continue;
  seen.add(canonKey);
  // draw
}
```

`wrap !== 0` 기반 필터는 절대 사용하지 말 것.

## Terrain Elevation 확장

공식 예제는 terrain elevation을 다루지 않는다. 지형 drape 효과를 위해 `map.terrain.getTerrainData(tileID)`를 함께 호출하여 DEM 텍스처를 샘플링한다. 단 이 API는 `@internal`.

### 추가 uniform 및 셰이더

```glsl
// Vertex shader 확장
${shaderDescription.vertexShaderPrelude}
${shaderDescription.define}

in vec2 a_pos;
uniform mat4 u_terrain_matrix;
uniform sampler2D u_terrain;
uniform float u_terrain_dim;
uniform vec4 u_terrain_unpack;
uniform float u_terrain_exaggeration;

// _prelude.vertex.glsl:146-166에서 복제
float get_elevation(vec2 pos) {
  vec2 coord = (u_terrain_matrix * vec4(pos, 0.0, 1.0)).xy * u_terrain_dim + 1.0;
  vec2 f = fract(coord);
  vec2 c = (floor(coord) + 0.5) / (u_terrain_dim + 2.0);
  float d = 1.0 / (u_terrain_dim + 2.0);
  // bilinear sample + unpack (실제 unpack은 u_terrain_unpack 공식 참조)
  // ...
  return sampled * u_terrain_exaggeration;
}

void main() {
  float ele = get_elevation(a_pos);
  // projectTileFor3D이 mercator/globe 모두 elevation 포함 투영 처리
  gl_Position = projectTileFor3D(a_pos, ele);
}
```

### 렌더 루프에 terrain 바인딩 추가

```js
render(gl, args) {
  const terrain = this.map.terrain;
  // ... 공통 setup

  for (const tileID of tiles) {
    const projectionData = this.map.transform.getProjectionData({
      overscaledTileID: tileID,
      applyTerrainMatrix: false,    // 우리가 직접 get_elevation 호출
      applyGlobeMatrix: true,
    });

    // projection uniforms 바인딩
    // ...

    // terrain DEM 바인딩 (terrain 활성 시만)
    if (terrain) {
      const td = terrain.getTerrainData(tileID);
      gl.activeTexture(gl.TEXTURE0 + 2);
      gl.bindTexture(gl.TEXTURE_2D, td.texture.texture);
      gl.uniform1i(locations.u_terrain, 2);
      gl.uniformMatrix4fv(locations.u_terrain_matrix, false, td.u_terrain_matrix);
      gl.uniform1f(locations.u_terrain_dim, td.u_terrain_dim);
      gl.uniform4fv(locations.u_terrain_unpack, td.u_terrain_unpack);
      gl.uniform1f(locations.u_terrain_exaggeration, td.u_terrain_exaggeration);
    }

    // mesh draw
  }
}
```

## 핵심 이점

1. **공식 API 기반**. `createTileMesh` + projection prelude 는 공개 안정 API. 마이너 업그레이드 파손 위험 낮음.
2. **Globe 자동 대응**. `vertexShaderPrelude`가 projection-specific `projectTile`/`projectTileFor3D`를 자동 제공 → mercator/globe 셰이더 분기 불필요.
3. **Pole / seam 처리 내장**. `extendToNorthPole`/`extendToSouthPole` 플래그 + `projectTile(pos, rawPos)`의 pole vertex 감지.
4. **Subdivision granularity 공식 지원**. `map.style.projection.subdivisionGranularity.tile.getGranularityForZoomLevel(z)`로 줌별 적정 분할.
5. **Terrain elevation 확장 가능**. `terrain.getTerrainData` + `get_elevation` 셰이더 함수 복제로 drape 추가 (단 terrain API는 `@internal`).

## 제약 및 주의점

### 1. RTT 스택 미참여

`drawCustom`은 translucent pass에서 호출됨. 기본 terrain 합성 이후 **그 위에 덧그려짐**. 심볼이 커스텀 렌더 위에 얹히지 않음 → depth/stencil로 조정 필요. [Arch 3](arch-3-customlayer-rtt-fork.md) 참조 (RTT 필수 시).

### 2. 1px 정밀도

메시가 polygon 단위이므로 line/fill 효과는 fragment shader 구현. 내장 line의 `u_ratio × projectLineThickness` 스케일링을 직접 구현해야 동등 정밀도 확보.

### 3. `@internal` API 의존 (terrain elevation 확장 시만)

`terrain.getTerrainData`는 `@internal`. drape 효과가 필요 없으면 이 의존성 불필요.

### 4. Terrain 비활성 시 폴백

`map.terrain === null` 체크 후 elevation=0 경로 또는 셰이더 분기 (`#ifdef NO_TERRAIN`).

### 5. 타일 경계 seam

`generateBorders: true`로 메시에 border 추가 후 stencil 기반 2-pass 드로우로 seam 제거 가능 (공식 예제 주석 참조).

### 6. queryRenderedFeatures 미지원

CustomLayer 기본 한계. 자체 공간 인덱스 유지 필요.

## 공식 예제 참조

- `test/examples/add-a-custom-layer-with-tiles-to-a-globe.html` — 기본 globe + tile 렌더 (reference)
- `test/examples/add-a-simple-custom-layer-on-a-globe.html` — 단순 버전
- `test/examples/display-a-globe-with-a-fill-extrusion-layer.html` — extrusion 예제 (내장 레이어)

## Testing

- `createTileMesh`의 출력 검증 (vertex/index 버퍼 크기)
- 공식 예제 구조를 기반으로 smoke test
- globe 전환 시 mesh 생성/캐시 동작 확인
- terrain on/off 양쪽 경로 렌더 결과 검증

## Debugging

- Spector.js로 frame capture, uniform 바인딩 확인
- `shaderData.variantName` 로깅 (mercator/globe 전환 감지)
- 타일 경계 시각화: `generateBorders: true` + 디버그 색상
- 셰이더 컴파일 에러: prelude include 확인

## Performance

- `meshMap` 캐시로 동일 `(granularity, N-pole, S-pole, border)` 키 재사용
- `shaderMap` 캐시로 variantName별 program 재사용
- 가시 타일 수 × `drawElements` 1회 = 낮은 프레임 비용
- globe에서 far-side 타일은 `clippingPlane`으로 자동 컬링

## Memory

- 메시 캐시 크기 제한 (granularity별 최대 N개)
- `onRemove`에서 VBO/IBO/program 해제

## 검증 체크리스트

- [ ] `createTileMesh`로 생성한 mesh가 mercator 모드에서 정상 렌더
- [ ] globe 모드 전환 시 자동 구면 투영 (`vertexShaderPrelude` 효과)
- [ ] 북/남극 타일에서 pole seam 없음 (`extendToNorthPole` 등 적용)
- [ ] 타일 경계에 가시적 seam 없음 (또는 `generateBorders` + stencil로 해결)
- [ ] Globe transition 중간값(0.0/0.5/1.0)에서 흔들림 없음
- [ ] (terrain 확장 시) terrain ON/OFF 양쪽 정상 동작
- [ ] Z-order: 심볼이 custom 위에 정상 렌더
- [ ] GPU memory 증가 없음

## 참고 파일

- `src/util/create_tile_mesh.ts:116` — `createTileMesh` 공개 함수
- `src/index.ts:382` — export
- `src/geo/projection/projection.ts:91` — `subdivisionGranularity` 인터페이스
- `src/webgl/draw/draw_custom.ts:26` — prelude 생성 지점
- `src/shaders/glsl/_projection_mercator.vertex.glsl`, `_projection_globe.vertex.glsl` — projection 함수 정의
- `src/shaders/glsl/_prelude.vertex.glsl:146-166` — `get_elevation` (터레인 확장 복제용)
- `src/render/terrain.ts:264` — `getTerrainData` (@internal)

---

**See also**: [README](../README.md) · [Appendix B — Globe + Terrain](../appendix/b-globe-terrain-interaction.md) · [Use Cases — Military](../use-cases/military-rtt-layers.md) · [FAQ](../faq.md)
