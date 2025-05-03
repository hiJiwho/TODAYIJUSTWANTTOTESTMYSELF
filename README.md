```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>심리 테스트</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; }
    #question-area, #answer-area, #result-area { margin-bottom: 20px; }
    .answer-btn { padding: 10px 20px; margin-right: 10px; cursor: pointer; }
    #restart-btn { padding: 10px 20px; cursor: pointer; }
  </style>
</head>
<body>
  <h1>심리 테스트</h1>
  <!-- 질문이 표시될 영역 -->
  <div id="question-area"></div>
  <!-- 답변 버튼이 동적으로 생성될 영역 -->
  <div id="answer-area"></div>
  <!-- 최종 결과 메시지가 보여질 영역 -->
  <div id="result-area"></div>
  
  <script>
    // 전역 변수 선언
    let questions = [];    // 질문들을 저장할 배열 : { no: 번호, text: 질문내용 }
    let scoreRules = {};   // 각 질문의 척도에 따른 점수 규칙 저장: scoreRules[질문번호][답변척도] = 점수 변화량
    let resultRules = [];  // 총 점수에 따른 결과 규칙 배열: { min, max, message }

    let currentQuestionIndex = 0; // 현재 질문 인덱스
    let totalScore = 0;           // 누적 점수

    // DOM 요소 가져오기
    const questionArea = document.getElementById("question-area");
    const answerArea = document.getElementById("answer-area");
    const resultArea = document.getElementById("result-area");

    // 텍스트 파일을 fetch API로 불러오는 함수
    function fetchText(file) {
      return fetch(file)
        .then(response => {
          if (!response.ok) throw new Error(`파일 불러오기 실패: ${file}`);
          return response.text();
        });
    }
    
    // quiz.txt 파싱 함수 (형식: "Q번호: 질문 내용")
    function parseQuiz(text) {
      const lines = text.trim().split('\n');
      lines.forEach(line => {
        const trimmedLine = line.trim();
        const parts = trimmedLine.split(':');
        if (parts.length >= 2) {
          const questionIdRaw = parts[0].trim();
          const questionNo = parseInt(questionIdRaw.replace('Q', ''), 10);
          const questionText = parts.slice(1).join(':').trim();
          questions.push({ no: questionNo, text: questionText });
        }
      });
    }
    
    // score.txt 파싱 함수 (두 가지 유형: 점수 규칙과 결과 규칙)
    function parseScore(text) {
      const lines = text.trim().split('\n');
      lines.forEach(line => {
        const trimmedLine = line.trim();
        // 점수 규칙: "R[질문번호]-[척도]: [점수변화량]"
        if (trimmedLine.startsWith('R')) { 
          const colonIndex = trimmedLine.indexOf(':');
          if (colonIndex === -1) return;
          const ruleKey = trimmedLine.substring(1, colonIndex).trim(); // 예: "1-3"
          const scoreValueRaw = trimmedLine.substring(colonIndex + 1).trim(); // 예: "+3"
          const scoreValue = parseInt(scoreValueRaw, 10);
          const [qNoStr, scaleStr] = ruleKey.split('-');
          const qNo = parseInt(qNoStr, 10);
          const scale = parseInt(scaleStr, 10);
          if (!scoreRules[qNo]) {
            scoreRules[qNo] = {};
          }
          scoreRules[qNo][scale] = scoreValue;
        } 
        // 결과 규칙: "Result[최소점수]-[최대점수 or Max]: 결과 메시지"
        else if (trimmedLine.startsWith('Result')) {
          const colonIndex = trimmedLine.indexOf(':');
          if (colonIndex === -1) return;
          const rangePart = trimmedLine.substring(6, colonIndex).trim(); // "최소-최대"
          const message = trimmedLine.substring(colonIndex + 1).trim();
          const [minStr, maxStr] = rangePart.split('-');
          const min = parseInt(minStr, 10);
          let max;
          if (maxStr.toLowerCase() === 'max') {
            max = Infinity;
          } else {
            max = parseInt(maxStr, 10);
          }
          resultRules.push({ min, max, message });
        }
      });
    }
    
    // 현재 질문과 척도 버튼 생성 함수
    function displayQuestion() {
      // 기존 내용 초기화
      questionArea.innerHTML = "";
      answerArea.innerHTML = "";
      resultArea.innerHTML = "";
      
      if (currentQuestionIndex < questions.length) {
        const currentQuestion = questions[currentQuestionIndex];
        questionArea.textContent = currentQuestion.text;
  
        // 1~5 척도 버튼 생성
        for (let rating = 1; rating <= 5; rating++) {
          const btn = document.createElement('button');
          btn.textContent = rating;
          btn.className = "answer-btn";
          btn.addEventListener('click', () => handleAnswer(currentQuestion.no, rating));
          answerArea.appendChild(btn);
        }
      } else {
        // 모든 질문이 끝난 경우 결과 표시
        showResult();
      }
    }
    
    // 답변 처리 함수: 선택한 점수를 누적한 후 다음 질문으로 이동
    function handleAnswer(questionNo, chosenScale) {
      const rule = scoreRules[questionNo] ? scoreRules[questionNo][chosenScale] : 0;
      totalScore += rule;
      currentQuestionIndex++;
      displayQuestion();
    }
    
    // 총 점수에 따라 결과 메시지 표시 함수
    function showResult() {
      questionArea.innerHTML = "";
      answerArea.innerHTML = "";
      let finalMessage = "결과를 판단할 수 없습니다.";
      // 결과 규칙을 순회하여 해당 범위와 일치하는 메시지 결정
      for (let i = 0; i < resultRules.length; i++) {
        const rule = resultRules[i];
        if (totalScore >= rule.min && totalScore <= rule.max) {
          finalMessage = rule.message;
          break;
        }
      }
      resultArea.textContent = `당신의 총 점수: ${totalScore}. ${finalMessage}`;
      
      // "다시 시작하기" 버튼 생성
      const restartBtn = document.createElement('button');
      restartBtn.id = 'restart-btn';
      restartBtn.textContent = "다시 시작하기";
      restartBtn.addEventListener('click', restartQuiz);
      resultArea.appendChild(document.createElement('br'));
      resultArea.appendChild(restartBtn);
    }
    
    // 퀴즈를 다시 초기화하는 함수
    function restartQuiz() {
      currentQuestionIndex = 0;
      totalScore = 0;
      displayQuestion();
    }
    
    // 초기화 함수: quiz.txt와 score.txt 파일을 불러와 파싱 후 퀴즈 시작
    function initQuiz() {
      Promise.all([fetchText('quiz.txt'), fetchText('score.txt')])
        .then(([quizData, scoreData]) => {
          parseQuiz(quizData);
          parseScore(scoreData);
          displayQuestion();
        })
        .catch(err => {
          questionArea.textContent = "데이터 로딩중 에러 발생: " + err.message;
        });
    }
    
    // 페이지 로딩 후 퀴즈 시작
    window.addEventListener('load', initQuiz);
  </script>
</body>
</html>
```

