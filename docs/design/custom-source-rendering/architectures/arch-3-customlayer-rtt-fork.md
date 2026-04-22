# Architecture 3 — CustomLayer RTT Fork

MapLibre 코어를 수정하여 `CustomLayer`가 RTT 스택에 참여하도록 확장한다. 사용자에게는 **완전한 GLSL 자유도 + RTT 자동 합성**을 제공한다. 코어 fork 유지가 전제이며, 군사 프로젝트처럼 이미 fork를 운영 중인 환경에서 현실적 선택.

## 적합 시나리오

- 임의 GLSL 셰이더가 필수이면서 RTT 참여도 필수 (symbols 위에 덧그려지지 않아야 함)
- MapLibre core fork 유지 비용 수용 가능
- 장기적으로 upstream PR 기여를 고려
- Arch 1의 텍스처 해상도 제약을 받아들일 수 없고, Arch 4의 z-order 제약도 받아들일 수 없음

## 최소 변경 사항 (~80 LoC)

### (1) `src/webgl/render_to_texture.ts:16-23` — 화이트리스트 추가

```ts
const LAYERS_TO_TEXTURES: { [keyof in StyleLayer['type']]?: boolean } = {
  background: true,
  fill: true,
  line: true,
  raster: true,
  hillshade: true,
  'color-relief': true,
  custom: true,              // 추가
};
```

### (2) `src/webgl/draw/draw_custom.ts` — coords 파라미터 + per-tile 호출 분기

```ts
export function drawCustom(
  painter: Painter,
  tileManager: TileManager,
  layer: CustomStyleLayer,
  coords: OverscaledTileID[],    // 추가
  renderOptions: RenderOptions
) {
  const impl = layer.implementation;
  const {isRenderingGlobe} = renderOptions;
  const context = painter.context;
  const transform = painter.transform;
  const projection = painter.style.projection;

  // RTT 모드에서는 per-tile 렌더 훅 사용
  if (renderOptions.isRenderingToTexture && impl.renderTile) {
    painter.setCustomLayerDefaults();
    context.setColorMode(painter.colorModeForRenderPass());

    for (const tileID of coords) {
      const projectionData = transform.getProjectionData({
        overscaledTileID: tileID,
        applyTerrainMatrix: false,
        applyGlobeMatrix: !renderOptions.isRenderingToTexture,
      });

      const customLayerArgs: CustomRenderMethodInput = {
        farZ: transform.farZ,
        nearZ: transform.nearZ,
        fov: transform.fov * Math.PI / 180,
        modelViewProjectionMatrix: transform.modelViewProjectionMatrix,
        projectionMatrix: transform.projectionMatrix,
        shaderData: {
          variantName: projection.shaderVariantName,
          vertexShaderPrelude: `const float PI = 3.141592653589793;\nuniform mat4 u_projection_matrix;\n${projection.shaderPreludeCode.vertexSource}`,
          define: projection.shaderDefine,
        },
        defaultProjectionData: projectionData,
        tileID,                  // RTT 모드 추가 정보
      };

      impl.renderTile(context.gl, customLayerArgs);
    }

    context.setDirty();
    painter.setBaseState();
    return;
  }

  // 기존 비-RTT 경로 유지 (원본과 동일)
  // ...
}
```

### (3) `src/style/style_layer/custom_style_layer.ts` — 인터페이스 확장

```ts
export interface CustomLayerInterface {
  id: string;
  type: 'custom';
  renderingMode?: '2d' | '3d';
  renderToTexture?: boolean;        // 추가: opt-in 플래그
  render: CustomRenderMethod;
  prerender?: CustomRenderMethod;
  renderTile?: CustomRenderMethod;  // 추가: RTT 모드 per-tile 훅
  onAdd?(map: Map, gl: WebGLRenderingContext | WebGL2RenderingContext): void;
  onRemove?(map: Map, gl: WebGLRenderingContext | WebGL2RenderingContext): void;
}

export interface CustomRenderMethodInput {
  // ... 기존 필드
  tileID?: OverscaledTileID;         // 추가: RTT 모드에서만 제공
}
```

### (4) `src/render/painter.ts:658-690` — renderLayer dispatch에 coords 전달

```ts
case 'custom':
  this.drawFunctions.custom(this, tileManager, layer as CustomStyleLayer, coords, renderOptions);
  break;
```

## 사용자 API

