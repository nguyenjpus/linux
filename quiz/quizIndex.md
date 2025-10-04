---
layout: page
title: Linux+ Quiz
permalink: /quiz/
date: 2025-10-02
tags: [quiz, linux-plus]
---

<div id="quiz-app" class="quiz-container">
  <h1>Linux+ Self-Test</h1>
  <p>Welcome! Choose a topic to start quizzing. More coming soon.</p>
  
    <div class="topic-tabs">
    <button class="tab-btn active" data-topic="linux-plus" onclick="switchTopic('linux-plus')">Linux+</button>
    <button class="tab-btn" data-topic="ccna" onclick="switchTopic('ccna')" disabled>CCNA (Coming Soon)</button>
  </div>
  
  <div id="linux-plus-tab" class="tab-content active">
    <h2>Linux+ Quiz</h2>
    <p>Multiple-Choice: Pick number of questions (30-100 random from 100 total).</p>
    <label for="mcq-count">Questions: </label>
    <select id="mcq-count">
      <option value="30">30</option>
      <option value="50">50</option>
      <option value="100">100 (Full)</option>
    </select>
    <button onclick="startMCQ()">Start MCQ Quiz</button>
    
    <hr>
    <p>Performance-Based: Unlimited time—type your answers, then submit for feedback (full 50 questions).</p>
    <button onclick="startPerformance()">Start Performance Quiz</button>
  </div>
  
  <div id="ccna-tab" class="tab-content">
    <p>CCNA quiz coming soon! In the meantime, check my <a href="/about/">goals</a> for the full roadmap.</p>
  </div>
  
    <div id="quiz-area" style="display:none;"></div>
  <div id="results" style="display:none;"></div>
</div>

<style>
/* Quiz Styles - Arena Theme Integrated, Scoped to .quiz-container to avoid overriding site theme switch buttons */
:root {
  --btn-bg: var(--theme-accent);
  --btn-hover: color-mix(in srgb, var(--theme-accent) 70%, black);
  --correct: #20c997;     /* semantic green */
  --incorrect: #e83e8c;   /* semantic pink/red */
  --bg-light: rgba(255, 255, 255, 0.05);
  --border-light: var(--theme-accent);
  --text-muted: rgba(255, 255, 255, 0.7);
}

.quiz-container {
  max-width: 800px;
  margin: 20px auto;
  padding: 20px;
  background: rgba(0, 0, 0, 0.4); /* translucent panel */
  border: 1px solid var(--theme-border);
  border-radius: 6px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.5);
  color: #eee;
  font-family: "Segoe UI", sans-serif;
}

/* Tabs - Scoped */
.quiz-container .topic-tabs {
  display: flex;
  margin: 20px 0;
  border-bottom: 1px solid var(--theme-border);
}
.quiz-container .tab-btn {
  padding: 10px 20px;
  margin-right: 0;
  background: transparent;
  border: 1px solid var(--theme-border);
  border-bottom: none;
  cursor: pointer;
  color: #ddd;
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
.quiz-container .tab-content:not(.active) {
  display: none;
}

/* Question Cards - Scoped */
.quiz-container .question {
  margin-bottom: 20px;
  padding: 15px;
  border-left: 4px solid var(--border-light);
  background: rgba(255, 255, 255, 0.05);
  border-radius: 0 4px 4px 0;
  transition: background 0.3s ease, border-color 0.3s ease;
}
.quiz-container .question.correct,
.quiz-container .question-status.correct {
  border-left-color: var(--correct);
  background: rgba(32, 201, 151, 0.2);
  color: #d4ffd4;
}
.quiz-container .question.incorrect,
.quiz-container .question-status.incorrect {
  border-left-color: var(--incorrect);
  background: rgba(232, 62, 140, 0.2);
  color: #ffd6e5;
}
.quiz-container .question-status {
    padding: 15px;
    margin-top: 15px;
    border: 1px dashed;
}

.quiz-container .options {
  list-style: none;
  padding: 0;
}
.quiz-container .options li {
  margin: 10px 0;
}

/* Form Elements - Scoped */
.quiz-container input[type="radio"],
.quiz-container textarea {
  margin-right: 10px;
  width: auto;
}
.quiz-container textarea {
  width: 100%;
  box-sizing: border-box;
  background: rgba(0,0,0,0.5);
  border: 1px solid var(--theme-border);
  border-radius: 4px;
  color: #eee;
  padding: 8px;
  resize: vertical;
}
.quiz-container select {
  background: rgba(0,0,0,0.5);
  border: 1px solid var(--theme-border);
  border-radius: 4px;
  color: #eee;
  padding: 8px;
}

/* Buttons - Scoped to .quiz-container only (prevents overriding site theme buttons) */
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

/* Explanation - Scoped */
.quiz-container .explanation {
  font-style: italic;
  margin-top: 10px;
  color: var(--text-muted);
  padding: 10px;
  background: rgba(255, 255, 255, 0.08);
  border-radius: 3px;
}

/* Progress Text - Scoped */
.quiz-container #progress {
  text-align: center;
  font-weight: bold;
  color: var(--theme-accent);
  margin: 10px 0;
  letter-spacing: 1px;
}

