---
layout: page
title: Linux+ Quiz
permalink: /quiz/
date: 2025-10-03
tags: [quiz, linux-plus]
---

<div id="quiz-app" class="quiz-container">
  <h1>Linux+ Self-Test</h1>
  <p>Welcome! Choose a topic to start quizzing. More coming soon.</p>

  <!-- Topic Tabs -->
  <div class="topic-tabs">
    <button class="tab-btn active" data-topic="linux-plus" onclick="switchTopic('linux-plus')">Linux+</button>
    <button class="tab-btn" data-topic="ccna" onclick="switchTopic('ccna')" disabled>CCNA (Coming Soon)</button>
  </div>

  <div id="linux-plus-tab" class="tab-content active">
    <h2>Linux+ Quiz</h2>
    <p>Multiple-Choice: Pick number of questions (30–100 random from full set).</p>
    <label for="mcq-count">Questions: </label>
    <select id="mcq-count">
      <option value="30">30</option>
      <option value="50">50</option>
      <option value="100">100 (Full)</option>
    </select>
    <button onclick="startMCQ()">Start MCQ Quiz</button>

    <hr>
    <p>Performance-Based: Type your answers and get feedback instantly (full 50 questions).</p>
    <button onclick="startPerformance()">Start Performance Quiz</button>

  </div>

  <div id="ccna-tab" class="tab-content">
    <p>CCNA quiz coming soon! Meanwhile, check my <a href="/about/">goals</a> page for the roadmap.</p>
  </div>

  <!-- Quiz Area -->
  <div id="quiz-area" style="display:none;"></div>
  <div id="results" style="display:none;"></div>
</div>

<style>
/* Scoped Quiz Styles */
:root {
  --btn-bg: var(--theme-accent);
  --btn-hover: color-mix(in srgb, var(--theme-accent) 70%, black);
  --correct: #20c997;
  --incorrect: #e83e8c;
  --bg-light: rgba(255, 255, 255, 0.05);
  --border-light: var(--theme-accent);
  --text-muted: rgba(255, 255, 255, 0.7);
}
.quiz-container {
  max-width: 800px;
  margin: 20px auto;
  padding: 20px;
  background: rgba(0, 0, 0, 0.4);
  border: 1px solid var(--theme-border);
  border-radius: 6px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.5);
  color: #eee;
  font-family: "Segoe UI", sans-serif;
}
.quiz-container .topic-tabs {
  display: flex;
  margin: 20px 0;
  border-bottom: 1px solid var(--theme-border);
}
.quiz-container .tab-btn {
  padding: 10px 20px;
  background: transparent;
  border: 1px solid var(--theme-border);
  border-bottom: none;
  color: #ddd;
  cursor: pointer;
  transition: background 0.3s ease, color 0.3s ease;
}
.quiz-container .tab-btn.active {
  background: var(--theme-accent);
  color: #111;
  font-weight: bold;
}
.quiz-container .tab-btn:hover:not(:disabled) {
  background: rgba(255, 255, 255, 0.1);
}
.quiz-container .tab-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}
.quiz-container .tab-content {
  padding: 20px;
  border: 1px solid var(--theme-border);
  border-top: none;
}
.quiz-container .tab-content:not(.active) { display: none; }
.quiz-container .question {
  margin-bottom: 20px;
  padding: 15px;
  border-left: 4px solid var(--border-light);
  background: rgba(255, 255, 255, 0.05);
  border-radius: 0 4px 4px 0;
  transition: background 0.3s ease, border-color 0.3s ease;
}
.quiz-container .question.correct {
  border-left-color: var(--correct);
  background: rgba(32, 201, 151, 0.2);
  color: #d4ffd4;
}
.quiz-container .question.incorrect {
  border-left-color: var(--incorrect);
  background: rgba(232, 62, 140, 0.2);
  color: #ffd6e5;
}
.quiz-container .options { list-style: none; padding: 0; }
.quiz-container .options li { margin: 10px 0; }
.quiz-container textarea,
.quiz-container select {
  background: rgba(0,0,0,0.5);
  border: 1px solid var(--theme-border);
  border-radius: 4px;
  color: #eee;
  padding: 8px;
}
.quiz-container textarea {
  width: 100%;
  resize: vertical;
}
.quiz-container button {
  background: var(--btn-bg);
  color: #111;
  padding: 10px 15px;
  border: none;
  border-radius: 3px;
  cursor: pointer;
  margin: 5px;
  font-weight: bold;
  transition: background 0.3s ease, transform 0.2s ease;
}
.quiz-container button:hover {
  background: var(--btn-hover);
  transform: translateY(-2px);
}
.quiz-container .explanation {
  display: none;
  font-style: italic;
  margin-top: 10px;
  color: var(--text-muted);
  padding: 10px;
  background: rgba(255, 255, 255, 0.08);
  border-radius: 3px;
}
.quiz-container #progress {
  text-align: center;
  font-weight: bold;
  color: var(--theme-accent);
  margin: 10px 0;
  letter-spacing: 1px;
}
@media (max-width: 600px) {
  .quiz-container .topic-tabs { flex-direction: column; }
}
</style>

