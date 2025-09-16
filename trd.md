# TRD: 비대면 채용 면접 시스템 (웹, 파일 기반, Next.js + Supabase Storage)

## 1. 범위 및 목표
- 범위
  - 지원자용: 로그인(개인화 링크 + 비밀번호) → 랜딩(장치 점검/동의) → 면접 진행(오프닝 → Q1~Q4) → 종료/제출
  - 관리자용: 채용 라운드 생성/편집, 질문 영상 업로드(공통3/개별1), 지원자 등록/초대 발송, 상태 보드, 결과 열람(영상/스크립트), LLM 평가
  - 파일 기반 데이터 관리: DB 미사용, 모든 데이터는 파일(JSON, JSONL, 텍스트, 동영상)로 관리
- 핵심 목표
  - 안정적인 연속 녹화(WebM), 질문 영상 동시 재생, 90초 타이머 정확도(±0.2초), 실시간 STT(클라이언트) 저장
  - 네트워크 중단 시 자동 보존 및 상태=중단됨 처리
  - 고유 10자리 링크값을 URL/디렉토리/파일 접두에 공통 적용

## 2. 아키텍처 및 기술 스택
- 플랫폼: 웹 브라우저(데스크톱 크롬/엣지 최신)
- 프레임워크
  - Next.js 14(App Router, SSR), TypeScript, Node.js 20
  - 커스텀 Node 런타임(self-hosted)으로 WebSocket 서버(ws) 구동
- 저장소(파일 기반)
  - Supabase Storage (오브젝트 스토리지만 사용, DB 미사용)
  - 모든 스토리지 접근은 Next.js 백엔드 경유(클라이언트에서 직접 접근 금지)
  - 서버 임시 디스크: 청크 조립/후처리(FFmpeg) 용
- 미디어/스트리밍
  - 클라이언트: MediaRecorder(WebM, VP8/VP9, 720p 중간 화질), WebAudio
  - 서버: ws(WebSocket), ffmpeg-static(완료 후 컨테이너 정합성 리뮤즈)
- STT
  - 1순위: 브라우저 Web Speech API(한국어/영어 자동감지)
  - 2순위(옵션/폴백): 서버 측 일괄 변환(OpenAI Whisper API), 실시간 요건 미보장
- 이메일
  - Nodemailer + SMTP(예: SendGrid/SES/사내 SMTP)
- LLM 평가
  - OpenAI SDK (gpt-4o-mini 또는 동급 모델)
- 보안/인증
  - 지원자: 개인화 링크(랜덤 10자리) + 6자리 숫자 비밀번호
  - 관리자: 세션 기반 로그인(iron-session), 단일 관리자 계정(.env에 해시)
  - 비밀번호 해시: argon2
  - 속도 제한(rate limit): 메모리 기반 토큰 버킷(인스턴스 로컬)
- 기타 라이브러리
  - 파일 업로드 파서: busboy
  - PDF 내보내기: pdfkit
  - 상태 머신(UI 흐름 엄격성): xstate(선택)
  - 이메일 템플릿: handlebars

## 3. 모듈 구성 및 책임
- Applicant Web (지원자)
  - 로그인, 랜딩(장치/동의), 면접 플레이어(오프닝/질문 재생), 타이머, 녹화/WS 송신, STT 송신, 종료 제출
- Admin Web (관리자)
  - 채용 라운드 CRUD, 공통 질문 업로드(3개), 지원자 등록/일괄 업로드, 개별 질문 업로드, 초대 이메일 발송, 상태 보드, 결과 열람, LLM 평가/내보내기
- Backend API (Next.js Route Handlers)
  - 인증(지원자/관리자), 업로드/다운로드 서명/프록시, 녹화 수신(WS), STT 수신(HTTP), 상태 관리, 평가 실행, 이메일 발송
- Storage Adapter
  - Supabase Storage 파일 읽기/쓰기/목록, 원자적 쓰기, 파일 무결성 검증(해시/크기), 경로 규칙 강제
