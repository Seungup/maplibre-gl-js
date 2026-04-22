# FAQ

Custom Source Rendering 설계에 관한 꼬리 질문과 답변. 카테고리별 26개 항목.

## 선택·결정 (Q1-4)

### Q1. 셰이더 자유도가 필요 없고 RTT만 원하면 GeoJSON + line을 왜 안 쓰나?

기본적으로 GeoJSON + line이 최선. 본 문서는 그 경로로 불가능한 경우만 다룬다. 다음 중 하나가 해당될 때 커스텀 소스 검토:

- 비-GeoJSON 데이터 포맷 (바이너리, 스트리밍, 자체 규약)
- 런타임 procedural 생성
- 내장 paint property로 표현 못 하는 셰이더 효과
- 대용량 main-thread tessellation 회피 필요 (Worker 전처리 전제)

**관련**: [README 결정 가이드](README.md#선택-가이드)

### Q2. `addSourceType`을 쓸 가치가 있는 최소 데이터 규모는?

- **수천~수만 feature 이하**: `GeoJSONSource` + `setData` 재호출로 충분
- **수십만 이상 + 타일 단위 lazy loading**: 커스텀 소스 가치 발생
- **동적 생성/API 스트리밍**: 규모 무관하게 커스텀 소스 권장

### Q3. Arch 1과 4를 동시에 써도 되나?

가능. 서로 다른 레이어로 존재. 예: Arch 1으로 fog of war 드레이프 + Arch 4로 sweep line 3D.

- **주의**: 두 경로 모두 `terrain.getTerrainData` 호출 시 프레임당 GL 비용 증가
- **권장**: 공유 리소스(TerrainBridge) 한 곳에서 관리

**관련**: [Arch 4 TerrainBridge](architectures/arch-4-customlayer-terrain-mesh.md#terrainbridge-격리-패턴)

### Q4. MapLibre 업그레이드 주기와 각 Arch의 파손 위험은?

| Arch | 의존 API 안정성 | 업그레이드 영향 |
|---|---|---|
| 1 | `Texture`, `Context` — 안정 | 마이너 업그레이드 대체로 안전 |
| 2 | `LineBucket` populate — `@internal` | 중간 위험, 업그레이드마다 검증 필요 |
| 3 | 자체 fork | upstream 변경 흡수 비용 존재 |
| 4 | `createTileMesh` + `subdivisionGranularity` + `vertexShaderPrelude` — **공개 안정** | 낮은 위험. terrain elevation 확장 시에만 `getTerrainData` `@internal` 의존 |

## 데이터 흐름 (Q5-8)

### Q5. loadTile/prepare 분리 시 첫 프레임에 빈 타일이 보이지 않나?

안 보임. render 순서 `style.prepare → painter.render`이므로 loadTile 완료 후 같은 프레임 prepare에서 렌더됨. 단 loadTile이 비동기 await 중이면 다음 프레임까지 지연 (기존 네트워크 tile과 동일한 1프레임 지연).

**관련**: [Arch 1 타일 라이프사이클](architectures/arch-1-custom-raster-source.md#타일-라이프사이클)

### Q6. prepare() 내부에서 async/await가 가능한가?

기술적으로 가능하지만 권장 안 함. `prepare`는 동기 GL 호출 컨텍스트에 최적화됨.

- 데이터 로딩은 `loadTile`에서 async 처리
- `prepare`에서는 준비된 데이터로 동기 렌더링만
- 비동기 GPU 작업 필요 시 `loadTile`에서 해결하고 `prepare`는 결과 binding만

### Q7. hasTransition()을 항상 true로 반환하면 배터리/CPU 영향은?

- 맵이 항상 re-render 루프 → `idle` 이벤트 미발생 → 전력 소모 지속
- 모바일/임베디드에서 수 watt 단위 추가 소모 가능
- **반드시 조건부 반환**: `_pendingRender.size > 0 || hasStaleZoom`

### Q8. setData() 호출 시 기존 타일을 개별 invalidate 할 수 있나?

- **기본 방식**: `fire('data', {sourceDataType: 'content'})` — 모든 타일 재로드
- **세밀 제어**: source 내부에서 영향받는 타일 key만 `_pendingRender`에 추가 → 다음 prepare에서 재렌더
- **SourceCache의 shouldReloadTile hook** 활용 가능 (GeoJSONSource 참조)

## 성능·메모리 (Q9-12)

### Q9. FBO 텍스처를 painter.saveTileTexture 풀에 넣어도 안전한가?

- **동일 tileSize + 동일 filter + 동일 format** 타일만 재사용 가능
- FBO attachment 이력이 있는 텍스처는 pool 재사용 시 의도치 않은 부작용 가능
- **권장**: Arch 1에서는 자체 pool 유지, painter pool 미사용

### Q10. 모든 가시 타일 재래스터화가 프레임 드롭을 일으킬 때 throttle 방법은?

- 프레임당 최대 N개 타일 처리 후 나머지는 다음 프레임으로 이월
- `_pendingRender`를 우선순위 큐로 변경 (화면 중앙 타일 우선)
- `_zoomEpsilon` 증가로 재래스터화 임계값 완화
- 극단적 경우 `requestIdleCallback` 활용 (단 타이밍 비결정적)

### Q11. 가시 타일이 100+일 때 어떻게 감당하나?

globe에서 흔한 상황 (구의 far-side까지 타일 요청 가능).

- `hasTile()` 필터로 데이터와 겹치지 않는 타일 제외
- Arch 2: Worker에서 tessellation 분산 준비 후 메인 스레드 bucket 조립
- Arch 1: 타일 우선순위 + frame budget cap
- Arch 4: mesh 재사용으로 draw 비용이 상대적으로 낮음

### Q12. Worker 데이터 전처리 패턴은?

- **Worker**: 사용자 데이터 파싱, 공간 인덱스 구축, feature 필터링, per-tile feature 리스트 생성
- **Main**: `loadTile`에서 Worker에 `postMessage({tileID})`, 결과를 받아 Texture/Bucket 생성
- **Transferable 활용**: `ArrayBuffer`로 정점 데이터 전달하여 copy 비용 제거
- Arch 2에서 `LineBucket` 자체는 main에서 생성하지만 feature 배열은 Worker 준비

## 통합·호환 (Q13-16)

### Q13. queryRenderedFeatures는 어떻게 동작하나 / 자체 구현?

- **Arch 2**: 네이티브 지원 (`FeatureIndex` 구성 시)
- **Arch 1/4**: 미지원. 자체 구현 시 소스가 공간 인덱스 유지 + Map click 이벤트에서 lng/lat을 소스에 질의
- **FeatureIndex 구성 방법**: `src/data/feature_index.ts` 참조, LineBucket 생성 시 함께 populate

### Q14. 다른 RTT 레이어와 같은 타일 좌표계에서 합성되나?

- **Arch 1/2**: 동일 RTT 스택에 속하면 자동 합성 (스타일 순서 기준)
- **Arch 3**: 스택 참여 (fork 수정 범위)
- **Arch 4**: 불참 — translucent pass에서 위에 덧그려짐

**관련**: [Appendix A — 레이어 순서와 RTT 스택](appendix/a-rtt-mechanism.md#레이어-순서와-rtt-스택)

### Q15. Globe transition 중간에서는 어떻게 렌더되나?

- `transitionState` 0~1 사이 값 → mercator/vertical_perspective 행렬 보간
- 커스텀 소스는 특별 처리 불필요 (`getProjectionData` 사용 시 자동)
- 애니메이션 중 타일 재로드 없음 (mercator tile 좌표계 유지)
- 시각적 아티팩트 주의: 극지방 근처 타일에서 일시적 왜곡 가능

**관련**: [Appendix B — Globe + Terrain](appendix/b-globe-terrain-interaction.md)

### Q16. High-DPI에서 tileSize 규약은?

- `tileSize` 속성은 **논리 픽셀**
- 실제 FBO 해상도는 `tileSize * map.getPixelRatio()` 권장
- retina(pixelRatio=2)에서 512 논리 → 1024 물리 픽셀 FBO
- 셰이더에서 `u_device_pixel_ratio` uniform 활용

## 디버깅·운영 (Q17-20)

### Q17. "내 타일이 안 그려진다"를 어디서부터 디버깅하나?

순서:
1. `loadTile` 호출 확인 (console.log)
2. `tile.state === 'loaded'` 확인
3. `tile.texture` 또는 `tile.buckets` 존재 확인
4. painter 호출 경로 확인
5. FBO 내용 `readPixels`로 검증
6. 셰이더 컴파일 에러 확인

일반적 원인:
- `hasTile()` false 반환
- `tile.state` 잘못 설정
- GL 상태 오염

### Q18. loadTile이 예외를 던지면 painter가 중단되나?

중단되지 않음. `TileManager`가 예외를 catch하여 `tile.state = 'errored'`로 설정. 해당 타일만 렌더 제외, 맵 전체는 정상.

단 console에 에러 출력되므로 조용한 실패 방지 권장 (내부 try/catch + `fire('error')`).

### Q19. WebGL Context loss 복구?

- `webglcontextlost` / `webglcontextrestored` 이벤트 구독
- Lost 시: 내부 GL 리소스(program, FBO, texture) 모두 무효화
- Restored 시: `onRemove` → `onAdd` 재실행 패턴, 모든 리소스 재생성
- MapLibre가 자체 복구 수행하므로 커스텀 소스/레이어만 자체 복구 책임

### Q20. Style hot-reload 시 커스텀 소스 상태는?

- `map.setStyle(newStyle)` 시 기존 sources 제거 → 새 sources 생성
- 커스텀 소스의 사용자 데이터가 메모리에 있다면 새 소스 인스턴스로 명시적 전달 필요
- 전략: `map.setStyle(style, {diff: true})`로 변경분만 적용
- Custom source 재초기화 비용이 크면 style 교체 대신 `addLayer`/`removeLayer` 사용

## 군사 특화 (Q21-23)

### Q21. 레이더 sweep 애니메이션을 Arch 1 재베이크로 60fps 달성 가능한가?

- 가시 타일 20개 기준, sweep 한 라인 + 페이드 그라데이션 셰이더는 타일당 0.1ms 이하
- 전체 프레임 비용: 20 × 0.1 = 2ms 수준, 60fps 예산 내
- 타일당 셰이더 복잡도가 높으면 Arch 3 fork 고려
- Frame budget 모니터링 및 graceful degradation 필수

**관련**: [Military — 레이더 Sweep](use-cases/military-rtt-layers.md#레이더-sweep-애니메이션--두-가지-옵션)

### Q22. Fog of war 여러 소스 합성 권장 패턴은?

- **단일 커스텀 소스 multi-pass**: pass1 exploration mask, pass2 blur, final output → `tile.texture`
- **분리된 소스 + 분리된 raster 레이어 2개**: `raster-opacity`로 블렌딩 — flexibility 높지만 `tile.texture` 메모리 2배

### Q23. MIL-STD-2525 심볼과 Arch 4 z-order 조율?

- Arch 4는 translucent 패스 중 스타일 순서대로 그려짐
- 심볼을 MapLibre 외부(HUD)에서 그린다면 전체 canvas 렌더 순서에서 조율
- MapLibre `symbol`로 그린다면 style 순서 조정: `[base, custom Arch 4, symbol]`

**관련**: [Military — z-order 조율](use-cases/military-rtt-layers.md#mil-std-2525--app-6-심볼과의-z-order-조율)

## 진화·마이그레이션 (Q24-26)

### Q24. 기존 CustomLayer 구현을 Arch 1/4로 포팅 체크리스트?

**Arch 4 (가장 유사)**:
- `render` 로직 거의 그대로 유지
- **`maplibregl.createTileMesh()` 기반 per-tile 루프로 감싸기** (공식 패턴, 공개 API)
- 기존 `terrain.getTerrainMesh` 사용하던 코드는 `createTileMesh`로 교체 권장 — 공개 API 사용으로 업그레이드 안정성 향상
- 셰이더의 `projectTile`/`projectTileFor3D`는 `args.shaderData.vertexShaderPrelude`로 자동 제공

**Arch 1**:
- `render` 로직을 `prepare`로 이전
- `tile.texture` 결과물로 방향 전환
- `raster` 레이어 추가

**공통**:
- 전역 좌표 → tile-local 좌표 변환 로직 추가
- 회귀 테스트: screenshot 비교, 프레임 시간 프로파일 비교

### Q25. Arch 3 fork를 upstream 기여 시 고려 사항?

- 관련 issue/PR 검색 (maplibre-gl-js 저장소)
- 제안 범위 최소화: `CustomLayerInterface`에 optional 필드만 추가, 기존 render 동작 불변
- 테스트 추가: RTT + custom 조합 시나리오 (mercator, globe, terrain 조합)
- 문서화: `CustomLayerInterface` JSDoc 업데이트
- breaking change 아닌 확장 형태로 제안
- PR을 3개 작은 단위로 분할 (인터페이스 → 리팩터링 → 구현)

**관련**: [Arch 3 — Upstream 기여 관점](architectures/arch-3-customlayer-rtt-fork.md#upstream-기여-관점)

### Q26. WebGPU 지원 시 각 Architecture 영향?

MapLibre가 WebGPU 백엔드를 도입하면 `Texture`/`Context` 추상화가 바뀜:

- **Arch 1**: `Texture` 래퍼 API만 업데이트되면 거의 무영향
- **Arch 2**: `LineBucket` 내부가 WebGPU로 재작성 시 wrapper 필요
- **Arch 3**: fork 전체 마이그레이션 필요 (painter 파이프라인 변경)
- **Arch 4**: `getTerrainMesh` 반환 타입 변경 가능, 셰이더 WGSL 재작성

2026년 기준 WebGPU는 MapLibre roadmap에서 실험 단계.

### Q27. Arch 4에서 `createTileMesh`와 `terrain.getTerrainMesh` 차이?

**핵심 차이는 vertex 밀도**. elevation 샘플링은 per-vertex interpolation이므로 terrain 활성 시 밀도가 중요하다.

| 항목 | `maplibregl.createTileMesh` | `terrain.getTerrainMesh` |
|---|---|---|
| 공개 여부 | 공개 (`src/index.ts:382` export) | `@internal` |
| Mercator에서 vertex 수 | **2×2 = 4** (`noSubdivision`) | 129×129 = 16,641 |
| Globe z=0 vertex 수 | 129×129 = 16,641 | 129×129 = 16,641 |
| Globe z≥3 vertex 수 | **33×33 = 1,089** (min 32 clamp) | 129×129 = 16,641 |
| Granularity 제어 | 호출자 (`options.granularity`) | 고정 `meshSize = 128` |
| Vertex format | `Int16 × 2` (a_pos x,y) | `Pos3dArray` (x, y, frame-bit) stride=8 |
| Output | `{vertices, indices, uses32bitIndices}` typed array | VBO/IBO 포함 `Mesh` 객체 (즉시 바인딩 가능) |
| Pole 처리 | `extendToNorthPole`/`extendToSouthPole` 플래그 | globe에서 자동 판정 |
| 캐싱 | 호출자 책임 | 내장 `_meshCache` (tile 간 공유) |

**권장**:

- **Terrain OFF**: `createTileMesh` + `subdivisionGranularity.tile.getGranularityForZoomLevel(z)` 사용. 공개 안정 API + projection 곡률 재현에 충분.
- **Terrain ON**: `terrain.getTerrainMesh(tileID)` 사용. `createTileMesh`의 기본 granularity는 elevation 재현에 부족 (mercator는 vertex 4개, globe 고줌은 1,089개). `createTileMesh`로 동일 밀도를 확보하려면 `{granularity: 128}` 명시 호출.

**왜 `createTileMesh`만으로는 terrain에 불충분한가**: `subdivisionGranularity.tile`은 **projection 곡률**(globe의 구면) 재현용이지 **elevation 샘플링**용이 아니다. Mercator는 `noSubdivision`이라 vertex 4개만 생성 → 타일 내부 elevation이 선형 평탄화됨. Globe도 z≥3에서 32로 고정되어 DEM 해상도를 못 따라감.

**`getTerrainMesh` 재활용의 숨은 비용**: terrain mesh는 단순 정규 격자가 아니라 **main grid + top/bottom frame + left/right frame** 3 섹션이 하나의 VBO에 합쳐진 특수 구조. vertex 포맷은 `Pos3dArray`(Int16 × 3)이고 3번째 컴포넌트는 **frame-bit** (z=1이면 frame vertex). 터레인 셰이더는 `a_pos3d.z == 1.0 ? u_ele_delta : 0.0`로 frame vertex를 `terrain.getMeshFrameDelta(zoom)` 만큼 아래로 내려 cross-zoom 타일 경계 seam을 숨긴다. Custom layer에서 이 mesh를 재활용하며 elevation까지 쓴다면, 셰이더에 동일한 frame-bit 처리를 **반드시 복제**해야 한다. 단순히 `in vec2 a_pos`로 xy만 읽으면 zoom 전환 시 seam이 보인다 (다만 elevation을 안 쓰면 괜찮음 — frame vertex가 메인 vertex와 같은 (x,y)라서 화면에서 degenerate).

**관련**: [Arch 4 Mesh 선택 가이드](architectures/arch-4-customlayer-terrain-mesh.md#mesh-선택-가이드--중요--vertex-밀도-차이)

---

**See also**: [README](README.md) · [Glossary](glossary.md)
