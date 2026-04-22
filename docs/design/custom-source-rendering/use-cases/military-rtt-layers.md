# Use Case — Military RTT Layers

군사용 전술 디스플레이에서 MapLibre 기반 커스텀 렌더링을 구축할 때의 가이드. 일반 문서의 관점(본 세트 README 참조)에 **군사 도메인 특화 요구사항**(MapLibre 독립성, 결정론적 프레임 타임, 극한 최적화)을 반영한 권장 조합.

## 도메인 특성 요구사항

1. **MapLibre 독립성** — 코어 fork 또는 벤더링 레이어 유지가 표준. 공급 체인 관점에서 업스트림 의존성 최소화.
2. **극한 최적화** — 임베디드 GPU, 통합 메모리, 배터리 제약. 불필요한 오버드로/메모리 할당 금지.
3. **결정론적 프레임 타임** — 전술 디스플레이는 프레임 편차보다 일관성이 더 중요한 경우 많음. 60fps 일관성이 70fps 평균보다 우선.
4. **확정적 레이어 순서** — HUD 스택(지도 → 그리드 → 아/적 → 위협 → 심볼 → 컨트롤)이 MapLibre style 내부 순서보다 우선.
5. **MIL-STD-2525 / APP-6** 심볼은 대부분 MapLibre 외부(Canvas 2D, 자체 심볼 엔진)에서 처리.

## 유스케이스 분류

### 드레이프 (지형 표면에 칠해지는) 효과

| 유스케이스 | 특성 | 권장 Arch |
|---|---|---|
| Fog of war (미탐색 영역 마스크) | 저빈도 갱신, 그라데이션 | Arch 1 |
| 레이더/센서 coverage zone | 그라데이션, 정적 또는 저빈도 | Arch 1 |
| SAM/위협 엔벨로프 (지면 footprint) | 복잡한 모양, 저빈도 | Arch 1 |
| 포병 사거리/위험 영역 | 링/채움, 저빈도 | Arch 1 |
| Line-of-sight / 가시영역 분석 | per-fragment 계산, 저빈도 | Arch 1 |
| 지형 슬로프/상세 음영 | 힐셰이드 응용 | Arch 1 |
| 지뢰지대/출입 제한 | 드레이프 영역 | Arch 1 |
| 이동 회랑 (선 기반) | 샤프 1px 라인 필요 | Arch 2 또는 Arch 1 + 큰 tileSize |
| 사거리 링 (선 강조) | 샤프 라인 | Arch 2 또는 Arch 3 |
| 레이더 Sweep 애니메이션 | 매 프레임 갱신 드레이프 | Arch 1 + `hasTransition` 또는 Arch 3 |

### 공중/3D (비드레이프) 효과

| 유스케이스 | 권장 Arch |
|---|---|
| 3D 유닛 마커 (지형 위 공중) | Arch 4 |
| 궤적/비행경로 | Arch 4 |
| 드론 스트리밍 뷰 포인터 | Arch 4 |
| 위협 볼륨 (3D 돔) | Arch 4 |

## 왜 Arch 1이 군사 RTT 레이어 대부분에 우위인가

### `prepare()`의 실제 셰이더 자유도

Arch 1의 `prepare()`는 CustomLayer의 `render()`와 본질적으로 동일한 능력을 가진다. 차이는 "스크린 대신 텍스처에 쓴다"는 것뿐. 다음이 모두 가능:

- **Multi-pass 효과**: pass1 → FBO_A, pass2 → FBO_B, final → tile.texture
- **Stencil 기반 마스킹**: 유효 coverage 외부 discard
- **복수 데이터 텍스처 샘플링**: 센서 데이터, DEM, 과거 프레임, 위협 DB
- **Depth testing**: 자체 depth attachment로 3D 효과
- **Blend mode 자유**: additive, subtractive, multiply
- **Time-based uniform**: sweep line, blinker 애니메이션

