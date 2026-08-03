<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Game Kartu Misteri</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Pengaturan tema misteri untuk background */
        body {
            background: radial-gradient(circle at center, #2d1b4e 0%, #0a0514 100%);
            min-height: 100vh;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }

        /* Animasi dan efek 3D untuk membalik kartu */
        .card-container {
            perspective: 1000px;
            cursor: pointer;
        }
        .card-inner {
            position: relative;
            width: 100%;
            height: 100%;
            text-align: center;
            transition: transform 0.6s cubic-bezier(0.4, 0.2, 0.2, 1);
            transform-style: preserve-3d;
            box-shadow: 0 4px 8px rgba(0,0,0,0.5);
            border-radius: 0.75rem;
        }
        .card-container.flipped .card-inner {
            transform: rotateY(180deg);
        }
        .card-front, .card-back {
            position: absolute;
            width: 100%;
            height: 100%;
            backface-visibility: hidden;
            border-radius: 0.75rem;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 3rem;
        }
        /* Bagian belakang kartu (saat belum ditebak) */
        .card-front {
            background: linear-gradient(135deg, #4c1d95 0%, #1e1b4b 100%);
            border: 2px solid #8b5cf6;
            color: #c4b5fd;
            transform: rotateY(0deg);
        }
        .card-front::after {
            content: '?';
            font-size: 2.5rem;
            font-weight: bold;
            text-shadow: 0 0 10px rgba(139, 92, 246, 0.8);
        }
        /* Bagian depan kartu (gambar/ikon misteri) */
        .card-back {
            background: #1f2937;
            border: 2px solid #34d399;
            transform: rotateY(180deg);
        }
        
        /* Efek hover agar lebih interaktif */
        .card-container:hover .card-front {
            box-shadow: 0 0 15px rgba(139, 92, 246, 0.6);
        }

        /* Animasi kartu cocok */
        @keyframes pulse-match {
            0% { transform: rotateY(180deg) scale(1); }
            50% { transform: rotateY(180deg) scale(1.1); }
            100% { transform: rotateY(180deg) scale(1); }
        }
        .card-container.matched .card-inner {
            animation: pulse-match 0.5s ease-in-out;
        }
        .card-container.matched .card-back {
            background: #064e3b;
            border-color: #10b981;
        }
    </style>
</head>
<body class="text-white flex flex-col items-center justify-center p-4">

    <div class="max-w-2xl w-full text-center mt-8">
        <h1 class="text-4xl md:text-5xl font-bold mb-2 text-transparent bg-clip-text bg-gradient-to-r from-purple-400 to-pink-600 drop-shadow-lg">
            Kartu Misteri
        </h1>
        <p class="text-purple-200 mb-6">Temukan semua pasangan kartu yang tersembunyi di dalam kegelapan!</p>
        
        <div class="flex justify-between items-center bg-black bg-opacity-40 p-4 rounded-xl mb-8 border border-purple-800 shadow-xl">
            <div class="text-xl font-semibold">
                Langkah: <span id="moves" class="text-pink-400">0</span>
            </div>
            <button onclick="initGame()" class="px-6 py-2 bg-gradient-to-r from-purple-600 to-indigo-600 hover:from-purple-500 hover:to-indigo-500 rounded-lg font-bold shadow-lg shadow-purple-500/30 transition-all duration-200 transform hover:scale-105 active:scale-95">
                Mulai Ulang
            </button>
        </div>

        <!-- Grid Kartu -->
        <div id="game-board" class="grid grid-cols-4 gap-3 md:gap-4 mb-8">
            <!-- Kartu akan dihasilkan oleh JavaScript di sini -->
        </div>
    </div>

    <!-- Modal Kemenangan -->
    <div id="win-modal" class="fixed inset-0 bg-black bg-opacity-80 flex items-center justify-center hidden z-50">
        <div class="bg-gradient-to-b from-gray-900 to-purple-900 p-8 rounded-2xl border-2 border-purple-500 shadow-2xl text-center max-w-sm mx-4 transform scale-95 transition-transform duration-300">
            <div class="text-6xl mb-4">🏆</div>
            <h2 class="text-3xl font-bold mb-2 text-transparent bg-clip-text bg-gradient-to-r from-yellow-300 to-yellow-600">Luar Biasa!</h2>
            <p class="text-lg text-purple-200 mb-6">Anda memecahkan misteri dalam <strong id="final-moves" class="text-white text-xl">0</strong> langkah.</p>
            <button onclick="initGame()" class="w-full py-3 bg-gradient-to-r from-green-500 to-emerald-600 hover:from-green-400 hover:to-emerald-500 rounded-xl font-bold text-lg shadow-lg shadow-green-500/40 transition-all duration-200 transform hover:scale-105">
                Main Lagi
            </button>
        </div>
    </div>

    <script>
        // Kumpulan ikon misteri (Emoji)
        const baseSymbols = ['🔮', '🌙', '🦇', '🕷️', '🦉', '👻', '💎', '🗝️'];
        let cards = [];
        let flippedCards = [];
        let matchedPairs = 0;
        let moves = 0;
        let lockBoard = false; // Mencegah klik saat animasi berjalan

        const gameBoard = document.getElementById('game-board');
        const movesDisplay = document.getElementById('moves');
        const winModal = document.getElementById('win-modal');
        const finalMovesDisplay = document.getElementById('final-moves');

        // Fungsi untuk mengacak array (Fisher-Yates Shuffle)
        function shuffle(array) {
            let currentIndex = array.length, randomIndex;
            while (currentIndex !== 0) {
                randomIndex = Math.floor(Math.random() * currentIndex);
                currentIndex--;
                [array[currentIndex], array[randomIndex]] = [array[randomIndex], array[currentIndex]];
            }
            return array;
        }

        // Memulai dan mereset game
        function initGame() {
            gameBoard.innerHTML = '';
            flippedCards = [];
            matchedPairs = 0;
            moves = 0;
            lockBoard = false;
            movesDisplay.innerText = moves;
            
            // Menyembunyikan modal jika sedang tampil
            winModal.classList.add('hidden');
            winModal.querySelector('div').classList.replace('scale-100', 'scale-95');

            // Membuat pasangan kartu dan mengacaknya
            cards = shuffle([...baseSymbols, ...baseSymbols]);

            // Membuat elemen HTML untuk setiap kartu
            cards.forEach((symbol, index) => {
                const cardEl = document.createElement('div');
                cardEl.classList.add('card-container', 'h-24', 'md:h-32');
                cardEl.dataset.symbol = symbol;
                cardEl.dataset.index = index;
                
                cardEl.innerHTML = `
                    <div class="card-inner">
                        <div class="card-front"></div>
                        <div class="card-back">${symbol}</div>
                    </div>
                `;

                cardEl.addEventListener('click', () => flipCard(cardEl));
                gameBoard.appendChild(cardEl);
            });
        }

        // Fungsi untuk menangani saat kartu diklik
        function flipCard(card) {
            // Abaikan jika papan sedang dikunci, atau kartu sudah dibalik/cocok
            if (lockBoard) return;
            if (card.classList.contains('flipped')) return;

            card.classList.add('flipped');
            flippedCards.push(card);

            // Jika dua kartu sudah dibalik, periksa kecocokan
            if (flippedCards.length === 2) {
                moves++;
                movesDisplay.innerText = moves;
                checkForMatch();
            }
        }

        // Memeriksa apakah dua kartu yang dibalik memiliki simbol yang sama
        function checkForMatch() {
            const [card1, card2] = flippedCards;
            const isMatch = card1.dataset.symbol === card2.dataset.symbol;

            if (isMatch) {
                disableCards();
            } else {
                unflipCards();
            }
        }

        // Jika kartu cocok, biarkan terbuka dan tandai selesai
        function disableCards() {
            flippedCards[0].classList.add('matched');
            flippedCards[1].classList.add('matched');
            
            matchedPairs++;
            flippedCards = [];

            // Cek apakah pemain menang (semua pasangan ditemukan)
            if (matchedPairs === baseSymbols.length) {
                setTimeout(showWinModal, 500);
            }
        }

        // Jika kartu tidak cocok, balikkan kembali setelah jeda
        function unflipCards() {
            lockBoard = true; // Kunci papan agar tidak bisa klik kartu lain
            
            setTimeout(() => {
                flippedCards[0].classList.remove('flipped');
                flippedCards[1].classList.remove('flipped');
                flippedCards = [];
                lockBoard = false; // Buka kunci papan
            }, 1000); // Jeda 1 detik
        }

        // Menampilkan pesan kemenangan
        function showWinModal() {
            finalMovesDisplay.innerText = moves;
            winModal.classList.remove('hidden');
            // Sedikit delay untuk efek pop up
            setTimeout(() => {
                winModal.querySelector('div').classList.replace('scale-95', 'scale-100');
            }, 50);
        }

        // Memulai game pertama kali saat halaman dimuat
        document.addEventListener('DOMContentLoaded', initGame);

    </script>
</body>
</html>
