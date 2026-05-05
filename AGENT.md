# 昭和スタジオ — 에이전트 기능 추가 지시문

> CLAUDE.md와 함께 읽으세요.
> 이 파일은 Claude API를 활용한 AI 에이전트 기능을 index.html에 추가하는 지시문입니다.

---

## 에이전트 아키텍처 개요

```
사용자 입력
    ↓
[에이전트 라우터] — 어떤 에이전트를 실행할지 판단
    ↓
┌──────────────────────────────────────────────────┐
│  에이전트 1: 대본 작가     (ScriptAgent)          │
│  에이전트 2: 프롬프트 생성 (PromptAgent)          │
│  에이전트 3: 품질 검수관   (ReviewAgent)          │
│  에이전트 4: 스토리 코치   (StoryAgent)           │
│  에이전트 5: 전략 분석가   (StrategyAgent)        │
└──────────────────────────────────────────────────┘
    ↓
결과 → UI에 스트리밍 표시
```

---

## API 연결 기본 설정

```javascript
// App.agent — 에이전트 코어 모듈
App.agent = {
  
  // 기본 API 호출 함수
  async call(systemPrompt, userMessage, onChunk) {
    const response = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 1000,
        stream: true,
        system: systemPrompt,
        messages: [{ role: 'user', content: userMessage }]
      })
    });

    // 스트리밍 처리
    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let fullText = '';

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      const chunk = decoder.decode(value);
      const lines = chunk.split('\n').filter(l => l.startsWith('data: '));
      for (const line of lines) {
        try {
          const data = JSON.parse(line.slice(6));
          if (data.type === 'content_block_delta' && data.delta?.text) {
            fullText += data.delta.text;
            if (onChunk) onChunk(data.delta.text, fullText);
          }
        } catch {}
      }
    }
    return fullText;
  },

  // 스트리밍 없는 단순 호출
  async callSimple(systemPrompt, userMessage) {
    const response = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 1000,
        system: systemPrompt,
        messages: [{ role: 'user', content: userMessage }]
      })
    });
    const data = await response.json();
    return data.content?.[0]?.text || '';
  }
};
```

---

## 에이전트 1: 대본 작가 (ScriptAgent)

### 기능
사용자가 제목과 키워드만 입력하면 완성된 일본어 대본을 스트리밍으로 생성

### UI 위치
탭 2(대본) — 기존 폼 아래에 "🤖 AI 대본 생성" 버튼 추가

### 버튼 디자인
```html
<button class="btn-agent" onclick="App.agent.Script.run()">
  🤖 AI가 대본 작성해줘
  <span class="agent-badge">Agent</span>
</button>
```

```css
.btn-agent {
  background: linear-gradient(135deg, #d4a843, #a07820);
  color: #fff;
  border: none;
  border-radius: 12px;
  padding: 14px 20px;
  font-size: 15px;
  font-weight: 700;
  width: 100%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}
.agent-badge {
  background: rgba(255,255,255,0.25);
  padding: 2px 8px;
  border-radius: 20px;
  font-size: 11px;
}
```

### 스트리밍 출력 박스
```html
<div id="agentScriptOutput" class="agent-output" style="display:none">
  <div class="agent-output-header">
    <span class="agent-icon spinning">⚙️</span>
    <span id="agentScriptStatus">대본 작성 중...</span>
    <button onclick="App.agent.Script.stop()">⏹ 중지</button>
  </div>
  <div id="agentScriptText" class="agent-stream-text"></div>
  <div class="agent-actions" id="agentScriptActions" style="display:none">
    <button onclick="App.agent.Script.accept()">✅ 이 대본 사용</button>
    <button onclick="App.agent.Script.regenerate()">🔄 다시 생성</button>
    <button onclick="App.copyText('agentScriptText')">📋 복사</button>
  </div>
</div>
```

### 에이전트 로직

```javascript
App.agent.Script = {

  systemPrompt: `당신은 일본 시니어(50~70대) 대상 YouTube 채널의 전문 대본 작가입니다.
  