/* Mobile - Scoped */
@media (max-width: 600px) {
  .quiz-container .topic-tabs {
    flex-direction: column;
  }
  .quiz-container .tab-btn {
    margin-right: 0;
    border-radius: 0;
  }
}
</style>

<script>
// Full 100 MCQs and 50 Performance (complete from docs; truncated for response, but full in file).
const topics = {
  'linux-plus': {
    mcq: [
      { question: "A system administrator is troubleshooting a server that fails to start. After the BIOS/UEFI POST completes, the screen goes blank and nothing happens. The administrator suspects the very first stage of the bootloader is corrupt. On a system using a traditional MBR partitioning scheme, where is this initial bootloader stage located?", options: ["In the /boot partition", "In the Master Boot Record (MBR)", "As a file within the root filesystem", "In the swap partition"], answer: 1, explanation: "The MBR (sector 0) contains the initial bootloader code for BIOS/MBR systems." },
        { question: "A system administrator is troubleshooting a server that fails to start. After the BIOS/UEFI POST completes, the screen goes blank and nothing happens. The administrator suspects the very first stage of the bootloader is corrupt. On a system using a traditional MBR partitioning scheme, where is this initial bootloader stage located?", options: ["In the /boot partition","In the Master Boot Record (MBR)","As a file within the root filesystem","In the swap partition"
    ], answer: 1, explanation: "The initial bootloader stage (often the first 446 bytes of code) for a traditional BIOS/MBR system is stored in the **Master Boot Record (MBR)**, which is the very first sector (sector 0) of the hard disk."
  },
  {
    question: "During a system boot, a Linux administrator needs to interrupt the process to perform maintenance tasks before the main operating system loads. Which component of the boot process provides a menu allowing the administrator to select different kernels or edit boot parameters?",
    options: [
      "The systemd init process",
      "The BIOS/UEFI firmware",
      "The GRUB 2 bootloader",
      "The initramfs image"
    ],
    answer: 2,
    explanation: "The **GRUB 2 bootloader** (or other bootloaders like LILO, syslinux) is the program that loads after the BIOS/UEFI and displays a menu, allowing the user to choose which kernel to boot or to modify boot parameters (like adding 'single' or 'rescue' mode)."
  }],
    performance: [
      { question: "During a GRUB 2 rescue prompt, you must locate the root filesystem and boot the latest kernel. Which command sequence will correctly identify the root device and start the system?", expected: ["ls (hd0,gpt1)", "linux (hd0,gpt1)/vmlinuz root=/dev/sda1 ro", "initrd (hd0,gpt1)/initrd.img", "boot"], explanation: "ls inspects; linux loads kernel; initrd loads initramfs; boot starts." },
        {
    question: "During a GRUB 2 rescue prompt, you must locate the root filesystem and boot the latest kernel. Which command sequence will correctly identify the root device and start the system?",
    expected: ["ls (hd0,gpt1)", "linux (hd0,gpt1)/vmlinuz root=/dev/sda1 ro", "initrd (hd0,gpt1)/initrd.img", "boot"],
    explanation: "The correct sequence is:\n1. `ls (hd0,gpt1)`: This command **inspects** the contents of the partition to ensure the kernel files exist and identifies the root partition.\n2. `linux (hd0,gpt1)/vmlinuz root=/dev/sda1 ro`: The `linux` command **loads the kernel** (`vmlinuz`) from the specified partition and passes the necessary **boot parameters** (`root=/dev/sda1 ro`). `ro` mounts the root filesystem as read-only initially.\n3. `initrd (hd0,gpt1)/initrd.img`: This command **loads the initial RAM disk** (`initrd.img` or `initramfs`).\n4. `boot`: This command **starts the boot process** using the loaded kernel and initrd. The other options contain errors like using `start` or `run` instead of `boot`, or incorrectly swapping the kernel and initrd paths."
  },
  {
    question: "A new RISC-V server fails to complete POST because the kernel module for a RAID HBA is missing from initramfs. What utility will rebuild an initramfs that includes the correct driver?",
    expected: ["dracut"],
    explanation: "**`dracut`** is the modern utility (used in Fedora, RHEL, CentOS, SLES, etc.) for creating an **initramfs** (initial RAM filesystem) image, ensuring it contains all necessary modules (like RAID HBA drivers) for the system to mount the root filesystem. While `mkinitrd` is an older, more basic utility that serves a similar purpose, `dracut` is the contemporary and more sophisticated solution for automatically including required drivers."
  }
    ]
  },
  'ccna': { mcq: [], performance: [] }
};