- Recorder/WS Server
  - 세션별 파일 핸들 관리, 청크 순서/오프셋 보장, 완료 시 FFmpeg 리뮤즈, 에러/중단 처리
- Evaluator
  - 프롬프트 템플릿 로딩 → OpenAI 호출 → 결과 파일 저장(JSON/PDF)
- Telemetry/Logs
  - 이벤트 로그(JSONL), 상태 전이 기록, 오류 리포팅

## 4. 데이터 모델 및 파일 레이아웃
- 식별자 규칙
  - candidateToken: [a-zA-Z0-9] 랜덤 10자리(충돌확률 낮춤)
  - pin: 6자리 숫자(문자열), 서버에 argon2 해시로 저장
  - 파일명 접두: candidateToken_
- 디렉토리 구조(Supabase Storage 내)
  - recruitments/{recruitmentId}/
    - meta.json
    - common_questions/
      - 1.mp4|webm
      - 2.mp4|webm
      - 3.mp4|webm
    - candidates/
      - {candidateToken}/
        - meta.json
        - status.json
        - logs/events.jsonl
        - logs/error.jsonl
        - individual_question.mp4|webm
        - interview/
          - {candidateToken}_recording.webm
          - {candidateToken}_recording.remuxed.webm (완료 후)
          - scripts/
            - q1.txt
            - q2.txt
            - q3.txt
            - q4.txt
            - combined.json
          - marks.json (오프닝/질문별 타임스탬프, 타이머 로그)
          - stt_raw.jsonl (실시간 STT 이벤트)
        - evaluation/
          - result.json
          - report.pdf
- 파일 스키마(요약)
  - recruitments/{id}/meta.json
    - { id, name, description, createdAt, retentionMonths, languagePolicy: "ko-en-auto", openingVideoPath, ownerEmail }
  - candidates/{token}/meta.json
    - { recruitmentId, token, number, name, email, pinHash, createdAt, linkUrl }
  - candidates/{token}/status.json
    - { state: "not_started|landing|in_progress|interrupted|completed|error|locked", loginFailedCount, lastLoginAt, startedAt, finishedAt, networkIssues: [timestamps], attempt: { allowed: true }, notes }
  - candidates/{token}/interview/marks.json
    - { opening: {start,end}, questions: [{id:1,start,end},{id:2,...}], timer: [{qid, start, end, durationMs}], media: { codec, width, height, fps, bitrate } }
  - candidates/{token}/interview/scripts/combined.json
    - { segments: [{qid, start, end, text}], language: "ko|en|auto", coverage: {voiceMs, sttMs, ratio} }
  - candidates/{token}/logs/events.jsonl
    - 각 줄: {"ts": ISO8601, "type": "...", "payload": {...}}
  - evaluation/result.json
    - { model, templateId, scores: [{qid, criteria:[{name, score, rationale}], total}], summary, createdAt, promptVersion }
- 검증
  - 모든 파일은 UTF-8, JSON은 쉼표/키 검증
  - 레코딩 완료 후 remuxed 파일 존재 확인

## 5. API 명세 (Next.js Route Handlers, 서버 전용)
- 인증
  - POST /api/auth/applicant/login
    - req: { token: string(10), pin: string(6) }
    - res: { ok: true, session: { token, expiresAt } } | { ok:false, reason }
    - 효과: status.json.loginFailedCount 증가/초기화, 5회 실패 시 state=locked
  - POST /api/auth/admin/login
    - req: { email, password }
    - res: { ok:true, session:{ role:"admin" } } | { ok:false }
- 리소스 조회
  - GET /api/recruitments/:id
    - res: meta.json + 질문 파일 경로 목록
  - GET /api/candidates/:token/summary (관리자/본인)
    - res: meta/status 요약, 진행 상태, 파일 존재 여부
