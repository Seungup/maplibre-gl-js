# Custom Source Rendering — Design Documentation

MapLibre GL JS에서 CustomLayer가 RTT(Render-to-Texture)에 참여하지 못하는 제약을 우회하여, 커스텀 벡터/래스터 데이터를 terrain drape와 함께 렌더링하는 네 가지 아키텍처를 정리한 설계 문서 세트.

## 문제

`CustomLayer`는 translucent 패스에서 framebuffer에 직접 그리므로 `RenderToTexture` 파이프라인에 포함되지 않는다. 결과적으로 terrain drape, hillshade 합성, 기타 RTT 의존 효과가 적용되지 않는다. `LAYERS_TO_TEXTURES` 화이트리스트(`src/webgl/render_to_texture.ts:16-23`)에 `custom`이 없음.

사용자 목표:
- 커스텀 데이터(포맷·생성 로직이 내장 소스와 다름)를 타일 기반으로 렌더링
- terrain 및 globe 모드에서 정확하게 합성
- 가능하면 커스텀 GLSL 셰이더 자유도 유지
- MapLibre 내부 API 의존을 최소화

## 네 가지 아키텍처

| | [Arch 1](architectures/arch-1-custom-raster-source.md) | [Arch 2](architectures/arch-2-custom-vector-bucket-source.md) | [Arch 3](architectures/arch-3-customlayer-rtt-fork.md) | [Arch 4](architectures/arch-4-customlayer-terrain-mesh.md) |
|---|---|---|---|---|
| **접근** | Custom Raster Source + `tile.texture` | Custom Vector Bucket Source + `tile.buckets` | CustomLayer RTT 확장 (코어 fork) | CustomLayer + terrain 여부에 따른 mesh 선택 |
| **레이어 타입** | `raster` | `line` / `fill` / `circle` | `custom` + `renderToTexture` | `custom` |
| **셰이더 자유도** | 완전 (prepare 내부) | 내장 paint property 한정 | 완전 | 완전 |
| **1px 정밀도** | 텍스처 해상도 제한 | GPU `u_ratio`로 자동 | 사용자 구현 | per-pixel DEM 샘플링 |
| **RTT 참여** | 예 | 예 | 예 (fork 후) | 아니오 |
| **Terrain drape** | 자동 | 자동 | 자동 (fork 후) | 수동 (`get_elevation` 호출) |
| **Globe 자동 대응** | 예 | 예 | 사용자 구현 | 예 (`vertexShaderPrelude`) |
| **코어 수정** | 불필요 | 불필요 | 필요 (~80 LoC) | 불필요 |
| **API 안정성** | 상 | 중 (`LineBucket` @internal) | 해당 없음 | terrain OFF: 상 (`createTileMesh` 공개) / terrain ON: 중 (`getTerrainMesh` @internal, elevation 밀도 필수) |
| **Feature query 네이티브** | 미지원 | 지원 | 미지원 | 미지원 |

## 선택 가이드

```
요구사항
│
├─ 내장 paint property (line-width/color/pattern/gradient/dasharray) 조합으로 표현 가능?
│   └─ Yes → Arch 2 (네이티브 품질 + queryRenderedFeatures)
│
├─ 임의 GLSL 셰이더가 필수?
│   ├─ RTT 참여 + 코어 수정 가능 → Arch 3 (가장 깔끔)
│   ├─ RTT 참여 필수 + 코어 수정 불가 → Arch 1 (texture 해상도 타협)
│   └─ RTT 불필요 (translucent pass OK) → Arch 4 (가장 직접적)
│
└─ 하이브리드 (정적 오버레이 + 라이브 효과) → Arch 1 + Arch 4 조합
```

## 유스케이스별 빠른 매칭

- **표준 벡터 도메인 (철도/도로/경계)** → [Arch 2](architectures/arch-2-custom-vector-bucket-source.md)
- **Fog of war / 커버리지 / 위협 엔벨로프 (드레이프)** → [Arch 1](architectures/arch-1-custom-raster-source.md)
- **레이더 sweep / 라이브 애니메이션 드레이프** → Arch 1 + hasTransition 또는 Arch 3
- **3D 유닛 마커 / 궤적 (공중)** → [Arch 4](architectures/arch-4-customlayer-terrain-mesh.md)
- **군사 전술 디스플레이 종합** → [군사 유스케이스 가이드](use-cases/military-rtt-layers.md)

## 문서 구조

### Architectures
- [Arch 1 — Custom Raster Source](architectures/arch-1-custom-raster-source.md)
- [Arch 2 — Custom Vector Bucket Source](architectures/arch-2-custom-vector-bucket-source.md)
- [Arch 3 — CustomLayer RTT Fork](architectures/arch-3-customlayer-rtt-fork.md)
- [Arch 4 — CustomLayer + Terrain Mesh](architectures/arch-4-customlayer-terrain-mesh.md)

### Appendix (배경 지식)
- [A. RTT Mechanism](appendix/a-rtt-mechanism.md) — per-tile 렌더 루프, 데이터 규약
- [B. Globe + Terrain Interaction](appendix/b-globe-terrain-interaction.md) — Layered rendering
- [C. Vector Pipeline Reference](appendix/c-vector-pipeline-reference.md) — 내장 벡터 파이프라인
- [D. Overscale Strategies](appendix/d-overscale-strategies.md) — 연속 zoom 정밀도 전략 A-D

### Use Cases
- [Military RTT Layers](use-cases/military-rtt-layers.md)

### Reference
- [FAQ (26 questions)](faq.md)
- [Glossary](glossary.md)

## 핵심 발견 요약

- **RTT 렌더 분기는 레이어 타입 기준**이지 소스 타입 기준이 아님 (`src/render/painter.ts:658-690`). 커스텀 소스 + `raster` 레이어 조합 가능.
- **`drawRaster`는 `tile.texture`만 요구** — 소스 타입 검증 없음 (`src/webgl/draw/draw_raster.ts`).
- **Source-Layer 타입 호환성 검증 없음** — `style.ts`가 sourceLayer 유효성만 검증.
- **Globe + Terrain은 공존 가능** — RTT pass에서 mercator로 per-tile FBO 생성, `drawTerrain`이 `applyGlobeMatrix: true`로 terrain mesh를 globe 투영 (`src/webgl/draw/draw_terrain.ts:93`).
- **CustomLayer는 RTT 스택에 참여 불가** — translucent 패스의 전역 render 호출 1회, tile-local 컨셉 없음.
- **커스텀 소스의 `prepare()`는 벡터 소스의 RTT per-tile 렌더와 기능적으로 동등** — 텍스처 저장소를 통해 간접 경로로 RTT 합성에 합류.
- **CustomLayer에서 tile-based 렌더링은 공식 API로 지원됨** — `maplibregl.createTileMesh()` (`src/index.ts:382` export) + `map.style.projection.subdivisionGranularity.tile.getGranularityForZoomLevel(z)` + `args.shaderData.vertexShaderPrelude`. 공식 예제: `test/examples/add-a-custom-layer-with-tiles-to-a-globe.html`.

## 로드맵

- 본 문서에서 제안된 패턴이 실제 프로젝트에서 검증되면 MapLibre core에 다음 중 하나를 제안 가능:
  - `CustomRasterSourceBase` / `CustomVectorBucketSourceBase` 추상 베이스 클래스
  - `CustomLayerInterface.renderToTexture` 옵션 (Arch 3의 공식 수용)
  - `@internal` terrain/bucket API의 공식 API 승격

---

**See also**: [Glossary](glossary.md) · [FAQ](faq.md)