전문 분야:
- 쇼와 시대(昭和) 향수 콘텐츠
- 일본어 네이티브 수준 집필
- 시니어 친화적 톤 (따뜻하고 그리운 감성)

대본 작성 규칙:
1. 오프닝 훅: 첫 15초 안에 시청자의 기억을 건드리는 문장
2. 구조: 훅 → 배경설명 → 핵심내용 → 현재비교 → AI고지 → CTA
3. AI 재현 고지는 반드시 포함: 「この映像はAIで生成した昭和の再現です」
4. 댓글 유도 질문은 구체적으로: 장소명·과자명 등 구체적 추억 질문
5. 과장·조롱·단정적 결론 절대 금지
6. 의료·금융·연금 관련 단정 표현 금지
7. 각 섹션 앞에 【섹션명】 표시

출력 형식: 일본어 대본만 출력. 한국어 설명 없이.`,

  async run() {
    const title = document.getElementById('scriptTitle')?.value?.trim();
    const duration = App.state.scriptSettings?.duration || 600;
    const axis = document.getElementById('contentAxis')?.value || '쇼와향수';
    const keywords = document.getElementById('scriptKeywords')?.value?.trim();

    if (!title) { App.toast('⚠️ 영상 제목을 먼저 입력하세요', 'warn'); return; }

    const minutes = Math.floor(duration / 60);
    const axisMap = {
      '쇼와향수': '昭和の懐かし文化',
      '시니어생활': 'シニアの生活と健康',
      '한국인의시선': '韓国人の視点で見る日本文化',
      '지역관광': '地方の観光と歴史',
      '문학역사': '文学と歴史の再現'
    };

    const userMessage = `以下の条件で日本語の動画スクリプトを書いてください：

タイトル: 「${title}」
コンテンツ軸: ${axisMap[axis] || axis}
動画の長さ: 約${minutes}分
キーワード: ${keywords || 'なし'}
ターゲット: ${(App.state.targetAge || ['60代']).join('・')}の日本人視聴者

スクリプトを書いてください。`;

    const outputEl = document.getElementById('agentScriptOutput');
    const textEl = document.getElementById('agentScriptText');
    const statusEl = document.getElementById('agentScriptStatus');
    const actionsEl = document.getElementById('agentScriptActions');

    outputEl.style.display = 'block';
    textEl.innerHTML = '';
    actionsEl.style.display = 'none';
    statusEl.textContent = '✍️ 대본 작성 중...';

    this._aborted = false;

    try {
      await App.agent.call(
        this.systemPrompt,
        userMessage,
        (chunk, fullText) => {
          if (this._aborted) return;
          textEl.innerHTML = fullText.replace(/\n/g, '<br>');
          textEl.scrollTop = textEl.scrollHeight;
        }
      );
      statusEl.textContent = '✅ 대본 완성!';
      document.querySelector('.agent-icon').classList.remove('spinning');
      actionsEl.style.display = 'flex';
    } catch (err) {
      statusEl.textContent = '❌ 오류 발생. 다시 시도해주세요.';
      console.error(err);
    }
  },

  stop() { this._aborted = true; },

  accept() {
    const text = document.getElementById('agentScriptText').innerText;
    App.state.lastScript = text;
    App.save();
    App.toast('✅ 대본이 저장됐습니다!');
  },

  regenerate() { this.run(); }
};
```

---

## 에이전트 2: 프롬프트 생성 (PromptAgent)

### 기능
대본 내용을 분석해서 필요한 Morphic 프롬프트를 자동으로 생성

### UI 위치
탭 1(프롬프트) 하단 OR 탭 2(대본) 대본 완성 후 버튼

```javascript
App.agent.Prompt = {

  systemPrompt: `당신은 Morphic AI 영상 생성 전문가입니다.

일본 쇼와 시대 YouTube 콘텐츠를 위한 Morphic 프롬프트를 생성합니다.