- 업로드
  - POST /api/admin/recruitments
    - req: { name, description, retentionMonths, openingVideo(file)?, commonQ1(file), commonQ2(file), commonQ3(file) }
  - POST /api/admin/candidates/batch
    - req: CSV(File) 또는 JSON: [{number,name,email}]
    - res: [{token, pin(평문은 이메일용으로만 응답), emailStatus}]
  - POST /api/admin/candidates/:token/individual-question
    - req: file(mp4|webm)
- 이메일
  - POST /api/admin/invitations/send
    - req: { recruitmentId, candidates:[token], deadline? }
- 면접 진행
  - POST /api/interview/:token/start
    - 효과: state=“in_progress”, marks.opening.start 기록
  - POST /api/interview/:token/mark
    - req: { event: "opening_end|q_start|q_end|timer_start|timer_end", qid? , ts }
  - POST /api/interview/:token/stt
    - req: { qid, start, end, text, lang, isFinal }
    - 효과: stt_raw.jsonl append, scripts/q{n}.txt 및 combined.json 머지
  - POST /api/interview/:token/finish
    - 효과: 녹화 종료 신호, FFmpeg remux 실행, state=“completed” 또는 에러시 “error”
- 평가
  - POST /api/admin/eval/:token/run
    - req: { templateId, weights?, model? }
    - res: { ok:true, resultPath }
- 결과 열람
  - GET /api/admin/candidates/:token/artifacts
    - res: 파일 위치 목록(재생/다운로드용 프록시 URL)

- 에러 처리 공통 규약
  - JSON: { ok:false, code:"ERR_xxx", message, hint? }
  - 429: rate limit, 401/403: 인증/권한 오류, 409: 상태충돌, 500: 내부 오류

## 6. 프런트엔드 흐름/상태 머신
- 지원자 상태 머신(클라이언트 레벨)
  - states: login → landing(deviceReady,consentChecked) → ready → openingPlaying → q1 → q2 → q3 → q4 → finishing → done
  - transitions:
    - openingPlaying → q1: 오프닝 종료 또는 “1번 질문” 클릭 시(오프닝은 1회 재생)
    - qn → q(n+1): 타이머 종료 또는 버튼 클릭(선행 단계 완료 필수)
    - q4 → finishing: “면접 종료”
  - 가드:
    - 순서 외 버튼 비활성화
    - 장치/동의 미완료 시 진행 불가
- 타이머
  - Web Worker 기반 카운트다운 90초, tick 100ms, 정확도 ±0.2s
  - 0초 시점 시각적(색/진동) + 청각적 비프

## 7. 미디어 녹화/스트리밍 파이프라인
- 클라이언트
  - getUserMedia(720p, 30fps), MediaRecorder(MIME: video/webm; codecs=vp8, audio=opus)
  - timeslice=1000ms로 ondataavailable 청크 생성 → WebSocket 전송
  - Q1 시작 시 MediaRecorder start, Q4 종료 또는 “면접 종료” 시 stop
- 서버(WS)
  - 경로: /ws/record/:token
  - 세션 시작 시 {token} 검증 및 파일 핸들 오픈({token}_recording.webm.tmp)
  - 수신 청크 순서대로 append(원자적 쓰기)
  - 클라이언트 close/finish 수신 시 파일 close → FFmpeg remux → {token}_recording.remuxed.webm 생성 → tmp 제거
- 복원력
  - 네트워크 단절 시 현재까지 파일은 유효(append flush), state=“interrupted”
  - 중단된 세션은 관리자가 재시도 허용 시 새 세션으로 재녹화(파일 버전 관리: suffix .v2 등)

## 8. STT 파이프라인
- 1순위: Web Speech API(ko/en auto)
  - interim/final 결과를 질문 경계로 분리 저장
  - API 이벤트마다 POST /stt (qid, start, end, text, isFinal)