```ts
prepare() {
  const gl = this.map.painter.context.gl;
  const prevFB = gl.getParameter(gl.FRAMEBUFFER_BINDING);
  const prevVP = gl.getParameter(gl.VIEWPORT);

  gl.bindFramebuffer(gl.FRAMEBUFFER, this._fbo);

  for (const tile of tilesToRerender) {
    gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0,
                            gl.TEXTURE_2D, tile.texture.texture, 0);

    // Pass 1: exploration mask
    gl.useProgram(this._maskProgram);
    this.bindSensorData(gl);
    gl.drawElements(gl.TRIANGLES, ...);

    // Pass 2: blur
    gl.useProgram(this._blurProgram);
    gl.drawArrays(gl.TRIANGLES, 0, 6);

    // Pass 3: final composite
    gl.useProgram(this._compositeProgram);
    gl.drawArrays(gl.TRIANGLES, 0, 6);
  }

  gl.bindFramebuffer(gl.FRAMEBUFFER, prevFB);
  // ...
}
```

### RTT 자동 참여 → 다른 군사 레이어와의 정확한 합성

여러 RTT 레이어가 있을 때 MapLibre RTT 스택이 자동 합성:

- Fog of war (Arch 1) + SAM 엔벨로프 (Arch 1) + 이동 회랑 (Arch 2)
- 모두 동일 RTT FBO에 순서대로 기록 → terrain drape 하나로 합성

Arch 4 CustomLayer들은 이 합성에 참여 못 함 → 위에 덧그려지기만.

### 결정론적 프레임 타임

- `hasTransition()` 조건부 반환 + `_zoomEpsilon`/`_dataRevisionEpsilon` 재베이크 트리거로 변경 발생 시에만 재렌더
- 정지 상태에서는 `prepare()` 호출되어도 early-return → 제로 비용
- Arch 4는 매 프레임 mesh draw 발생 (비용 낮지만 고정)

### MapLibre 독립성

- Arch 1 의존 API: `Texture` 클래스 + `context.gl` + `context.bindFramebuffer.dirty`
- 매우 얕은 커플링 — `TextureBridge` 하나로 격리 가능
- Arch 4의 terrain API 커플링(3개 getter + 셰이더 공식 1개)보다도 단순

## 레이더 Sweep 애니메이션 — 두 가지 옵션

### 옵션 A: Arch 1 + 적극적 `hasTransition`

```ts
hasTransition(): boolean {
  return this._hasLiveAnimation;   // sweep 활성 시 항상 true
}

prepare(): void {
  const now = performance.now();
  if (now - this._lastBakeTime < this._minBakeInterval) return;

  for (const tile of this._visibleTiles) {
    this._rasterizeWithSweepAngle(tile, this._sweepAngle);
  }
  this._lastBakeTime = now;
}
```

- 장점: RTT 자동 참여, 코어 수정 불필요
- 단점: 타일 수 × 매 프레임 draw call (단순한 sweep은 per-tile 수 백 정점이므로 가볍다)

### 옵션 B: Arch 3 코어 포크

군사 프로젝트는 대부분 MapLibre fork를 유지하므로 현실적. [Arch 3 상세](../architectures/arch-3-customlayer-rtt-fork.md)에 80라인 수준의 변경 내용.

- 장점: Arch 1의 2-pass 블릿 제거 → 최대 성능
- 단점: fork 유지 비용

## 결정론적 프레임 타임 확보 패턴

1. **프레임 예산 고정**: 60fps 기준 16.67ms, 전술 디스플레이 권장 5-8ms
2. **프레임당 최대 처리 타일 수 제한**: `prepare()`에서 N개 이상이면 다음 프레임 이월
3. **우선순위 큐**: 화면 중앙 타일 먼저
4. **Throttle**: `_zoomEpsilon` 충분히 크게 (0.2 정도) → 불필요한 재래스터화 회피
5. **예열**: `onAdd` 시점에 현재 가시 타일 즉시 래스터화
6. **Graceful degradation**: 프레임 드롭 감지 시 해상도 자동 하향

## TerrainBridge / SourceBridge 격리 레이어