프롬프트 작성 규칙:
1. 언어: 일본어로 작성
2. 필수 요소: 시대명 + 장소 + 등장인물(선택) + 시간대/빛 + 카메라워크 + 분위기 + 클립길이
3. 시대는 반드시 명시: 昭和XX年代
4. 금지 요소 명시: スマートフォン禁止、現代の看板禁止 등
5. 클립 길이: 5〜8秒 기본
6. 분위기: 16mmフィルム風 or セピア調 등 레트로 감성 필수

출력 형식:
각 장면을 번호로 구분해서 출력
① [장면명]: [프롬프트]
② [장면명]: [프롬프트]`,

  async runFromScript() {
    const scriptText = document.getElementById('agentScriptText')?.innerText
      || App.state.lastScript || '';

    if (!scriptText || scriptText.length < 50) {
      App.toast('⚠️ 먼저 대본을 생성하세요', 'warn'); return;
    }

    const userMessage = `以下の動画スクリプトを分析して、
Morphic AIで生成すべき映像シーン${5}個のプロンプトを作成してください。

スクリプト:
${scriptText.slice(0, 1500)}

各シーンに最適なMorphicプロンプトを日本語で書いてください。`;

    const container = document.getElementById('agentPromptOutput');
    container.style.display = 'block';
    container.innerHTML = '<div class="agent-loading">🔮 장면 분석 중...</div>';

    const result = await App.agent.callSimple(this.systemPrompt, userMessage);
    
    // 파싱 후 카드형으로 표시
    const scenes = result.split(/\n(?=①|②|③|④|⑤|⑥|\d+\.)/).filter(Boolean);
    container.innerHTML = scenes.map((scene, i) => `
      <div class="prompt-card">
        <div class="prompt-card-text">${scene.replace(/\n/g, '<br>')}</div>
        <button class="btn btn-sm" onclick="navigator.clipboard.writeText(\`${scene.replace(/`/g,"'")}\`).then(()=>App.toast('복사됨!'))">
          📋 복사
        </button>
      </div>
    `).join('');
  },

  // 설정값으로 직접 생성
  async runFromSettings() {
    const scenes = App.state.selectedScenes || ['商店街'];
    const era = App.state.promptSettings?.era || 1965;
    const eraStr = `昭和${era - 1925}年代`;

    const userMessage = `以下の設定でMorphicプロンプトを3つのバリエーションで作成してください：
シーン: ${scenes.join('、')}
時代: ${eraStr}
3パターン（通常・夕暮れ・雨の日）`;

    const result = await App.agent.callSimple(this.systemPrompt, userMessage);
    document.getElementById('promptText').innerHTML =
      `<button class="copy-btn" onclick="App.copyText('promptText',this)">복사</button>${result}`;
    document.getElementById('promptOutput').style.display = 'block';
    App.toast('🔮 AI가 프롬프트를 생성했습니다!');
  }
};
```

---

## 에이전트 3: 품질 검수관 (ReviewAgent)

### 기능
대본이나 프롬프트를 입력받아 AI가 구체적인 피드백과 개선안을 제시

### UI 위치
탭 4(평가) 내 "🤖 AI 검수" 버튼

```javascript
App.agent.Review = {

  systemPrompt: `당신은 일본 YouTube 콘텐츠 품질 검수 전문가입니다.

검수 기준:
1. 시니어 친화성 (50~70대 일본인 대상)
2. 쇼와 시대 고증 정확성
3. 일본어 자연스러움
4. AI 재현 고지 포함 여부
5. 저작권 위험 요소
6. 민감 표현 (의료·금융·역사 왜곡)
7. 훅 강도 (첫 15초)
8. CTA 효과성

반드시 다음 형식으로 출력하세요:

【종합 점수】XX점 / 100점
【등급】S/A/B/C/D

【항목별 평가】
✅ 강점: ...
⚠️ 개선 필요: ...
❌ 문제점: ...

【구체적 수정 제안】
1. ...
2. ...
3. ...

【수정된 훅 예시】(훅이 약한 경우에만)
...

