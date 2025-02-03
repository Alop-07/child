<!DOCTYPE html>
<html>
<head>
    <title>Memory Card Game</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css">
    <style>
        body {
            display: flex;
            flex-direction: column;
            align-items: center;
            min-height: 100vh;
            margin: 0;
            background-color: #4f64d8;
            font-family: Arial, sans-serif;
        }

        .game-board {
            display: grid;
            grid-template-columns: repeat(4, 100px);
            gap: 10px;
            margin: 20px;
        }
        .time {
            color: white;
        }

        .moves {
            color: white;
        }

        .card {
            width: 100px;
            height: 100px;
            perspective: 1000px;
            cursor: pointer;
            position: relative;
        }

        .card-front, .card-back {
            position: absolute;
            width: 100%;
            height: 100%;
            backface-visibility: hidden;
            display: flex;
            justify-content: center;
            align-items: center;
            border-radius: 10px;
            transition: transform 0.6s;
            box-shadow: 0 2px 5px rgba(0,0,0,0.2);
        }

        .card-back {
            background: #2196F3;
            color: white;
            font-size: 2em;
            transform: rotateY(0deg);
        }

        .card-front {
            background: white;
            transform: rotateY(180deg);
        }

        .card.flipped .card-back {
            transform: rotateY(180deg);
        }

        .card.flipped .card-front {
            transform: rotateY(360deg);
        }

        .card.matched {
            opacity: 0.5;
            cursor: not-allowed;
        }

        .score-panel {
            text-align: center;
            margin: 20px 0;
            display: flex;
            justify-content: space-between;
            max-width: 500px;
            margin: 20px auto;
        }

        .restart-btn {
            padding: 10px 20px;
            background: #2196F3;
            border: none;
            border-radius: 5px;
            color: white;
            cursor: pointer;
        }

        .win-modal {
            display: none;
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 0 20px rgba(0,0,0,0.2);
            text-align: center;
        }

        .stars {
            margin: 10px 0;
        }

        .star {
            color: gold;
            margin: 0 2px;
        }
    </style>
</head>
<body>
    <div class="score-panel">
        <div class="timer">Time: <span id="time">0</span>s</div>
        <div class="moves">Moves: <span id="moves">0</span></div>
        <div class="stars">
            <i class="fas fa-star star"></i>
            <i class="fas fa-star star"></i>
            <i class="fas fa-star star"></i>
        </div>
        <button class="restart-btn" onclick="game.restart()"><i class="fas fa-redo"></i></button>
    </div>
    <div class="game-board" id="gameBoard"></div>
    <div class="win-modal" id="winModal">
        <h2>Congratulations! 🎉</h2>
        <p>You won in <span id="finalTime">0</span> seconds!</p>
        <p>With <span id="finalMoves">0</span> moves</p>
        <div id="finalStars"></div>
        <button class="restart-btn" onclick="game.restart()">Play Again</button>
    </div>

    <script>
        class MemoryGame {
            constructor() {
                this.cards = [];
                this.flippedCards = [];
                this.moves = 0;
                this.time = 0;
                this.timerId = null;
                this.stars = 3;
                this.canFlip = true;
                this.images = [
                    'https://picsum.photos/100?1',
                    'https://picsum.photos/100?2',
                    'https://picsum.photos/100?3',
                    'https://picsum.photos/100?4',
                    'https://picsum.photos/100?5',
                    'https://picsum.photos/100?6'
                ];

                this.init();
            }

            init() {
                this.gameBoard = document.getElementById('gameBoard');
                this.createCards();
                this.render();
            }

            createCards() {
                const imagePairs = [...this.images, ...this.images];
                const shuffledImages = imagePairs.sort(() => Math.random() - 0.5);

                this.cards = shuffledImages.map((img, index) => {
                    const cardEl = document.createElement('div');
                    cardEl.className = 'card';
                    cardEl.innerHTML = `
                        <div class="card-front">
                            <img src="${img}" alt="Card">
                        </div>
                        <div class="card-back">?</div>
                    `;

                    return {
                        id: index,
                        img,
                        isFlipped: false,
                        isMatched: false,
                        element: cardEl
                    };
                });
            }

            render() {
                this.gameBoard.innerHTML = '';
                this.cards.forEach(card => {
                    const classes = ['card'];
                    if (card.isFlipped) classes.push('flipped');
                    if (card.isMatched) {
                        classes.push('matched');
                        card.element.style.pointerEvents = 'none';
                    } else {
                        card.element.style.pointerEvents = 'auto';
                    }
                    card.element.className = classes.join(' ');
                    card.element.onclick = () => this.flipCard(card);
                    this.gameBoard.appendChild(card.element);
                });
            }

            flipCard(card) {
                if (!this.canFlip || this.flippedCards.length === 2 || card.isFlipped || card.isMatched) return;
                if (!this.timerId) this.startTimer();

                card.isFlipped = true;
                this.flippedCards.push(card);
                this.render();

                if (this.flippedCards.length === 2) {
                    this.moves++;
                    this.updateScore();
                    this.checkMatch();
                }
            }

            checkMatch() {
                this.canFlip = false;
                const [card1, card2] = this.flippedCards;

                if (card1.img === card2.img) {
                    card1.isMatched = true;
                    card2.isMatched = true;
                    this.playMatchSound();
                    this.canFlip = true;
                } else {
                    setTimeout(() => {
                        card1.isFlipped = false;
                        card2.isFlipped = false;
                        this.render();
                        this.canFlip = true;
                    }, 1000);
                }

                this.flippedCards = [];

                if (this.cards.every(c => c.isMatched)) {
                    this.gameWon();
                }
            }

            updateScore() {
                document.getElementById('moves').textContent = this.moves;
                this.updateStars();
            }

            updateStars() {
                const stars = document.querySelectorAll('.star');
                this.stars = this.moves < 15 ? 3 : this.moves < 25 ? 2 : 1;
                stars.forEach((star, index) => {
                    star.style.color = index < this.stars ? 'gold' : 'lightgray';
                });
            }

            startTimer() {
                this.timerId = setInterval(() => {
                    this.time++;
                    document.getElementById('time').textContent = this.time;
                }, 1000);
            }

            gameWon() {
                clearInterval(this.timerId);
                const winModal = document.getElementById('winModal');
                winModal.style.display = 'block';
                document.getElementById('finalTime').textContent = this.time;
                document.getElementById('finalMoves').textContent = this.moves;
                document.getElementById('finalStars').innerHTML =
                    Array(this.stars).fill('<i class="fas fa-star star"></i>').join('');
            }

            playMatchSound() {
                const audioContext = new (window.AudioContext || window.webkitAudioContext)();
                if (audioContext.state === 'suspended') {
                    audioContext.resume();
                }

                const oscillator = audioContext.createOscillator();
                const gainNode = audioContext.createGain();

                oscillator.connect(gainNode);
                gainNode.connect(audioContext.destination);

                oscillator.frequency.value = 440;
                gainNode.gain.setValueAtTime(0.1, audioContext.currentTime);

                oscillator.start();
                setTimeout(() => oscillator.stop(), 200);
            }

            restart() {
                clearInterval(this.timerId);
                this.timerId = null;
                this.moves = 0;
                this.time = 0;
                this.stars = 3;
                this.flippedCards = [];
                this.canFlip = true;
                document.getElementById('time').textContent = '0';
                document.getElementById('moves').textContent = '0';
                document.querySelectorAll('.star').forEach(star => star.style.color = 'gold');
                document.getElementById('winModal').style.display = 'none';
                this.createCards();
                this.render();
            }
        }

        const game = new MemoryGame();
    </script>
</body>
</html>