MapLibre 독립성 극대화를 위한 공유 추상 레이어. Arch 4는 `maplibregl.createTileMesh` (공개 API) 기반이므로 커플링이 얕고, 브리지 범위는 주로 Arch 1의 `Texture`/`Context` 추상화에 집중하면 된다. Arch 4에서 terrain elevation 확장이 필요한 경우에만 `getTerrainData`(@internal) 브리지가 필요.

```ts
interface SourceBridge {
  registerSourceType(name: string, cls: any): Promise<void>;
  addSource(id: string, spec: any): void;
  removeSource(id: string): void;
}

interface RenderContextBridge {
  gl(): WebGL2RenderingContext;
  createTexture(opts): TextureHandle;
  markFramebufferDirty(): void;
}
```

CI에서 브리지 단위 테스트 유지 → MapLibre 업그레이드 시 회귀 탐지.

## MIL-STD-2525 / APP-6 심볼과의 z-order 조율

### 시나리오 1: MapLibre 외부 심볼 엔진

- HUD 계층에서 직접 Canvas 2D 또는 WebGL로 심볼 렌더
- MapLibre canvas 위에 overlay
- Arch 1/2/3/4 모두 영향 없음 (심볼이 항상 최상단)

### 시나리오 2: MapLibre `symbol` 레이어 사용

- Style 순서 조정:
  - Arch 1/2/3 (RTT): `[base-raster, fog, threat, rail, symbol]` → symbol이 자동으로 위
  - Arch 4 (non-RTT): `[base-raster, symbol, custom-arch4]`처럼 symbol을 custom 앞에 배치

### 시나리오 3: 혼합

- Arch 1로 드레이프 효과 + MapLibre symbol + Arch 4로 공중 3D 유닛
- Style 순서: `[base, Arch 1 레이어들, symbol, Arch 4 레이어]`

## 다중 커스텀 소스 조율

여러 도메인 레이어(fog + coverage + threat + 3D 유닛)를 동시 활성화할 때:

- **공유 FBO 전략** (Arch 1): 각 소스가 독립 FBO 유지, `tile.texture`만 라이프사이클 관리
- **공유 프로그램 캐시** (모든 Arch): 동일 셰이더 variant 공유하여 compile 비용 절감
- **공유 mesh 캐시** (Arch 4): `maplibregl.createTileMesh` 결과를 여러 레이어가 공유 — `(granularity, pole flags, borders)` 키 기반 캐시를 모듈 스코프에 배치
- **레이어 순서로 합성 순서 제어**: style 순서로 blend 순서 명시. RTT 참여 (Arch 1) 레이어는 스택 합성, Arch 4 레이어는 translucent pass에서 뒤에 덧그려짐
- **데이터 원천 공유**: 센서 데이터 텍스처 하나를 여러 소스가 참조

## 검증 체크리스트

- [ ] MapLibre 독립성: Bridge 인터페이스 단위 테스트 통과
- [ ] 결정론적 프레임 타임: 1분 stress test에서 5-8ms 유지
- [ ] GPU 메모리 안정: 반복 zoom in/out 1분 후 증가 없음
- [ ] Z-order: 심볼이 항상 전술 레이어 위
- [ ] Fog + Coverage + Threat 동시 활성화 시 시각적 합성 정상
- [ ] Sweep 애니메이션 60fps 유지
- [ ] Globe 모드 + terrain 활성화 시 3D 유닛 위치 정확

## 참고 문서

- [Arch 1](../architectures/arch-1-custom-raster-source.md) — 대부분의 드레이프 효과 기본 선택
- [Arch 2](../architectures/arch-2-custom-vector-bucket-source.md) — 샤프 라인이 필요한 경우
- [Arch 3](../architectures/arch-3-customlayer-rtt-fork.md) — fork 유지 시 최적
- [Arch 4](../architectures/arch-4-customlayer-terrain-mesh.md) — 공중/3D 효과
- [Appendix A — RTT Mechanism](../appendix/a-rtt-mechanism.md)
- [Appendix B — Globe + Terrain](../appendix/b-globe-terrain-interaction.md)
- [FAQ Q21-23](../faq.md) — 군사 특화 질문

---

**See also**: [README](../README.md) · [Glossary](../glossary.md)
