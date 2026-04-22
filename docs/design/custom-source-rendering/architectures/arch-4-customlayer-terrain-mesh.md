# Architecture 4 — CustomLayer + Terrain Mesh 재활용

`CustomLayer`의 `render()` 안에서 `map.terrain.getTerrainMesh(tileID)` + `getTerrainData(tileID)` + `map.coveringTiles()`를 직접 호출하여, 기존 terrain mesh 위에 사용자 셰이더로 한 번 더 렌더링. RTT를 우회하면서 per-pixel elevation 정확도를 얻는다.

## 적합 시나리오

- CustomLayer로 이미 구현된 렌더 로직을 terrain drape까지 확장
- 임의 GLSL 셰이더 필수 + per-pixel elevation 정확도 필요
- Arch 3의 코어 fork 비용 회피
- Z-order 제약을 수용 가능 (symbols 등이 위에 겹쳐짐)

## 공개 API 검증

| API | 위치 | 용도 |
|---|---|---|
| `map.coveringTiles(options)` | `src/ui/map.ts:980` | 현재 가시 타일 ID 배열 |
| `map.terrain` | `src/ui/camera.ts:259` | Terrain 인스턴스 (미활성 시 null) |
| `terrain.getTerrainMesh(tileID)` | `src/render/terrain.ts:433` | 128×128 정규 그리드 mesh, 공유 캐싱 |
| `terrain.getTerrainData(tileID)` | `src/render/terrain.ts:264` | DEM 텍스처 + 행렬 + uniforms |
| `terrain.tileManager.getRenderableTiles()` | `src/tile/terrain_tile_manager.ts` | terrain 기준 가시 타일 |
| `map.transform.getProjectionData(opts)` | `src/geo/transform.ts` | tile-local + globe-aware projection matrix |
| Elevation 샘플링 공식 | `src/shaders/glsl/_prelude.vertex.glsl:146-166` | `get_elevation(vec2 pos)` |

## 클래스 스켈레톤

```ts
class CustomDomainLayer implements CustomLayerInterface {
  id = 'rail-custom';
  type = 'custom';
  renderingMode: '3d' = '3d';

  private map!: Map;
  private program!: WebGLProgram;
  private uniforms!: Record<string, WebGLUniformLocation>;

  onAdd(map: Map, gl: WebGL2RenderingContext) {
    this.map = map;
    this.program = compileShaders(gl, VERTEX_SHADER, FRAGMENT_SHADER);
    this.uniforms = locateUniforms(gl, this.program);
  }

  render(gl: WebGL2RenderingContext, args: CustomRenderMethodInput) {
    const terrain = this.map.terrain;
    if (!terrain) {
      this._renderFlat(gl, args);   // terrain 비활성 폴백 (아래 참조)
      return;
    }

    gl.useProgram(this.program);

    for (const tile of terrain.tileManager.getRenderableTiles()) {
      const tileID = tile.tileID;
      const mesh = terrain.getTerrainMesh(tileID);
      const td = terrain.getTerrainData(tileID);
      const proj = this.map.transform.getProjectionData({
        overscaledTileID: tileID,
        applyTerrainMatrix: false,   // 우리가 직접 get_elevation 호출
        applyGlobeMatrix: true,      // globe 자동 대응
      });

      // DEM 텍스처 바인딩
      gl.activeTexture(gl.TEXTURE0 + 2);
      gl.bindTexture(gl.TEXTURE_2D, (td as any).texture.texture);
      gl.uniform1i(this.uniforms.u_terrain, 2);
      gl.uniformMatrix4fv(this.uniforms.u_terrain_matrix, false, td.u_terrain_matrix as Float32Array);
      gl.uniform1f(this.uniforms.u_terrain_dim, td.u_terrain_dim);
      gl.uniform4fv(this.uniforms.u_terrain_unpack, td.u_terrain_unpack);
      gl.uniform1f(this.uniforms.u_terrain_exaggeration, td.u_terrain_exaggeration);

      // Projection matrix
      gl.uniformMatrix4fv(this.uniforms.u_projection_matrix, false, proj.mainMatrix);

      // 사용자 도메인 데이터 uniform/texture 바인딩
      this.bindDomainData(gl, tileID);

      // 내장 terrain mesh를 직접 그린다
      gl.bindBuffer(gl.ARRAY_BUFFER, (mesh.vertexBuffer as any).buffer);
      gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, (mesh.indexBuffer as any).buffer);
      gl.enableVertexAttribArray(0);
      gl.vertexAttribPointer(0, 3, gl.SHORT, false, 8, 0);
      gl.drawElements(gl.TRIANGLES, mesh.segments.get()[0].primitiveLength * 3,
                      gl.UNSIGNED_SHORT, 0);
    }
  }

  private _renderFlat(gl: WebGL2RenderingContext, args: CustomRenderMethodInput) {
    // terrain OFF 폴백: 자체 mercator 그리드 mesh 생성 + z=0 사용
    // 또는 이 레이어를 단순 비활성화
  }

  private bindDomainData(gl: WebGL2RenderingContext, tileID: OverscaledTileID) {
    // 사용자 도메인 데이터 텍스처/uniform 바인딩
  }
}
```

