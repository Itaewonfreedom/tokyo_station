# 🛠 개선 제안 (Improvement Proposals)

이 문서는 도쿄역 철도 시뮬레이터의 재현도를 한 단계 끌어올리기 위해
필요한 외부 리소스와 구현 방법을 정리한 제안서입니다.

---

## 1. 신칸센 "앞대가리" — 진짜 3D 모델 도입하기

현재 노즈는 코드로 깎은 파라메트릭 로프트(단면 26×26)라 실루엣은 비슷하지만
실물의 복잡한 곡면(3차원 자유곡면, 헤드라이트 함몰부, 커플러 커버 등)은
수식으로 따라가기 어렵습니다. **모델 파일을 구해서 로드하는 것이 정답**입니다.

### 어떤 파일 형식을 구해야 하나

| 형식 | 권장도 | 비고 |
|---|---|---|
| **glTF 2.0 (.glb)** | ★★★ 최우선 | Three.js 공식 권장. 텍스처/재질 내장, `GLTFLoader` 한 줄로 로드 |
| FBX (.fbx) | ★★ | `FBXLoader` 있음. 구하면 Blender에서 .glb로 변환 권장 |
| OBJ (+MTL) | ★★ | 재질 단순. 변환 쉬움 |
| STL | ★ | 색/UV 없음 — 비추천 |
| BVE/Train Simulator 전용 포맷 (.x, .csv 등) | △ | 철도 시뮬 커뮤니티에 E5 모델이 많지만 변환 작업 필요 (Blender 임포터 사용) |

**요청 스펙**: E5계(하야부사) 선두차 1량 또는 노즈 부분만, 삼각형 5만개 이하
(LOD용으로 1만개 버전도 있으면 최상), 텍스처 2048² 이하 PNG/JPG,
Y-up, 실측 스케일(선두차 약 26m) 또는 명시된 스케일.

### 어디서 구하나

1. **Sketchfab** (sketchfab.com) — "E5 Shinkansen", "Hayabusa train" 검색.
   CC-BY / CC0 라이선스 필터를 켜고 **glTF 다운로드** 지원 모델 선택.
2. **BlenderKit / CGTrader / TurboSquid** — 유료지만 품질 높음. 라이선스에
   "게임/리얼타임 사용" 포함 여부 확인.
3. **3D Warehouse**(SketchUp) — 무료 모델 다수. Collada(.dae) 내보내기 →
   Blender에서 정리 → .glb.
4. **직접 제작**: Blender에서 도면(side/top blueprint) 깔고 Subdivision
   Surface로 모델링. E5 도면은 철도 동호회 사이트에서 구할 수 있음.

> ⚠️ 라이선스 주의: JR 동일본 차량 디자인은 상표/의장 보호 대상입니다.
> 개인·가정용(아기 감상용 ✓)은 문제없지만, 공개 배포 시 모델 자체의
> 라이선스(CC-BY는 저작자 표기)를 README에 명시하세요.

### 코드 통합 방법 (이미 구조 준비됨)

```js
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js'; // 압축 모델용

const loader = new GLTFLoader();
loader.load('models/e5_nose.glb', (gltf) => {
  const nose = gltf.scene;
  nose.scale.setScalar(1.0);            // 모델 단위가 m가 아니면 보정
  nose.traverse(o => { if (o.isMesh) o.castShadow = true; });
  // buildTrain()의 addEnd()에서 getNoseGeo() 대신 nose.clone() 사용
});
```

- 교체 지점: `index.html`의 `addEnd()` 안 `isShink` 분기 — `getNoseGeo(T.noseLen)`
  메시 대신 로드한 `nose.clone()`을 붙이면 끝.
- 단일 HTML 유지가 목표라면 .glb도 base64 data URI로 임베드 가능
  (`loader.parse(arrayBuffer, ...)`). 5MB 이하 권장.
- 모델이 무거우면 `DRACOLoader`(geometry 압축, 1/5 크기) 병용.

---

## 2. "가땅고똥" 조인트 사운드 (rail joint clatter)

> ✅ **방법 A는 구현 완료** (index.html `sound.joint()`): 25m 이음매마다
> 대차 2연타 리듬을 속도 반비례로 스케일, 카메라 거리 감쇠 적용.
> 더 리얼한 질감을 원하면 아래 방법 B(실녹음 샘플)로 교체.

열차가 레일 이음매를 지날 때 나는 `가땅-고똥 … 가땅-고똥` 소리.
대차(bogie)가 차량 앞뒤 2개라서 **2연타 × 차량 수**로 반복되는 게 특징입니다.

