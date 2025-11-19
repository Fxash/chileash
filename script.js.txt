// Fx Ash Trading Analyzer - WebSocket Auto Connect
const APP_ID = '112324';
const API_TOKEN = 'ZAUmyCyWE6aGRgf';

let socket;
let analysisRunning = false;
let tickHistory = [];
let lastDigit = null;

const startBtn = document.getElementById('start-btn');
const stopBtn = document.getElementById('stop-btn');
const syntheticSelect = document.getElementById('synthetic-select');
const tickSequenceSelect = document.getElementById('tick-sequence');

const lastDigitDisplay = document.getElementById('last-digit');
const entryDisplay = document.getElementById('entry');
const historyDisplay = document.getElementById('history-data');
const overBox = document.getElementById('over-box');
const underBox = document.getElementById('under-box');

const circles = Array.from(document.querySelectorAll('.circle'));

// Connect WebSocket
function connectWebSocket() {
  socket = new WebSocket(`wss://ws.derivws.com/websockets/v3?app_id=${APP_ID}`);

  socket.onopen = () => {
    console.log('Connected to Deriv');
    authorize();
  };

  socket.onmessage = (msg) => {
    const data = JSON.parse(msg.data);
    if (data.error) {
      console.error('Deriv Error:', data.error.message);
      return;
    }

    if (data.msg_type === 'authorize') {
      console.log('Authorized successfully');
      subscribeTicks();
    }

    if (data.msg_type === 'tick') {
      handleTick(data.tick);
    }
  };

  socket.onerror = (err) => console.error('WebSocket error', err);
  socket.onclose = () => console.log('Connection closed');
}

// Authorize
function authorize() {
  socket.send(JSON.stringify({
    authorize: API_TOKEN
  }));
}

// Subscribe to ticks
function subscribeTicks() {
  const market = syntheticSelect.value;
  socket.send(JSON.stringify({
    ticks: market,
    subscribe: 1
  }));
}

// Handle incoming tick
function handleTick(tick) {
  lastDigit = parseInt(tick.quote.toString().slice(-1));
  lastDigitDisplay.textContent = `Last Digit: ${lastDigit} (Price: ${tick.quote})`;

  tickHistory.push(lastDigit);
  if (tickHistory.length > 10) tickHistory.shift();

  historyDisplay.textContent = tickHistory.join(', ');

  updateCircles();
  updateProbabilities();
  determineEntry();
}

// Update circle highlights & percentages
function updateCircles() {
  const counts = Array(10).fill(0);
  tickHistory.forEach(d => counts[d]++);
  
  circles.forEach(circle => {
    const digit = parseInt(circle.dataset.digit);
    const percent = ((counts[digit] / tickHistory.length) * 100).toFixed(1);
    circle.querySelector('.percent').textContent = `${percent}%`;

    circle.classList.remove('selected');
    if (digit === lastDigit) circle.classList.add('selected');
  });
}

// Update over/under probabilities
function updateProbabilities() {
  const overCount = tickHistory.filter(d => d > 3).length;
  const underCount = tickHistory.filter(d => d <= 3).length;
  const total = tickHistory.length || 1;

  overBox.textContent = `Over: ${((overCount / total) * 100).toFixed(1)}%`;
  underBox.textContent = `Under: ${((underCount / total) * 100).toFixed(1)}%`;
}

// Determine entry based on tick sequence
function determineEntry() {
  const sequenceLength = parseInt(tickSequenceSelect.value);

  // Basic example: pick the first under/over digit meeting your rules
  let candidate = null;

  // Count occurrences
  const counts = Array(10).fill(0);
  tickHistory.forEach(d => counts[d]++);

  // Example: if over 3 rule
  const overDigits = [];
  for (let i = 4; i <= 9; i++) if (counts[i] >= 1) overDigits.push(i);

  // Pick entry based on sequence length and over/under rules
  if (overDigits.length > 0) candidate = overDigits[0];
  else candidate = tickHistory[tickHistory.length - 1] || 0;

  entryDisplay.textContent = `Entry: ${candidate}`;
}

// Start/Stop Analysis
startBtn.onclick = () => {
  if (!analysisRunning) {
    analysisRunning = true;
    connectWebSocket();
  }
};

stopBtn.onclick = () => {
  analysisRunning = false;
  if (socket) socket.close();
};

// Circle click to manually select
circles.forEach(circle => {
  circle.addEventListener('click', () => {
    const digit = circle.dataset.digit;
    entryDisplay.textContent = `Entry: ${digit}`;
  });
});
