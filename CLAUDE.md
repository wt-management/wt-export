# 해외수출 관리 사이트 (wt-export)

담당자가 Claude Code로 직접 수정·배포하는 repo입니다. main에 push하면 1~2분 내 라이브에 반영됩니다.
라이브: https://wt-management.github.io/wt-export/

## 구조
- **index.html 단일 파일** 사이트입니다. HTML/CSS/JS가 전부 이 파일 안에 있습니다 (약 270KB).
- 메뉴: 대시보드 / 선적 진행(칸반) / 일일 현황·일정 / 선적 대장 / 장비 추적 / 수출 분석 / SCM 리스크
- 화면 전환은 해시 라우팅(`#v=ledger` 등), 각 화면은 `render<이름>()` 함수가 그립니다.
- 3개 팀 공동 사용: 해외영업(요청) → 영업전략파트(서류 준비) → 글로벌물류관리팀(출고).

## 데이터 (절대 HTML에 넣지 말 것)
- 선적 데이터는 Supabase `cons_cache` 테이블:
  - key=`wt_export` — 진행 중 + 당해연도 건 (객체 배열)
  - key=`wt_export_hist` — 2024~2025년 이관분 (컬럼배열 압축 형식, `HCOLS` 순서와 반드시 일치)
- **거래처·금액 등 실데이터를 index.html에 직접 임베드 금지** — 이 repo는 public이라 그대로 노출됩니다.
- **Supabase service key(sb_secret_...)를 커밋 금지.** 코드에는 publishable key만 사용.

## 수정 규칙
1. 수정 전 `git pull` 먼저 (다른 사람 작업분 반영)
2. 수정 후 **인라인 JS 문법검증 필수** — 문법오류 1건이면 사이트 전체가 죽습니다:
   ```bash
   node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const re=/<script(?![^>]*src)[^>]*>([\s\S]*?)<\/script>/g;let m,all='';while((m=re.exec(h))){all+=m[1]+'\n;\n';}fs.writeFileSync('_check.js',all);" && node --check _check.js && del _check.js
   ```
3. `git add index.html && git commit -m "..." && git pull --rebase && git push`
4. 배포 후 사이트 열어서 강력 새로고침(Ctrl+Shift+R)으로 확인
5. 잘못 올렸으면 `git revert HEAD && git push` 로 즉시 되돌리기

## 건드리면 안 되는 것
- `saveData()` 저장 로직 — live/hist 두 저장소를 나눠 쓰고 동시수정 충돌(`_base`/`updAt`)을 검사합니다. 여기를 고치면 남의 수정이 조용히 사라질 수 있습니다.
- `_diffFields` / `revertHist` — 변경 이력·되돌리기의 근간입니다.
- `WT_ERRLOG_V1` 스니펫(오류수집)과 사이트 이동(스위처) 블록 — 전 사이트 공용 모듈이므로 개별 수정 금지.
- `HCOLS` 배열 순서 — 2024~25 이관 데이터의 컬럼 순서입니다. 바꾸면 과거 데이터가 통째로 어긋납니다.
- `FXPER` 환율 상수 — 바꾸면 전 화면 금액이 동시에 바뀝니다. 변경 시 반드시 공유.

## 기안서 PDF 인식 (draftFromPdf)
- 출고기안서 PDF를 OCR(tesseract.js)해서 입력폼을 자동으로 채우는 기능입니다. 문서 3페이지 기준 2~3분 걸립니다.
- 판정 원칙: **표 합계 = 문서 총액이면 표를 정답으로 신뢰**하고 파일명 보완을 생략합니다(`d.tblOk`).
  표 인식이 실패(총액과 2% 초과 차이)하고 파일명 품목 합계가 총액과 맞으면 파일명 쪽을 채택합니다(`d.byName`).
- 정확도를 손보려면 `_detailRows`(장비정보 표 역파싱)와 `_draftFromText`를 봅니다. **총액과 대조해 검증되는 규칙만 추가할 것** — 이름 유사도 추측으로 행을 만들면 중복행이 생깁니다(과거 실제 발생).

## 권한
- 사이트 접속: crm_profiles의 `role=admin` 또는 access에 `intl`/`korea`/`export` 보유
- 데이터 수정: 로그인한 사람 전원(변경 전/후 값이 건별 이력에 남고 ↩ 되돌리기 가능)
- 삭제: 관리자 + access `export` 보유자