## 셰이더 스켈레톤

```glsl
// Vertex — _prelude.vertex.glsl의 projectTileFor3D 스타일
#version 300 es
${shaderData.vertexShaderPrelude}
${shaderData.define}

in vec3 a_pos3d;

uniform mat4 u_projection_matrix;
uniform mat4 u_terrain_matrix;
uniform sampler2D u_terrain;
uniform float u_terrain_dim;
uniform vec4 u_terrain_unpack;
uniform float u_terrain_exaggeration;

out vec2 v_tileUV;

// _prelude.vertex.glsl:146-166 복제
float get_elevation(vec2 pos) {
  vec2 coord = (u_terrain_matrix * vec4(pos, 0.0, 1.0)).xy * u_terrain_dim + 1.0;
  vec2 f = fract(coord);
  vec2 c = (floor(coord) + 0.5) / (u_terrain_dim + 2.0);
  float d = 1.0 / (u_terrain_dim + 2.0);
  // bilinear sample 4 corners
  float e00 = texture(u_terrain, c).r;
  float e10 = texture(u_terrain, c + vec2(d, 0)).r;
  float e01 = texture(u_terrain, c + vec2(0, d)).r;
  float e11 = texture(u_terrain, c + vec2(d, d)).r;
  // unpack + bilinear interpolation
  float e = mix(mix(e00, e10, f.x), mix(e01, e11, f.x), f.y);
  return e * u_terrain_exaggeration;   // 실제 unpack 공식은 u_terrain_unpack 사용
}

void main() {
  float ele = get_elevation(a_pos3d.xy);
  v_tileUV = a_pos3d.xy / 8192.0;   // EXTENT
  gl_Position = u_projection_matrix * vec4(a_pos3d.xy, ele, 1.0);
  // globe 모드에서는 projectTileFor3D 프렐루드 함수 사용 가능
}

// Fragment: 사용자 도메인 텍스처 샘플링 또는 절차적 렌더
```

## 핵심 이점

1. **CustomLayer의 GLSL 자유도 + terrain drape 양립** — Arch 1이 해결하지 못한 "텍스처 해상도 한계 없는 drape" 달성
2. **내부 데이터 중복 없음** — terrain이 이미 만든 mesh/DEM을 참조만, 메모리 추가 소비 거의 없음
3. **Globe 자동 대응** — `getProjectionData({applyGlobeMatrix: true})`가 `transitionState` 자동 처리
4. **RTT bucket 구축 복잡도 회피** — Arch 2의 `LineBucket` 내부 API 의존 없음
5. **coveringTiles 자연 활용** — 사용자 요구 "coveringTiles 결과 기반 렌더"에 정확히 부합
6. **per-pixel elevation 정확도** — 텍스처 해상도 제약 없음

## 제약 및 주의점

### 1. RTT 스택 미참여

`drawCustom`은 translucent pass에서 호출됨 (`src/webgl/draw/draw_custom.ts:45`). 기본 terrain 합성이 끝난 뒤 **그 위에 덧그려짐**. Z-order 결과:

- 내장 line/fill/symbol보다 뒤에 그려짐 (RTT 단계에서 먼저 합성)
- Sky/atmosphere보다는 앞
- 심볼이 커스텀 렌더 위에 얹히지 않음 → depth/stencil로 조정 필요

### 2. Terrain mesh 두 번 래스터화

기본 지도가 이미 그린 mesh에 우리가 또 그리는 overdraw. 128×128 × 타일 수 정도, 실무상 무시 가능하지만 고성능 요구 시 프로파일링 필요.

### 3. `@internal` API 의존