### 방법 A — WebAudio 합성 (외부 파일 불필요, 현재 구조와 동일) ★권장 시작점

```js
// 노이즈 버스트 한 방 = "땅"
function clack(when, vol) {
  const dur = 0.06;
  const buf = ctx.createBuffer(1, ctx.sampleRate * dur, ctx.sampleRate);
  const d = buf.getChannelData(0);
  for (let i = 0; i < d.length; i++)
    d[i] = (Math.random() * 2 - 1) * Math.exp(-i / (d.length * 0.25));
  const src = ctx.createBufferSource(); src.buffer = buf;
  const bp = ctx.createBiquadFilter();          // 금속성 톤
  bp.type = 'bandpass'; bp.frequency.value = 900 + Math.random() * 300; bp.Q.value = 1.2;
  const g = ctx.createGain(); g.gain.value = vol;
  src.connect(bp).connect(g).connect(ctx.destination);
  src.start(when);
}
// "가땅고똥" = 대차 1 (땅-땅) + 잠시 후 대차 2 (땅-땅)
function gatangotong(when, vol) {
  clack(when, vol); clack(when + 0.12, vol * 0.9);          // 가땅
  clack(when + 0.55, vol * 0.95); clack(when + 0.67, vol * 0.85); // 고똥
}
```

- **트리거 로직**: 레일 이음매를 25m 간격이라 치고, `updateTrain()`에서
  `t.sHead`가 25의 배수를 넘을 때마다 `gatangotong()` 호출.
  간격(0.12s, 0.55s)은 `속도에 반비례`로 스케일하면 가감속 시 리듬이
  자연스럽게 늘어졌다 빨라집니다: `dt = 축간거리 / t.speed`.
- **거리 감쇠**: `vol = 1 / (1 + dist²/400)` — 카메라와 열차 거리로 음량 조절.
  따라가기 모드에서 가장 크게 들리는 효과가 공짜로 생깁니다.

### 방법 B — 실제 녹음 샘플 (리얼함 최상)

1. **freesound.org**에서 CC0 검색어: `train track joint`, `train clickety clack`,
   `railroad joint bump`. 1~2초짜리 원샷(one-shot) 샘플이 좋음.
2. OGG/MP3 (96~128kbps mono, ~30KB)로 변환 → base64로 HTML에 임베드
   (현재 텍스처 아틀라스와 같은 방식 → file:// 유지).
3. 재생: `AudioBufferSourceNode` + `playbackRate`를 0.9~1.1 랜덤
   (반복감 제거) + **`PannerNode`(HRTF)** 로 열차 위치에 3D 정위:

```js
const panner = ctx.createPanner();
panner.panningModel = 'HRTF';
panner.positionX.value = train.x;  // 매 프레임 열차 위치로 갱신
```

### 보너스: 함께 넣으면 좋은 소리

- **VVVF 인버터 가속음** (E235 특유의 우우웅~ 상승음): 톱니파 오실레이터
  주파수를 `speed`에 비례시켜 sweep — 합성으로 충분히 그럴듯함.
- **레일 울림 (rumble)**: 갈색 노이즈 + lowpass 200Hz, 음량을 근처 주행
  열차 수·속도에 비례.
- **신칸센 통과풍**: 화이트노이즈 swell 2초.

---

## 3. 기타 백로그 (우선순위순)

1. **케이요선·요코스카/소부 지하 승강장** — 위성에는 안 보이지만 배선도에
   있는 지하 4선. 반투명 단면 컷어웨이로 보여주면 교육 효과 큼.
2. **PBR 텍스처 강화** — 현재 albedo만 사용. 아틀라스에 roughness/normal
   맵을 추가로 받으면 (`material.normalMap`) 햇빛에서 차체 입체감 상승.
3. **실시간 시간표 모드** — 야마노테 2~4분 간격 등 실제 배차로 스폰.
4. **승객 스프라이트** — 승강장에 빌보드 사람 몇 명, 문 열리면 타고 내림.
5. **날씨** — 비(파티클)+젖은 노면(roughness↓), 눈 쌓인 지붕.
6. **WebGPURenderer 전환** — three.js r160+ 실험 지원. 드로우콜 많은 이
   씬(열차 11편성)에서 프레임 안정성 개선 여지.
7. **모바일 조작 보강** — 따라가기 중 두 손가락 스와이프로 카메라 오프셋 조절.