- 음성 커버리지 산출
  - WebAudio로 음성 레벨 샘플링(예: 50ms 간격, 임계치 기반 VAD 근사) → voiceMs
  - combined.json 생성 시 sttMs(텍스트 구간 길이 합산) 및 ratio=sttMs/voiceMs 기록
- 폴백(브라우저 미지원시)
  - 질문 종료 후 각 질문 오디오 추출(서버 측 FFmpeg demux) → Whisper API 전송 → scripts/q{n}.txt 생성
  - PRD상 실시간 요구는 낮춤(표시 문구 제공)

## 9. LLM 평가 파이프라인
- 입력
  - combined.json, marks.json, 질문 메타(제목/의도)
- 구성
  - 템플릿 선택(templateId), 기준/가중치(의사소통, 논리성, 관련성 등)
- 처리
  - OpenAI 모델 호출(temperature 저온, deterministic 우선), 근거 요약 포함
- 산출
  - evaluation/result.json(점수/근거/버전)
  - report.pdf(요약/점수 테이블, 타임마커 링크)
- 재현성
  - promptVersion, templateId, model, weights 기록

## 10. 인증/권한
- 지원자
  - 링크: /i/{token}, 패스워드 6자리
  - 5회 실패 시 state=locked, 30분 후 자동 해제 또는 관리자 해제
  - 세션 쿠키(iron-session): { token, role:"applicant" }
- 관리자
  - /admin 로그인(이메일/비밀번호), 세션 쿠키: { role:"admin" }
  - 관리자 비밀번호 해시(argon2) .env로 주입