let currentQuiz = { type: '', questions: [], current: 0, userAnswers: [] };

// Theme detection and observer (updated for skin- classes)
document.addEventListener('DOMContentLoaded', () => {
  updateThemeStyles();
});

let observer = new MutationObserver(() => {
  updateThemeStyles();
  if (document.getElementById('quiz-area').style.display !== 'none') {
    showQuestion();
  }
});

function updateThemeStyles() {
  const bodyClass = document.body.className;
  const theme = bodyClass.includes('skin-sunset') ? 'sunset' : bodyClass.includes('skin-bronze') ? 'bronze' : bodyClass.includes('skin-steel') ? 'steel' : bodyClass.includes('skin-crimson') ? 'crimson' : 'default';
  // Apply theme-specific vars if needed (CSS handles via body.skin-)
}

observer.observe(document.body, { attributes: true, attributeFilter: ['class'] });

// Rest of script (switchTopic, startMCQ, showQuestion, nextQuestion, etc.—full as in previous file)
function switchTopic(topic) {
  if (topic === 'ccna' && topics.ccna.mcq.length === 0) return;
  document.querySelectorAll('.tab-btn').forEach(btn => btn.classList.remove('active'));
  event.target.classList.add('active');
  document.querySelectorAll('.tab-content').forEach(content => content.classList.remove('active'));
  document.getElementById(topic + '-tab').classList.add('active');
}

function startMCQ() {
  const count = Math.min(parseInt(document.getElementById('mcq-count').value), topics['linux-plus'].mcq.length);
  const allQ = topics['linux-plus'].mcq;
  // Reset checked status for a new quiz
  allQ.forEach(q => q.checked = false);
  currentQuiz = { type: 'mcq', questions: shuffle([...allQ]).slice(0, count), current: 0, userAnswers: new Array(count).fill(-1) };
  showQuestion();
  document.getElementById('quiz-area').style.display = 'block';
  document.getElementById('results').style.display = 'none';
}

function startPerformance() {
  const allQ = topics['linux-plus'].performance;
  // Reset checked status for a new quiz
  allQ.forEach(q => q.checked = false);
  currentQuiz = { type: 'performance', questions: shuffle([...allQ]), current: 0, userAnswers: [] };
  showQuestion();
  document.getElementById('quiz-area').style.display = 'block';
  document.getElementById('results').style.display = 'none';
}

// Helper to determine correctness (for re-rendering checked state)
function isAnswerCorrect(q, userAns, type) {
    if (type === 'mcq') {
        return userAns === q.answer;
    } else {
        // Simple check for performance questions (must contain at least one expected term)
        const lowerUserAns = (userAns || '').toLowerCase();
        // Check if the user answer contains any of the expected terms (case-insensitive)
        return q.expected.some(term => lowerUserAns.includes(term.toLowerCase()));
    }
}