`getTerrainData`, `getTerrainMesh`는 JSDoc에 `@internal` 표시. 마이너 버전 업그레이드 시 시그니처 변경 가능성. 벤더링 또는 `as any` 캐스트 필요.

### 4. Terrain 미활성 시 폴백

`map.terrain === null`일 때 이 경로 자체가 동작 안 함. 평면 렌더 대체 로직이 CustomLayer 내부에 필요:

```ts
private _createFlatMesh(gl: WebGL2RenderingContext) {
  // 128x128 정규 그리드 mesh 직접 생성 (z=0)
  // ...
}
```

또는 셰이더 분기: `#ifdef NO_TERRAIN` → `ele = 0.0`.

### 5. queryRenderedFeatures 미지원

CustomLayer 기본 한계. 자체 공간 인덱스 유지 필요.

### 6. Depth ordering 수동 관리

다른 RTT 레이어와의 상호작용을 사용자가 `gl.depthMask`, stencil로 직접 조정.

## TerrainBridge 격리 패턴

MapLibre 독립성을 위해 의존성을 격리:

```ts
interface TerrainBridge {
  getCoveringTiles(): TileID[];
  getTerrainMesh(id: TileID): { vbo: WebGLBuffer; ibo: WebGLBuffer; count: number };
  getTerrainData(id: TileID): {
    dem: WebGLTexture;
    matrix: Float32Array;
    exaggeration: number;
    unpack: [number, number, number, number];
    dim: number;
  };
  getProjection(id: TileID): Float32Array;
}

class MapLibreTerrainBridge implements TerrainBridge {
  // map.terrain.* 호출을 캡슐화
}
```

MapLibre 업그레이드 시 이 브리지 구현만 검증. 미래에 다른 렌더러(Cesium, Mapbox GL fork 등)로 포팅 시에도 이 인터페이스만 재구현.

## Testing

- `map.terrain` null/non-null 양쪽 경로 테스트
- terrain mesh 접근이 `@internal` API 파손을 감지하도록 smoke test
- Screenshot 비교 (terrain on/off)
- Globe transition 상태 0.0 / 0.5 / 1.0에서 렌더 정상

## Debugging

- `terrain.getRenderableTiles()` 타일 수 로깅
- DEM 텍스처 바인딩 확인 (Spector.js frame capture)
- 셰이더 컴파일 에러: `get_elevation` 공식 재검토
- Z-fighting 방지: `depthFunc`, `polygonOffset` 조정

## Performance Profiling

- 가시 타일 × 128² = 약 640K 삼각형 추가 draw
- 2024 이후 임베디드 GPU에서 1-2ms 수준
- Chrome GPU timeline에서 `drawElements` 측정

## Memory

- mesh/DEM 공유 참조이므로 거의 0 추가
- 사용자 program/uniform은 자체 관리

## Worker 활용

- 도메인 데이터 전처리 Worker 분리
- per-tile domain texture 준비를 Worker에서 (ImageData → transferable)
- GL 업로드는 메인 스레드 `render()`에서

## 검증 체크리스트

- [ ] terrain OFF + 폴백 경로 동작
- [ ] terrain ON + 구면 표면에 정확히 drape
- [ ] globe transition 중간값에서 흔들림 없음
- [ ] Z-order: symbol 레이어가 custom 위에 정상 렌더
- [ ] unit 수 많을 때 프레임 타임 안정 (<5ms per frame)
- [ ] GPU memory 증가 없음

## 참고 파일

- `src/ui/map.ts:980` — `coveringTiles` 공개 API
- `src/ui/camera.ts:259` — `map.terrain`
- `src/render/terrain.ts:264, 433` — `getTerrainData`, `getTerrainMesh`
- `src/tile/terrain_tile_manager.ts:136` — `getRenderableTiles`
- `src/shaders/glsl/_prelude.vertex.glsl:146-166` — `get_elevation`
- `src/shaders/glsl/terrain.vertex.glsl` — 참조 셰이더
- `src/style/style_layer/custom_style_layer.ts:200-269` — `CustomLayerInterface`
- `src/webgl/draw/draw_custom.ts:1-64` — 기존 `drawCustom`

---

**See also**: [README](../README.md) · [Appendix B — Globe + Terrain](../appendix/b-globe-terrain-interaction.md) · [Use Cases — Military](../use-cases/military-rtt-layers.md) · [FAQ](../faq.md)
