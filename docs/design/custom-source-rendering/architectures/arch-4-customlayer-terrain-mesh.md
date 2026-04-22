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

## Mesh 선택 가이드 (★ 중요 — vertex 밀도 차이)

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
      if (isGlobe && tileID.wrap !== 0) continue;

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

### 패턴 2 — terrain mesh 직접 재활용 (메모리 최적화 원할 때)

Terrain 활성 시 `terrain.getTerrainMesh()` mesh 인스턴스를 그대로 **참조만** 해서 draw. mesh 중복 생성 제거. 단 `@internal` API 의존 + Pos3dArray(stride=8) vertex 포맷 처리 필요.

```js
onAdd(map, gl) { this.map = map; },

render(gl, args) {
  const {program, aPos, locations} = this.getShader(gl, args.shaderData);
  const isGlobe = args.shaderData.variantName === 'globe';
  const terrain = this.map.terrain;

  gl.useProgram(program);
  // ... blend 등 setup

  const tiles = terrain
    ? terrain.tileManager.getRenderableTiles().map(t => t.tileID)
    : this.map.coveringTiles({tileSize: 512});

  for (const tileID of tiles) {
    if (isGlobe && !terrain && tileID.wrap !== 0) continue;

    const proj = this.map.transform.getProjectionData({
      overscaledTileID: tileID,
      applyTerrainMatrix: false,
      applyGlobeMatrix: true,
    });
    // uniforms 바인딩 (패턴 1과 동일)

    if (terrain) {
      // @internal API 사용: terrain mesh는 Pos3dArray (x, y, frame-bit) 3컴포넌트, stride=8
      // VertexBuffer/IndexBuffer 래퍼 접근도 @internal
      const mesh = terrain.getTerrainMesh(tileID);
      gl.bindBuffer(gl.ARRAY_BUFFER, mesh.vertexBuffer.buffer);
      gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, mesh.indexBuffer.buffer);
      gl.enableVertexAttribArray(aPos);
      // stride=8이지만 a_pos는 처음 2컴포넌트(x,y)만 읽음. 3번째 frame-bit는 무시
      gl.vertexAttribPointer(aPos, 2, gl.SHORT, false, 8, 0);
      gl.drawElements(
        gl.TRIANGLES,
        mesh.segments.get()[0].primitiveLength * 3,
        gl.UNSIGNED_SHORT,
        0
      );
    } else {
      // 폴백: createTileMesh 경로 (패턴 1의 getMesh 사용)
      const mesh = this.getMesh(gl, tileID.canonical.x, tileID.canonical.y, tileID.canonical.z);
      gl.bindBuffer(gl.ARRAY_BUFFER, mesh.vbo);
      gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, mesh.ibo);
      gl.enableVertexAttribArray(aPos);
      gl.vertexAttribPointer(aPos, 2, gl.SHORT, false, 0, 0);
      gl.drawElements(gl.TRIANGLES, mesh.indexCount, gl.UNSIGNED_SHORT, 0);
    }
  }
}
```

**장점**:
- terrain이 이미 만들어둔 mesh를 참조만, 메모리 중복 없음
- terrain의 pole/frame vertex가 그대로 활용됨 (stitching 유리)
- terrain의 `tileManager.getRenderableTiles()`로 타일 목록도 공유

**단점**:
- `@internal` API 의존: `mesh.vertexBuffer.buffer`, `mesh.indexBuffer.buffer`, `mesh.segments.get()` 모두 내부 타입
- 두 개의 vertex 포맷/stride 경로를 layer 코드가 관리해야 함
- MapLibre 마이너 업그레이드 시 검증 필요

### 선택 권장

| 우선순위 | 권장 패턴 |
|---|---|
| MapLibre 독립성, API 안정성 | **패턴 1** (`createTileMesh` 단일) |
| 메모리 최소화, terrain 심도 통합 | 패턴 2 (terrain mesh 참조) |
| 대부분의 실무 | **패턴 1** 기본값, 필요 시 2로 전환 |

군사/임베디드 환경에서 메모리가 타이트해 중복 mesh가 부담이면 패턴 2 고려. 그 외에는 패턴 1이 유지보수와 버전 호환성 면에서 우수.

**필수 uniform 리스트** (두 패턴 공통):
```js
const uniforms = [
  'u_projection_matrix', 'u_projection_fallback_matrix',
  'u_projection_clipping_plane', 'u_projection_transition',
  'u_projection_tile_mercator_coords',
];
```

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

## Mesh 선택 요약 (terrain 활성 여부 기준)

| 항목 | Terrain OFF | Terrain ON |
|---|---|---|
| Mesh 소스 | `maplibregl.createTileMesh` (공개 API) | `terrain.getTerrainMesh(tileID)` (`@internal`, 권장) |
| Vertex 밀도 | 4~1,089 (projection별) | 16,641 고정 (129×129) |
| Projection matrix | `map.transform.getProjectionData(...)` | 동일 |
| 셰이더 prelude | `args.shaderData.vertexShaderPrelude` | 동일 |
| Pole 처리 | `extendToNorthPole`/`extendToSouthPole` 플래그 | terrain 내부 자동 |
| Vertex format | Int16 × 2 (`a_pos` 2컴포넌트, stride 0 or 4) | Int16 × 3 (`Pos3dArray`, stride 8) |
| Elevation 샘플링 | 불필요 | `terrain.getTerrainData` + `get_elevation` 셰이더 함수 복제 |

### 왜 terrain 활성 시 `createTileMesh`만 쓰면 안 되나

`subdivisionGranularity.tile`은 projection 곡률 재현용 granularity이지 elevation 재현용이 아니다. Mercator에서는 `noSubdivision` → 타일당 2×2=4 vertex만 생성되므로, 4개 vertex의 elevation만 샘플링되어 타일 내부 elevation 변화가 선형으로 납작해진다. Globe에서도 z≥3이면 32×32로 고정되어 DEM 해상도를 따라가지 못한다.

Terrain이 활성일 때는 MapLibre 내부에서 `meshSize=128` 고정 mesh를 쓰고, 이를 `terrain.getTerrainMesh()`로 재사용하는 것이 가장 자연스럽다. 공개 API에서 동일 밀도를 원하면 `createTileMesh({granularity: 128})`로 명시 호출.

### 기존 문서 대비 변경점

이전 문서 버전은 "모든 경우에 `createTileMesh`"를 권장했으나, 이는 terrain 활성 시 vertex 부족 문제를 간과한 것이다. 현재 버전은 terrain 여부에 따른 분기를 권장한다.

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