```ts
const layer: CustomLayerInterface = {
  id: 'my-custom-rtt',
  type: 'custom',
  renderToTexture: true,
  renderingMode: '3d',

  onAdd(map, gl) {
    this.program = compileShaders(gl);
  },

  renderTile(gl, args) {
    // args.tileID — 현재 RTT 타일
    // args.defaultProjectionData — tile-local projection matrix
    // 바인딩된 FBO는 painter가 이미 설정
    gl.useProgram(this.program);
    gl.uniformMatrix4fv(this.uProjMatrix, false, args.defaultProjectionData.mainMatrix);
    this.drawDomainDataForTile(gl, args.tileID);
  },

  render(gl, args) {
    // 비-RTT 모드 또는 map.terrain === null일 때의 폴백 경로
    // renderToTexture: true인 경우 RTT 가능하면 renderTile이 대신 호출됨
  }
};

map.addLayer(layer);
map.setTerrain({source: 'dem'});   // RTT 경로로 자동 전환
```

## 핵심 이점

1. **완전한 GLSL 자유도 + RTT 자동 참여** — 두 축 모두 획득
2. **공식 API 형태** — `@internal` 의존 없음
3. **Per-tile matrix 자동 제공** — `defaultProjectionData`가 tile-local 변환 포함
4. **Globe 자동 대응** — `applyGlobeMatrix` 플래그 핸들링 painter가 수행
5. **폴백 호환** — `renderToTexture: false` 또는 terrain 비활성 시 기존 `render()` 호출

## Upstream 기여 관점

PR 제출 시 고려 사항:

1. **기존 API 불변** — `renderToTexture`는 optional, 기본값 `false`, 기존 CustomLayer 구현 영향 없음
2. **테스트 추가** — RTT + custom 조합 (mercator, globe, terrain 각각)
3. **문서 업데이트** — `CustomLayerInterface` JSDoc에 새 필드 설명
4. **예제 추가** — `test/examples/` 폴더에 `add-a-custom-rtt-layer.html`
5. **breaking change 아님** — 순수 확장

관련 이슈 검색 키워드: "custom layer RTT", "CustomLayer terrain drape", `renderToTexture flag`.

## Fork 유지 전략

### Vendoring 레이어

- 변경된 3개 파일(`render_to_texture.ts`, `draw_custom.ts`, `custom_style_layer.ts`)을 별도 디렉터리에 복제
- MapLibre 업그레이드 시 차이 merge만 수행
- CI에서 diff 자동 검증 (unexpected 코어 변경 탐지)

### 회귀 테스트

- 기존 CustomLayer 예제가 그대로 동작하는지 확인 (`renderToTexture: false`)
- 새 `renderTile` 경로의 mercator/globe/terrain 조합 screenshot 비교
- RTT stack에 custom layer 혼합 시 z-order 확인

### PR 분할 전략

1. PR #1: `CustomLayerInterface`에 optional 필드 추가만 (no behavior change)
2. PR #2: `draw_custom.ts` 리팩터링 (기존 동작 유지 + 구조 개선)
3. PR #3: `LAYERS_TO_TEXTURES` + renderTile 실제 구현

작은 단위 분할이 리뷰·머지 가능성 높임.

## 제약

1. **코어 fork 유지 필수** — upstream merge 시 conflict 해결 비용
2. **버전 호환성 검증** — MapLibre 메이저 업그레이드마다 재검증
3. **PR 수용 불확실성** — upstream이 거부할 수 있음
4. **Feature query 미지원** — Arch 2와 달리 queryRenderedFeatures 자동 동작하지 않음 (수동 구현 필요)
5. **다른 RTT 레이어와 blend 조율** — 사용자 셰이더가 painter의 global blend state와 일관되게 작동해야 함

## Testing

- 3-파일 변경의 단위 테스트 (Jest + mock WebGL)
- `renderTile`이 각 coord마다 호출되는지 검증
- 기존 `render` 경로 회귀 테스트

## 검증 체크리스트

- [ ] 기존 CustomLayer 예제가 변경 후에도 동일 동작
- [ ] `renderToTexture: true` + terrain 활성화 시 drape 정상
- [ ] globe 모드에서 tile-local 매트릭스 정상
- [ ] symbol 레이어가 custom 레이어 위에 올바르게 렌더 (z-order)
- [ ] 다른 RTT 레이어(hillshade 등)와 동일 스택에 묶일 때 합성 정상

## 참고 파일

- `src/webgl/render_to_texture.ts:16-23, 140-205` — RTT 화이트리스트 + 루프
- `src/webgl/draw/draw_custom.ts:1-64` — 기존 구현 (수정 대상)
- `src/style/style_layer/custom_style_layer.ts:200-269` — `CustomLayerInterface` (수정 대상)
- `src/render/painter.ts:658-690` — renderLayer dispatch (수정 대상)

---

**See also**: [README](../README.md) · [Appendix A — RTT](../appendix/a-rtt-mechanism.md) · [Arch 4](arch-4-customlayer-terrain-mesh.md)