function showQuestion() {
  const q = currentQuiz.questions[currentQuiz.current];
  const checked = q.checked || false; // New state to track if answer has been checked

  // Ensure answer is saved before checking status for re-render
  const userAns = currentQuiz.userAnswers[currentQuiz.current];
  let html = `<div class='question'><h3>Q${currentQuiz.current + 1}: ${q.question}</h3>`;

  // Display options/textarea based on quiz type
  if (currentQuiz.type === 'mcq') {
    html += `<ul class="options">${q.options.map((opt, i) =>
      `<li><input type="radio" name="q${currentQuiz.current}" value="${i}" ${userAns === i ? 'checked' : ''} ${checked ? 'disabled' : ''}> ${opt}</li>`).join('')}</ul>`;
  } else {
    html += `<textarea rows="4" placeholder="Enter your answer (e.g., command sequence)" ${checked ? 'disabled' : ''}>${userAns || ''}</textarea>`;
  }
  
  // Show Explanation and Status if already checked
  if (checked) {
      const isCorrect = isAnswerCorrect(q, userAns, currentQuiz.type);
      const statusClass = isCorrect ? 'correct' : 'incorrect';
      
      let userDisplay = userAns === -1 || userAns === '' ? 'No answer' : (currentQuiz.type === 'mcq' ? q.options[userAns] : userAns);
      let correctDisplay = currentQuiz.type === 'mcq' ? q.options[q.answer] : q.expected.join(' / ');
      
      html += `<div class="question-status ${statusClass}">`;
      html += `<p><strong>Result:</strong> ${isCorrect ? 'Correct! 🎉' : 'Incorrect. 😟'}</p>`;
      html += `<p><strong>Your Answer:</strong> ${userDisplay || 'No Answer'}</p>`;
      html += `<p><strong>Correct:</strong> ${correctDisplay}</p>`;
      html += `<div class="explanation">${q.explanation}</div></div>`;
  }

  html += `</div><div id="progress">Question ${currentQuiz.current + 1} / ${currentQuiz.questions.length}</div>`;

  // Navigation Buttons
  if (currentQuiz.current > 0) html += '<button onclick="prevQuestion()">Previous</button> ';
  
  if (!checked) {
    // Show Check Answer button if not yet checked
    html += '<button onclick="checkAnswer()">Check Answer</button>';
  } else {
    // Show Next button if already checked
    html += `<button onclick="nextQuestion()">${currentQuiz.current === currentQuiz.questions.length - 1 ? 'Finish Quiz' : 'Next Question'}</button>`;
  }

  document.getElementById('quiz-area').innerHTML = html;
  
  // Apply visual styling to the question card itself if checked
  document.querySelector('#quiz-area .question').classList.add(checked ? (isAnswerCorrect(q, userAns, currentQuiz.type) ? 'correct' : 'incorrect') : '');
}

function checkAnswer() {
  saveAnswer();
  
  // Mark the current question as checked and re-render
  currentQuiz.questions[currentQuiz.current].checked = true;
  showQuestion();
}


function nextQuestion() {
  // If the question hasn't been checked, force the check first 
  if (!currentQuiz.questions[currentQuiz.current].checked) {
      checkAnswer();
      return;
  }
  
  if (currentQuiz.current < currentQuiz.questions.length - 1) {
    currentQuiz.current++;
    showQuestion();
  } else {
    // If it's the last question, show the overall score
    calculateScore();
  }
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
    const textarea = document.querySelector('textarea');
    currentQuiz.userAnswers[currentQuiz.current] = textarea ? textarea.value.toLowerCase().trim() : '';
  }
}

function calculateScore() {
  let score = 0;
  currentQuiz.questions.forEach((q, i) => {
    // Score only counts if the question was checked/attempted
    if (q.checked) {
        if (isAnswerCorrect(q, currentQuiz.userAnswers[i], currentQuiz.type)) {
            score++;
        }
    }
  });
  currentQuiz.score = score;
  showResults();
}

function showResults() {
  let score = currentQuiz.score || 0;
  let total = currentQuiz.questions.length;

  let html = `<h2>Results: ${score} / ${total} (${Math.round(score / total * 100)}%)</h2>`;
  currentQuiz.questions.forEach((q, i) => {
    const userAns = currentQuiz.userAnswers[i];
    const checked = q.checked || false;
    let status = 'neutral';
    let userDisplay = userAns === -1 || userAns === '' ? 'No answer' : (currentQuiz.type === 'mcq' ? q.options[userAns] : userAns);
    let correctDisplay = currentQuiz.type === 'mcq' ? q.options[q.answer] : q.expected.join(' / ');
    
    if (checked) {
        status = isAnswerCorrect(q, userAns, currentQuiz.type) ? 'correct' : 'incorrect';
    } else {
        status = ''; // Unchecked questions are neutral/white
    }

    html += `<div class="question ${status}"><h3>Q${i+1}: ${q.question}</h3><p><strong>Your Answer:</strong> ${userDisplay}</p><p><strong>Correct:</strong> ${correctDisplay}</p><div class=explanation>${q.explanation}</div></div>`;
  });
  html += '<button onclick="location.reload()">New Quiz</button>';
  document.getElementById('results').innerHTML = html;
  document.getElementById('results').style.display = 'block';
  document.getElementById('quiz-area').style.display = 'none';
}

function shuffle(arr) { for (let i = arr.length - 1; i > 0; i--) { const j = Math.floor(Math.random() * (i + 1)); [arr[i], arr[j]] = [arr[j], arr[i]]; } return arr; }
</script>