출력 언어: 한국어 (일본어 인용문은 일본어 유지)`,

  async runScriptReview() {
    const scriptText = document.getElementById('reviewInput')?.value?.trim()
      || App.state.lastScript || '';

    if (!scriptText) { App.toast('⚠️ 검수할 대본을 입력하세요', 'warn'); return; }

    this._showLoading('reviewOutput', '📋 대본 분석 중...');

    const result = await App.agent.callSimple(
      this.systemPrompt,
      `다음 대본을 검수해주세요:\n\n${scriptText}`
    );

    this._showResult('reviewOutput', result);
    App.state.lastReview = result;
    App.save();
  },

  async runPromptReview() {
    const promptText = document.getElementById('promptReviewInput')?.value?.trim()
      || App.state.lastPrompt || '';

    if (!promptText) { App.toast('⚠️ 검수할 프롬프트를 입력하세요', 'warn'); return; }

    const promptSystem = `Morphic AI 프롬프트 품질 검수 전문가입니다.
체크 항목: 시대명 명시/장소 구체성/인물 묘사/조명 조건/카메라워크/금지요소/클립길이
점수와 구체적 개선안을 한국어로 출력하세요.`;

    this._showLoading('promptReviewOutput', '🔮 프롬프트 분석 중...');
    const result = await App.agent.callSimple(promptSystem, `검수:\n${promptText}`);
    this._showResult('promptReviewOutput', result);
  },

  _showLoading(elId, msg) {
    const el = document.getElementById(elId);
    if (!el) return;
    el.style.display = 'block';
    el.innerHTML = `<div class="agent-loading"><span class="spinning">⚙️</span> ${msg}</div>`;
  },

  _showResult(elId, result) {
    const el = document.getElementById(elId);
    if (!el) return;
    // 점수 추출해서 색상 표시
    const scoreMatch = result.match(/(\d+)점\s*\/\s*100점/);
    const score = scoreMatch ? parseInt(scoreMatch[1]) : null;
    const scoreColor = score >= 90 ? '#4caf7d' : score >= 80 ? '#d4a843' :
                       score >= 70 ? '#e09a4a' : '#e05252';

    el.innerHTML = `
      ${score ? `<div style="text-align:center;padding:12px 0;font-size:28px;font-weight:700;color:${scoreColor};">${score}점</div>` : ''}
      <div class="agent-result-text">${result.replace(/\n/g, '<br>')}</div>
      <div style="display:flex;gap:8px;margin-top:12px;">
        <button class="btn btn-sm btn-secondary" onclick="navigator.clipboard.writeText(\`${result.replace(/`/g,"'")}\`).then(()=>App.toast('복사됨!'))">📋 복사</button>
      </div>
    `;
  }
};
```

### 검수 탭 UI

```
탭 4 평가 화면:

┌─────────────────────────────────────┐
│ ⭐ AI 품질 검수                      │
│                                     │
│ ◉ 대본 검수  ○ 프롬프트 검수          │
│                                     │
│ ┌─────────────────────────────────┐ │
│ │ 검수할 내용을 붙여넣으세요...    │ │
│ │ (또는 최근 생성된 대본 자동 로드)│ │
│ └─────────────────────────────────┘ │
│                                     │
│ [최근 대본 불러오기]  [직접 입력]    │
│                                     │
│ [🤖  AI 검수 시작]                  │
└─────────────────────────────────────┘

결과 출력:
┌─────────────────────────────────────┐
│            83점                     │
│  ✅ 강점 3가지                      │
│  ⚠️ 개선 필요 2가지                 │
│  💡 수정 제안                       │
└─────────────────────────────────────┘
```

---

## 에이전트 4: 스토리 코치 (StoryAgent)

### 기능
주제 입력 시 스토리 구조와 감정 흐름을 AI가 제안

```javascript
App.agent.Story = {

  systemPrompt: `당신은 일본 쇼와 시대 향수 콘텐츠 스토리텔링 전문가입니다.

스토리 설계 원칙:
1. 시청자의 기억을 건드리는 구체적 디테일 사용
2. 감정 곡선: 훅(호기심) → 본문(그리움) → 반전(현실) → 클로징(따뜻함)
3. 시니어가 댓글 달고 싶어지는 구체적 요소 포함
4. 한국인 크리에이터의 외부자 시각을 강점으로 활용

