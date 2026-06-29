# 설정 내보내기/가져오기(백업·복원) 설계

작성일: 2026-06-29 · 대상: `kuji-app/index.html` (단일 PWA)

## 목적
운영 기기(아이패드/안드로이드)의 전체 세팅을 **파일 1개로 백업**하고, 다른/새 기기에 **그대로 복원**한다. PC에서 1회 세팅 → 각 기기 import. 기기 교체·재설치·데이터 손실 대비.

## 핵심 결정 (확정)
- **범위**: 전체(영상 포함) 완전 복제 — 설정 3종 + 미디어 전부.
- **포장**: 전용 바이너리 컨테이너 `.kuji` (외부 라이브러리 0, base64 부풀림 없음, iOS 메모리 안전).
- **가져오기 = 전체 교체(restore)** — 2단계 확인 후 덮어쓰기, 완료 시 앱 자동 새로고침.

## 담는 데이터
- localStorage 3키: `kuji.config.v1`(cfg), `kuji.stock.v1`, `kuji.log.v1` — 원본 문자열 그대로.
- IndexedDB `kuji-media`의 **모든 항목**:
  - 이미지 = dataURL 문자열(`img_*` 등) → 텍스트로 저장(작음).
  - 미디어 = `{buf:ArrayBuffer,type}`(BGM·효과음·대기영상·NPC) → 원본 바이너리로 저장. (레거시 Blob도 변환 처리)

## 파일 포맷 `.kuji`
```
[0..3]   매직 "KUJI" (ASCII)
[4..7]   uint32 LE = manifest JSON 바이트 길이 M
[8..8+M) manifest JSON (utf-8)
[8+M.. ] 미디어 페이로드 (각 항목 원본 바이트 연접)
```
manifest = `{ v:1, app:'kuji', exportedAt, ls:{cfg,stock,log}, media:[{key, kind:'image'|'buf', type, off, len}] }`
- off/len = 페이로드 영역 내 오프셋/길이.

## 동작
### 내보내기 `exportBackup()`
1. localStorage 3키 수집(null 허용).
2. IndexedDB 전 항목 순회 → 종류별 바이트화, off/len 기록.
3. `Blob([매직, M(4B), manifestBytes, ...parts])` 생성, `kuji-backup-YYYYMMDD-HHmm.kuji`.
4. iOS: `navigator.share({files:[File]})` 우선(파일앱/에어드랍/메일). 실패·미지원 시 `<a download>` 폴백.

### 가져오기 `importBackup()`
1. `pickFile()`(accept 미지정 — iOS 회색화 방지)로 파일 선택.
2. ArrayBuffer 읽기 → 매직/버전 확인(불일치 시 에러 토스트).
3. manifest 파싱 → "현재 설정·미디어를 모두 덮어씁니다. 계속?" 2단계 확인.
4. localStorage 3키 복원.
5. IndexedDB 미디어 **clear** 후, 각 항목을 슬라이스하여 **하나씩 순차 기록**(메모리 피크 최소화):
   - image → `idbPut(key, dataURL문자열)`
   - buf → `idbPut(key, {buf, type})`
6. `location.reload()` → 부팅 시 새 상태로 재초기화.

## 신규 헬퍼
- `idbAll()` 전체 {key,value} 열거 · `idbClear()` 스토어 비우기
- u32 LE 읽기/쓰기 · TextEncoder/Decoder

## UI (관리자 → 설정 탭)
- "설정 백업/복원" 카드: `📤 설정 내보내기` · `📥 설정 가져오기` 버튼 + 안내문(가져오기는 전체 교체, 영상 많으면 파일 큼).

## 안전장치 / 호환
- 매직넘버·`v` 버전 체크로 잘못된/구버전 파일 차단.
- 가져오기 2단계 확인, 진행 스피너/토스트, try/catch 에러 안내.
- 기존 데이터 모델(번들 포함) 그대로 직렬화 — 추가 변경 불필요.
- `sw.js` 캐시 버전 상향.

## 범위 외
- 부분 선택 백업, 자동 클라우드 동기화, 다중 기기 실시간 컨트롤타워(로드맵).