<script>
// Core Quiz Logic

const topics = {
  'linux-plus': { mcq: [], performance: [] },
  'ccna': { mcq: [], performance: [] }
};

let currentQuiz = { type: '', questions: [], current: 0, userAnswers: [], score: 0 };

// Load external question files
async function loadQuizData() {
  try {
    const [mcqRes, perfRes] = await Promise.all([
      fetch('quizzes/mcq.json'),  // Fixed relative path
      fetch('quizzes/performance.json')  // Fixed relative path
    ]);

    const mcqData = await mcqRes.json();
    const perfData = await perfRes.json();

    topics['linux-plus'].mcq = mcqData;
    topics['linux-plus'].performance = perfData;

    console.log('✅ Quiz data loaded successfully');
  } catch (err) {
    console.error('❌ Error loading quiz data:', err);
    // Optional: Show user-friendly alert if data fails to load
    alert('Failed to load quiz questions. Check console for details.');
  }
}

// Initialize after DOM load
document.addEventListener('DOMContentLoaded', async () => {
  await loadQuizData();
  updateThemeStyles();
});

// Theme tracking
let observer = new MutationObserver(() => updateThemeStyles());
observer.observe(document.body, { attributes: true, attributeFilter: ['class'] });
function updateThemeStyles() {
  const bodyClass = document.body.className;
  const theme = bodyClass.includes('skin-sunset')
    ? 'sunset'
    : bodyClass.includes('skin-bronze')
    ? 'bronze'
    : bodyClass.includes('skin-steel')
    ? 'steel'
    : bodyClass.includes('skin-crimson')
    ? 'crimson'
    : 'default';
}

function switchTopic(topic) {
  document.querySelectorAll('.tab-btn').forEach(btn => btn.classList.remove('active'));
  event.target.classList.add('active');
  document.querySelectorAll('.tab-content').forEach(c => c.classList.remove('active'));
  document.getElementById(topic + '-tab').classList.add('active');
}

function startMCQ() {
  const allQ = topics['linux-plus'].mcq;
  const count = Math.min(parseInt(document.getElementById('mcq-count').value), allQ.length);
  currentQuiz = { type: 'mcq', questions: shuffle([...allQ]).slice(0, count), current: 0, userAnswers: new Array(count).fill(-1), score: 0 };
  showQuestion();
  document.getElementById('quiz-area').style.display = 'block';
  document.getElementById('results').style.display = 'none';
}

function startPerformance() {
  const allQ = topics['linux-plus'].performance;
  currentQuiz = { type: 'performance', questions: shuffle([...allQ]), current: 0, userAnswers: [], score: 0 };
  showQuestion();
  document.getElementById('quiz-area').style.display = 'block';
  document.getElementById('results').style.display = 'none';
}