```txt
Q1: 새로운 도전은 나에게 즐거움이다.
Q2: 계획 없이 떠나는 여행을 선호한다.
Q3: 사람들과 함께 시간을 보내는 것을 좋아한다.
```

```txt
R1-1: +1
R1-2: +2
R1-3: +3
R1-4: +4
R1-5: +5

R2-1: +5
R2-2: +4
R2-3: +3
R2-4: +2
R2-5: +1

R3-1: +1
R3-2: +2
R3-3: +3
R3-4: +4
R3-5: +5

Result3-9: 당신은 안정적인 성향입니다.
Result10-15: 당신은 유연한 성향입니다.
Result16-Max: 당신은 활동적인 성향입니다.
```

─────────────────────────────  
설명:

1. HTML 뼈대에는 질문, 답변, 결과 표시를 위한 세 개의 영역(div)이 있습니다. 각각 id가 "question-area", "answer-area", "result-area"로 지정되었습니다.
2. 자바스크립트는 페이지 하단의 <script> 태그 내부에 작성되었으며, fetch API를 사용해 외부 파일 quiz.txt와 score.txt를 불러옵니다.
3. parseQuiz 함수는 quiz.txt의 각 줄("Q번호: 질문 내용")을 읽어 파싱한 후 questions 배열에 추가합니다.
4. parseScore 함수는 score.txt의 각 줄을 읽어 점수 규칙과 결과 규칙을 분리하여 각각 scoreRules 객체와 resultRules 배열에 저장합니다.
5. displayQuestion 함수는 현재 질문을 화면에 표시하고 1부터 5까지의 답변 버튼을 동적으로 생성합니다.
6. 사용자가 버튼을 클릭하면 handleAnswer 함수가 호출되어 해당 답변에 해당하는 점수를 scoreRules에서 찾아 누적 점수를 업데이트하고, 다음 질문을 불러옵니다.
7. 모든 질문에 답하면 showResult 함수가 실행되어 totalScore 값을 기준으로 resultRules 내의 범위와 일치하는 결과 메시지를 출력합니다. 또한 "다시 시작하기" 버튼을 제공하여 퀴즈를 새로 시작할 수 있습니다.
8. initQuiz 함수는 페이지 로딩 시 실행되어 파일들을 불러오고 파싱한 후 첫 질문을 표시합니다.

이 코드는 Github Pages에서 작동하는 간단한 심리 테스트 웹사이트를 구현하며, quiz.txt와 score.txt 파일을 통해 동적으로 테스트 내용과 결과를 설정합니다.