스토리 구조 제안 시 출력 형식:
【추천 구조】상실형/발견형/편지형/비교형
【감정 곡선】0분:__% → 3분:__% → (시간대별 감정 강도)
【핵심 장면 3개】구체적 시각 묘사
【예상 반응 댓글】시청자가 달 것 같은 댓글 예시 3개
【위험 요소】피해야 할 표현이나 방향`,

  async suggest(topic, duration) {
    const el = document.getElementById('storyAgentOutput');
    el.style.display = 'block';
    el.innerHTML = '<div class="agent-loading"><span class="spinning">⚙️</span> 스토리 구조 분석 중...</div>';

    const result = await App.agent.callSimple(
      this.systemPrompt,
      `주제: 「${topic}」\n영상 길이: ${Math.floor(duration/60)}분\n이 주제에 최적인 스토리 구조를 제안해주세요.`
    );

    el.innerHTML = `
      <div class="agent-result-text">${result.replace(/\n/g,'<br>')}</div>
      <div style="display:flex;gap:8px;margin-top:12px;">
        <button class="btn btn-primary btn-sm" onclick="App.agent.Story.applyToScript()">
          📝 이 구조로 대본 작성
        </button>
        <button class="btn btn-secondary btn-sm" onclick="App.agent.Story.regenerate('${topic}', ${duration})">
          🔄 다시 제안
        </button>
      </div>
    `;
    App.state.lastStoryAdvice = result;
    App.save();
  },

  regenerate(topic, duration) { this.suggest(topic, duration); },

  applyToScript() {
    // 스토리 제안을 대본 탭으로 전달
    App.switchTab('script');
    App.toast('💡 대본 탭에서 AI 대본 생성을 눌러주세요');
  }
};
```

---

## 에이전트 5: 전략 분석가 (StrategyAgent)

### 기능
업로드 성과 데이터를 입력하면 다음 영상 전략을 AI가 제안

### UI 위치
탭 5(관리) > 주간 KPI 섹션 하단

```javascript
App.agent.Strategy = {

  systemPrompt: `당신은 일본 YouTube 시니어 채널 전문 전략가입니다.

분석 기준:
- 일본 쇼와 향수 채널의 알고리즘 패턴
- 시니어(50~70대) 시청 행동 패턴
- CTR 기준: 4% 미만=나쁨, 4~7%=보통, 7%+=좋음
- 시청지속률 기준: 40% 미만=나쁨, 40~60%=보통, 60%+=좋음

출력 형식:
【성과 진단】잘 된 점 / 아쉬운 점
【원인 분석】데이터 기반 원인
【다음 영상 전략】구체적 제목 후보 3개 포함
【즉시 실행 액션】이번 주 할 일 3가지
출력 언어: 한국어`,

  async analyze() {
    const history = App.state.kpiHistory || [];
    if (history.length < 1) {
      App.toast('⚠️ KPI 기록이 없습니다. 먼저 성과를 입력해주세요', 'warn');
      return;
    }

    const latest = history[0];
    const userMessage = `최근 성과 데이터:
구독자 증가: +${latest.subs}명
조회수: ${latest.views}회
시청시간: ${latest.hours}h
수익: ¥${latest.revenue}

이 채널은 일본 50~70대 대상 쇼와 향수 콘텐츠 채널입니다.
분석과 전략을 제안해주세요.`;

    const el = document.getElementById('strategyOutput');
    el.style.display = 'block';
    el.innerHTML = '<div class="agent-loading"><span class="spinning">⚙️</span> 전략 분석 중...</div>';

    const result = await App.agent.callSimple(this.systemPrompt, userMessage);
    el.innerHTML = `
      <div class="agent-result-text">${result.replace(/\n/g,'<br>')}</div>
    `;
    App.state.lastStrategy = result;
    App.save();
  }
};
```

---

## 에이전트 공통 UI 스타일