- 권한 경계
  - /admin/*: 관리자만
  - /api/interview/*: 해당 token 세션만

## 11. 이메일 발송
- 트리거: 후보 등록/일괄 업로드 완료 시
- 템플릿 변수: {name, recruitmentName, linkUrl, pin, deadline, supportEmail}
- 제목/본문: 링크(URL), 6자리 비밀번호, 마감기한, 장치 안내
- 반송/실패 처리: 응답 로그에 기록, 관리자 대시보드 표시

## 12. 네트워크/오류 처리 정책
- 면접 중 네트워크 단절
  - WS 끊김 → 파일 flush → state=“interrupted”
  - 지원자 화면: 중단 안내, 재시도 불가(관리자 승인 필요)
- 타이머/영상 오류
  - 질문 영상 로딩 실패 → 1회 자동 재시도, 실패 시 안내
- 로그인 속도 제한
  - IP/토큰별 토큰 버킷(예: 10 req/min)
- 파일 무결성
  - 업로드 후 크기/해시 기록, 재생 불가시 FFmpeg 재리뮤즈 시도

## 13. 비기능 요구사항 구현 방안
- 성능
  - 초기 화면 ≤3s: 코드 분할, lazy load, 캐시 헤더
  - 질문 재생 ≤2s: video preload="metadata", 서버 range 요청 지원
- 안정성
  - MediaRecorder 단일 인스턴스 유지, 청크 주기=1s
  - WS 백오프 재연결(로그인/대기 중에만), 녹화 중에는 재연결 비사용
- 보안
  - HTTPS 필수, HSTS, 쿠키 Secure/HttpOnly/SameSite=Lax
  - 로그에 PIN 평문 금지
  - Supabase Service Key 서버 보관만
- 접근성
  - 키보드 내비게이션, 대비 준수, 타이머 ARIA 라이브리전

## 14. 프로젝트 폴더 구조
```
/app
  /i/[token]        # 지원자 플로우(로그인/랜딩/면접)
  /admin            # 관리자 대시보드
  /api
    /auth/applicant/login/route.ts
    /auth/admin/login/route.ts
    /recruitments/[id]/route.ts
    /candidates/[token]/summary/route.ts
    /admin/recruitments/route.ts
    /admin/candidates/batch/route.ts
    /admin/candidates/[token]/individual-question/route.ts
    /admin/invitations/send/route.ts
    /interview/[token]/start/route.ts
    /interview/[token]/mark/route.ts
    /interview/[token]/stt/route.ts
    /interview/[token]/finish/route.ts
    /admin/eval/[token]/run/route.ts
    /admin/candidates/[token]/artifacts/route.ts
/components
  /ui
  /interview
  /admin
/lib
  /storage/supabaseClient.ts
  /storage/storageAdapter.ts
  /auth/session.ts
  /media/wsClient.ts
  /stt/webSpeech.ts
  /eval/openai.ts
  /email/mailer.ts
  /pdf/report.ts
  /ffmpeg/remux.ts
  /rateLimit/memory.ts
  /validators/schemas.ts
/server
  /ws/index.ts         # ws 서버 초기화(Next.js 서버와 동일 프로세스)
  /jobs/ffmpegQueue.ts # 후처리 큐(단순 in-memory)
/public
  /videos/placeholders
/config
  /prompts
  /email-templates
/scripts
  /ops
  /migrate
```

## 15. 배포/운영
- 호스팅: Node.js self-hosted(WS 필요), Docker 권장
- 환경변수
  - SUPABASE_URL, SUPABASE_SERVICE_KEY
  - SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASS, MAIL_FROM
  - ADMIN_EMAIL, ADMIN_PASSWORD_HASH
  - OPENAI_API_KEY
  - APP_BASE_URL
- 런타임
  - Next.js 서버 + ws 서버 동프로세스 시작
  - /server/jobs/ffmpegQueue: 녹화 완료 이벤트 수신 후 remux 실행
- 백업/보존
  - 보존기간: recruitments/{id}/meta.retentionMonths 기준
  - 주기적 무결성 점검(파일 존재/크기)

## 16. 작업 분해(원자 단위, 우선순위/의존성)
- M1. 스토리지 어댑터(Supabase Storage R/W, list, exists)
  - 의존성: SUPABASE_* env
  - 검증: 파일 업/다운/목록 API 통과
- M2. 식별자/암호화 유틸(token10 생성, PIN argon2 해시/검증)
  - 검증: 유니크 충돌률 샘플링, 해시/검증 왕복
- M3. 지원자 인증 API(/auth/applicant/login)
  - 의존성: M1, M2
  - 검증: 5회 실패 잠금, 세션 쿠키 발급
- M4. 관리자 인증 API(/auth/admin/login)
  - 의존성: M2
  - 검증: 올바른 세션 생성
- M5. 채용 생성/질문 업로드 API
  - 의존성: M1
  - 검증: meta.json 생성, 질문3 업로드
- M6. 지원자 등록/일괄 업로드 + 초대 발송
  - 의존성: M1, M2, 메일러
  - 검증: token/pin 생성, 이메일 로그
- M7. 면접 플레이어 UI(오프닝/질문/버튼 흐름)
  - 검증: 순서 강제, 비활성화 로직
- M8. 타이머(Worker, 90초, 시그널)
  - 검증: ±0.2s 이내
- M9. 녹화 클라이언트(MediaRecorder, WS 송신)
  - 의존성: M7, WS 서버
  - 검증: 5분 녹화 수집, 손실 0
- M10. WS 서버(append + remux)
  - 의존성: ffmpeg-static
  - 검증: 출력 remuxed.webm 정상 재생
- M11. STT 클라이언트(Web Speech API) + 서버 저장
  - 검증: 질문 경계 분리, combined.json 생성
- M12. 면접 종료 처리(API finish, 장치 해제)
  - 검증: state=completed, 자원 해제
- M13. 상태 보드(미응시/진행중/중단됨/완료/오류)
  - 검증: status.json 반영 정확
- M14. 결과 열람(영상 플레이어 + 스크립트 동기화)
  - 검증: 타임스탬프 오차 ≤1s
- S15. LLM 평가 실행/저장/내보내기
  - 검증: 동일 답변 다른 템플릿 점수 변화
- S16. 알림(에러/중단 표시), 재발송
  - 검증: 중단 이벤트 후 대시보드 지시등
- M17. 네트워크 중단 정책 구현
  - 검증: 중단 시 파일 보존, state=interrupted
- M18. 로깅(JSONL), 감사 추적
  - 검증: 필수 이벤트 기록

우선순위: M(필수) → S(선호). 의존성은 상기 명시.

## 17. 검증 기준(수용/테스트)
- 인증
  - 올바른 token+pin: 성공률 ≥99%, 잘못된 pin 5회 → 잠금
- 진행/타이머
  - 각 질문 시작 시 타이머 즉시 90s, 0s 알림
- 녹화/미디어
  - 단일 remuxed.webm 생성, 무결성 검사(ffprobe 메타 파싱)
- STT
  - 질문별 스크립트 파일 존재, combined.json 세그먼트 수 ≥ 발화 수
  - 커버리지 ratio ≥ 0.95(테스트 음성 기준)
- 종료
  - 장치 해제, 세션 종료, 저장 성공 표시
- 관리자
  - 상태 보드 실시간/주기 반영, 필터/검색 동작
  - 결과 열람 시 타임마커/스크립트 동기화 오차 ≤1s
- 평가
  - 결과 JSON/PDF 저장, 템플릿 변경 시 재계산
- 네트워크 중단
  - 중단 직전까지의 녹화/스크립트 유실 0

## 18. 보안/프라이버시
- 전송: HTTPS/TLS, HSTS
- 저장: PIN 해시(argon2), 서비스 키 서버 전용
- 접근제어: 세션 쿠키 HttpOnly/Secure
- 로깅: 민감정보(원문 PIN) 비기록
- 링크 토큰 엔트로피: 62^10 조합(충분한 무작위성)

## 19. 인터페이스 명세(요약)
- WebSocket 메시지(바이너리)
  - 헤더(초기 1회, JSON): { token, codec, width, height, fps, bitrate }
  - 이후: 순수 바이너리 청크
- STT HTTP 메시지
  - Content-Type: application/json
  - body: { qid:int(1-4), start:ms, end:ms, text:string, lang?:string("ko"|"en"), isFinal:boolean }

## 20. 운영 절차/장애 대응
- 파일 재생 불가
  - 재리뮤즈(ffmpeg -c copy), 실패 시 관리자 알림
- 중단/오류
  - 상태=interrupted/error, 대시보드 경고, 재시도 허용 토글
- 보존/삭제
  - retentionMonths 만료 시 일괄 삭제 작업(백오피스 스크립트)

## 21. 결정 사항 요약
- 스택: Next.js + Supabase Storage(파일 전용) + WS(MediaRecorder 청크) + Web Speech API + OpenAI
- 파일 우선, DB 미사용. 모든 상태/메타는 JSON/JSONL로 저장
- 모든 스토리지 통신은 Next.js 백엔드 경유(클라이언트 직접 접근 금지)
- 10자리 토큰은 URL/디렉토리/파일 접두에 동일 적용
- 네트워크 중단 시 자동 저장 후 state=interrupted

## 22. 리스크 및 완화
- Web Speech API 가용성: 폴백(Whisper 일괄 변환), 안내 문구
- WebM 컨테이너 정합성: FFmpeg remux 필수 단계
- 파일 동시성: 원자적 쓰기, 임시파일(.tmp) → rename
- 단일 노드 메모리 rate limit: 다중 인스턴스 시 L7 차단 장비 권장

## 23. 추적 가능한 수용 기준 매핑
- PRD A~K 항목 전부 본 TRD의 API/파일/플로우에 매핑
  - 타이머 정확도: Worker 기반 ±0.2s
  - 상태 레이블: not_started/landing/in_progress/interrupted/completed/error/locked
  - LLM 평가: 템플릿/버전 기록, PDF/JSON 저장
  - 파일 세트: 메타/상태/녹화/스크립트/로그 필수 존재 검증

끝.