function showQuestion() {
  const q = currentQuiz.questions[currentQuiz.current];
  let html = `<div class="question" id="question-${currentQuiz.current}">
                <h3>Q${currentQuiz.current + 1}: ${q.question}</h3>`;

  if (currentQuiz.type === 'mcq') {
    html += `<ul class="options">
      ${q.options.map((opt, i) => `
        <li><input type="radio" name="q${currentQuiz.current}" value="${i}"> ${opt}</li>
      `).join('')}
    </ul>`;
  } else {
    const userAns = currentQuiz.userAnswers[currentQuiz.current] || '';
    html += `<textarea rows="4" placeholder="Enter your answer...">${userAns}</textarea>`;
  }

  html += `
    <div id="feedback" class="explanation"></div>
    <div id="progress">Question ${currentQuiz.current + 1} / ${currentQuiz.questions.length}</div>
    ${currentQuiz.current > 0 ? '<button onclick="prevQuestion()">Previous</button>' : ''}
    <button onclick="checkAnswer()">Check Answer</button>
    <button id="nextBtn" onclick="nextQuestion()" disabled>${currentQuiz.current === currentQuiz.questions.length - 1 ? 'Finish' : 'Next'}</button>
  `;
  document.getElementById('quiz-area').innerHTML = html;
}

function checkAnswer() {
  saveAnswer();
  const q = currentQuiz.questions[currentQuiz.current];
  const userAns = currentQuiz.userAnswers[currentQuiz.current];
  const feedback = document.getElementById('feedback');
  const questionDiv = document.getElementById(`question-${currentQuiz.current}`);

  let correct = false;
  if (currentQuiz.type === 'mcq') {
    correct = userAns === q.answer;
  } else {
    correct = q.expected.some(term => userAns.includes(term.toLowerCase()));
  }

  if (correct) {
    questionDiv.classList.add('correct');
    feedback.innerHTML = `✅ <strong>Correct!</strong><br>${q.explanation}`;
    currentQuiz.score++;
  } else {
    questionDiv.classList.add('incorrect');
    feedback.innerHTML = `❌ <strong>Incorrect.</strong><br>Correct Answer: ${
      currentQuiz.type === 'mcq' ? q.options[q.answer] : q.expected.join(' / ')
    }<br>${q.explanation}`;
  }

  feedback.style.display = 'block';
  document.querySelectorAll('input, textarea').forEach(el => (el.disabled = true));
  document.getElementById('nextBtn').disabled = false;
}

function nextQuestion() {
  saveAnswer();
  if (currentQuiz.current < currentQuiz.questions.length - 1) {
    currentQuiz.current++;
    showQuestion();
  } else showResults();
}

function prevQuestion() {
  saveAnswer();
  currentQuiz.current--;
  showQuestion();
}

function saveAnswer() {
  if (currentQuiz.type === 'mcq') {
    const selected = document.querySelector(`input[name="q${currentQuiz.current}"]:checked`);
    currentQuiz.userAnswers[currentQuiz.current] = selected ? parseInt(selected.value) : -1;
  } else {
    currentQuiz.userAnswers[currentQuiz.current] = document.querySelector('textarea').value.toLowerCase().trim();
  }
}

function showResults() {
  let html = `<h2>Results: ${currentQuiz.score} / ${currentQuiz.questions.length} (${Math.round(currentQuiz.score / currentQuiz.questions.length * 100)}%)</h2>`;
  currentQuiz.questions.forEach((q, i) => {
    const userAns = currentQuiz.userAnswers[i];
    const status = (currentQuiz.type === 'mcq'
      ? userAns === q.answer
      : q.expected.some(term => userAns.includes(term.toLowerCase())))
      ? 'correct'
      : 'incorrect';
    const userDisplay = currentQuiz.type === 'mcq'
      ? (userAns >= 0 ? q.options[userAns] : 'No answer')
      : userAns || 'No answer';
    const correctDisplay = currentQuiz.type === 'mcq' ? q.options[q.answer] : q.expected.join(' / ');
    html += `<div class="question ${status}">
      <h3>Q${i + 1}: ${q.question}</h3>
      <p><strong>Your Answer:</strong> ${userDisplay}</p>
      <p><strong>Correct:</strong> ${correctDisplay}</p>
      <div class="explanation">${q.explanation}</div>
    </div>`;
  });
  html += '<button onclick="location.reload()">New Quiz</button>';
  document.getElementById('results').innerHTML = html;
  document.getElementById('results').style.display = 'block';
  document.getElementById('quiz-area').style.display = 'none';
}

function shuffle(arr) { for (let i = arr.length - 1; i > 0; i--) { const j = Math.floor(Math.random() * (i + 1)); [arr[i], arr[j]] = [arr[j], arr[i]]; } return arr; }
</script>