```css
/* 에이전트 관련 CSS — index.html의 <style> 안에 추가 */

.agent-output {
  background: var(--surface);
  border: 1px solid var(--accent);
  border-radius: var(--radius);
  padding: 16px;
  margin-top: 12px;
}

.agent-output-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 12px;
  font-size: 14px;
  font-weight: 600;
  color: var(--accent);
}

.agent-loading {
  text-align: center;
  padding: 24px;
  color: var(--text2);
  font-size: 14px;
}

.agent-stream-text {
  background: var(--surface2);
  border-radius: var(--radius-sm);
  padding: 14px;
  font-size: 14px;
  line-height: 1.9;
  min-height: 80px;
  max-height: 400px;
  overflow-y: auto;
  white-space: pre-wrap;
}

.agent-result-text {
  font-size: 14px;
  line-height: 1.9;
  color: var(--text);
}

.agent-actions {
  display: flex;
  gap: 8px;
  margin-top: 12px;
  flex-wrap: wrap;
}

/* 스피너 애니메이션 */
@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}
.spinning { display: inline-block; animation: spin 1s linear infinite; }

/* AI 뱃지 */
.ai-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  background: var(--accent-dim);
  color: var(--accent);
  border: 1px solid var(--accent);
  border-radius: 20px;
  padding: 3px 10px;
  font-size: 11px;
  font-weight: 600;
}

/* 에이전트 구분선 */
.agent-divider {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 16px 0;
  color: var(--text3);
  font-size: 12px;
}
.agent-divider::before,
.agent-divider::after {
  content: '';
  flex: 1;
  height: 1px;
  background: var(--border);
}
```

---

## 에이전트 통합 초기화

```javascript
// App.init() 안에 추가
App.init = function() {
  // ... 기존 코드 ...
  
  // 에이전트 초기화 (API 연결 확인)
  this.agent.checkConnection();
};

App.agent.checkConnection = async function() {
  // API 연결 상태 표시
  const indicator = document.getElementById('agentStatus');
  if (!indicator) return;
  
  try {
    // 간단한 ping 테스트
    const res = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 5,
        messages: [{ role: 'user', content: 'hi' }]
      })
    });
    if (res.ok || res.status === 200) {
      indicator.innerHTML = '🟢 AI 연결됨';
      indicator.style.color = 'var(--green)';
    }
  } catch {
    indicator.innerHTML = '🔴 AI 오프라인';
    indicator.style.color = 'var(--red)';
  }
};
```

---

## 헤더에 에이전트 상태 표시 추가

```html
<header class="app-header">
  <span class="app-logo">🎬</span>
  <h1 class="app-title">昭和スタジオ</h1>
  <div class="header-right">
    <span id="agentStatus" style="font-size:11px;">⚙️ AI 초기화 중</span>
    <span id="saveStatus" style="font-size:11px; color:var(--green);">● 저장됨</span>
  </div>
</header>
```

---

## 구현 우선순위

CLAUDE.md의 전체 재설계와 함께 다음 순서로 구현하세요:

1. **필수 (반드시 구현)**
   - `App.agent.call()` — 스트리밍 API 호출 함수
   - `App.agent.callSimple()` — 단순 API 호출
   - `App.agent.Script.run()` — 대본 자동 생성
   - `App.agent.Review.runScriptReview()` — 대본 검수

2. **중요 (구현 권장)**
   - `App.agent.Prompt.runFromScript()` — 대본→프롬프트 자동 변환
   - `App.agent.Story.suggest()` — 스토리 구조 제안

3. **선택 (여유 있으면)**
   - `App.agent.Strategy.analyze()` — 성과 분석
   - 에이전트 연결 상태 표시

---

## Claude Code 실행 명령

```
CLAUDE.md와 AGENT.md를 모두 읽고,
index.html을 완전히 새로 작성해줘.

핵심 요구사항:
1. CLAUDE.md의 UI/UX 전면 개선 적용
2. AGENT.md의 5개 에이전트 모두 구현
3. 단일 index.html 파일로 완성
4. 외부 라이브러리 없이 순수 Vanilla JS
5. 모바일(375px)에서 완벽 동작
```
