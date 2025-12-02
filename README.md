# FlaisySnack.Kelompok-10
html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Flaisy Snack - Pedas Nikmat yang Bikin Ketagihan!</title>
    <!-- Memuat Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Memuat Google Fonts: Inter (Teks Utama) dan Fredoka (Slogan/Judul) -->
    <link href="https://fonts.googleapis.com/css2?family=Fredoka:wght@400;700&family=Inter:wght@300;400;600;700&display=swap" rel="stylesheet">
    <!-- Memuat Ikon (Lucide) -->
    <script src="https://unpkg.com/lucide@latest"></script>

    <style>
        /* Konfigurasi Warna Soft & Elegan */
        :root {
            --color-primary: #FEE2E2; /* Soft Rose */
            --color-secondary: #FDE68A; /* Soft Yellow */
            --color-accent: #EF4444; /* Chili Red Pop */
            --color-text: #44403C; /* Dark Stone */
            --color-background: #FAFAF9; /* Light Cream */
        }
        body {
            font-family: 'Inter', sans-serif;
            color: var(--color-text);
            background-color: var(--color-background);
            overflow-x: hidden; /* Mencegah scroll horizontal */
        }
        .font-fredoka {
            font-family: 'Fredoka', sans-serif;
        }

        /* Styling untuk Animasi Cabai */
        .chili-container {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            overflow: hidden;
            z-index: 10; /* Di atas segalanya, di bawah modal */
        }
        .chili {
            position: absolute;
            font-size: 1.5rem; /* Ukuran cabai */
            color: var(--color-accent);
            animation: fall linear infinite;
        }
        @keyframes fall {
            to {
                transform: translateY(100vh);
                opacity: 0;
            }
        }
        /* Custom scrollbar untuk elemen komentar agar terlihat lebih rapih */
        #testimonials-list {
            scrollbar-width: thin;
            scrollbar-color: #EF4444 #FAFAF9;
        }
        #testimonials-list::-webkit-scrollbar {
            width: 8px;
        }
        #testimonials-list::-webkit-scrollbar-track {
            background: #FAFAF9;
            border-radius: 10px;
        }
        #testimonials-list::-webkit-scrollbar-thumb {
            background-color: #EF4444;
            border-radius: 10px;
            border: 2px solid #FAFAF9;
        }
    </style>
</head>
<body class="antialiased">

    <!-- Kontainer Animasi Cabai -->
    <div class="chili-container" id="chili-container"></div>

    <!-- Pemuatan Firebase dan Logika Aplikasi -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, addDoc, onSnapshot, collection, query, orderBy, serverTimestamp, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Mengatur tingkat log Firestore (Opsional, untuk debugging)
        setLogLevel('debug');

        // Variabel global dari Canvas
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let db, auth, userId = null;

        // --- Fungsi Helper ---

        /** Mengubah array buffer base64 menjadi array buffer */
        const base64ToArrayBuffer = (base64) => {
            const binaryString = atob(base64);
            const len = binaryString.length;
            const bytes = new Uint8Array(len);
            for (let i = 0; i < len; i++) {
                bytes[i] = binaryString.charCodeAt(i);
            }
            return bytes.buffer;
        };

        /** Membuat audio WAV dari data PCM (diperlukan untuk TTS) */
        const pcmToWav = (pcm16, sampleRate) => {
            const numChannels = 1;
            const bytesPerSample = 2; // 16-bit PCM
            const blockAlign = numChannels * bytesPerSample;
            const byteRate = sampleRate * blockAlign;

            const buffer = new ArrayBuffer(44 + pcm16.length * bytesPerSample);
            const view = new DataView(buffer);

            // RIFF chunk descriptor
            writeString(view, 0, 'RIFF');
            view.setUint32(4, 36 + pcm16.length * bytesPerSample, true);
            writeString(view, 8, 'WAVE');
            // FMT sub-chunk
            writeString(view, 12, 'fmt ');
            view.setUint32(16, 16, true); // Sub-chunk size
            view.setUint16(20, 1, true);  // Audio format (1 = PCM)
            view.setUint16(22, numChannels, true);
            view.setUint32(24, sampleRate, true);
            view.setUint32(28, byteRate, true);
            view.setUint16(32, blockAlign, true);
            view.setUint16(34, bytesPerSample * 8, true); // Bits per sample
            // Data sub-chunk
            writeString(view, 36, 'data');
            view.setUint32(40, pcm16.length * bytesPerSample, true);

            // Tulis data PCM 16-bit
            let offset = 44;
            for (let i = 0; i < pcm16.length; i++) {
                view.setInt16(offset, pcm16[i], true);
                offset += 2;
            }

            return new Blob([buffer], { type: 'audio/wav' });
        };

        const writeString = (view, offset, string) => {
            for (let i = 0; i < string.length; i++) {
                view.setUint8(offset + i, string.charCodeAt(i));
            }
        };


        // --- Inisialisasi Firebase ---
        async function initializeFirebase() {
            try {
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);

                // Autentikasi: Prioritas Custom Token, lalu Anonim
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }

                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        userId = user.uid;
                        console.log("Firebase Auth berhasil. User ID:", userId);
                        // Mulai mendengarkan komentar setelah auth berhasil
                        setupTestimonialListener();
                    } else {
                        // Ini mungkin hanya terjadi jika signInAnonymously gagal
                        console.error("Gagal mendapatkan user auth.");
                    }
                });
            } catch (error) {
                console.error("Gagal menginisialisasi Firebase:", error);
            }
        }

        // --- Logika Testimonial (Firestore) ---

        const TESTIMONIALS_COLLECTION = `artifacts/${appId}/public/data/flaisy_testimonials`;

        function setupTestimonialListener() {
            const testimonialsListEl = document.getElementById('testimonials-list');
            // Batas 10 komentar terbaru, diurutkan berdasarkan waktu (descending)
            const q = query(
                collection(db, TESTIMONIALS_COLLECTION),
                // Firestore tidak mendukung orderBy dan getDocs bersamaan tanpa index
                // Kita akan mengambil data apa adanya dan mengurutkannya di frontend
            );

            // Menggunakan onSnapshot untuk real-time update
            onSnapshot(q, (snapshot) => {
                let comments = [];
                snapshot.forEach((doc) => {
                    comments.push({ id: doc.id, ...doc.data() });
                });

                // Urutkan di frontend berdasarkan timestamp (terbaru dulu)
                comments.sort((a, b) => (b.timestamp?.seconds || 0) - (a.timestamp?.seconds || 0));

                testimonialsListEl.innerHTML = '';
                if (comments.length === 0) {
                    testimonialsListEl.innerHTML = '<p class="text-center text-stone-500 italic">Belum ada komentar. Jadilah yang pertama!</p>';
                } else {
                    comments.forEach(comment => {
                        const date = comment.timestamp ? new Date(comment.timestamp.seconds * 1000).toLocaleDateString('id-ID') : 'Tanggal tidak diketahui';
                        const el = `
                            <div class="bg-white p-4 rounded-xl shadow-lg border border-rose-100 mb-4 transition duration-300 hover:shadow-xl">
                                <p class="font-semibold text-rose-600">${comment.name || 'Anonim'}</p>
                                <p class="text-sm italic text-stone-500 mb-2">(${date})</p>
                                <p class="text-stone-700 leading-relaxed">${comment.comment}</p>
                            </div>
                        `;
                        testimonialsListEl.innerHTML += el;
                    });
                }
            }, (error) => {
                console.error("Error mendengarkan testimonials:", error);
                testimonialsListEl.innerHTML = '<p class="text-center text-red-500">Gagal memuat komentar.</p>';
            });
        }

        async function submitComment(event) {
            event.preventDefault();
            if (!db) {
                showMessageBox("Sistem belum siap. Coba lagi sebentar.", 'error');
                return;
            }

            const nameInput = document.getElementById('comment-name');
            const commentInput = document.getElementById('comment-text');

            const name = nameInput.value.trim();
            const comment = commentInput.value.trim();

            if (!name || !comment) {
                showMessageBox("Nama dan komentar harus diisi.", 'warning');
                return;
            }

            try {
                await addDoc(collection(db, TESTIMONIALS_COLLECTION), {
                    name: name,
                    comment: comment,
                    timestamp: serverTimestamp(),
                });

                nameInput.value = '';
                commentInput.value = '';
                showMessageBox("Terima kasih! Komentar Anda telah terkirim.", 'success');

            } catch (e) {
                console.error("Error menambahkan dokumen: ", e);
                showMessageBox("Gagal mengirim komentar. Mohon coba lagi.", 'error');
            }
        }

        // Hubungkan fungsi ke form submission
        document.addEventListener('DOMContentLoaded', () => {
            const commentForm = document.getElementById('comment-form');
            if (commentForm) {
                commentForm.addEventListener('submit', submitComment);
            }
        });


        // --- Logika Order dan WhatsApp ---

        const adminWaNumber = '6289510121183';
        const products = [
            { id: 'basreng', name: 'Basreng Pedas', price: 5000 },
            { id: 'makaroni', name: 'Makaroni Pedas', price: 5000 }
        ];

        function updateOrderSummary() {
            const basrengQty = parseInt(document.getElementById('qty-basreng').value) || 0;
            const makaroniQty = parseInt(document.getElementById('qty-makaroni').value) || 0;
            const totalQty = basrengQty + makaroniQty;

            const basrengTotal = basrengQty * products[0].price;
            const makaroniTotal = makaroniQty * products[1].price;
            const grandTotal = basrengTotal + makaroniTotal;

            document.getElementById('total-qty').textContent = totalQty;
            document.getElementById('total-price').textContent = grandTotal.toLocaleString('id-ID', { style: 'currency', currency: 'IDR', minimumFractionDigits: 0 });

            // Simpan Grand Total di hidden input untuk akses mudah saat submit
            document.getElementById('grand-total-input').value = grandTotal;

            // Atur status tombol
            const submitButton = document.getElementById('order-submit-button');
            if (grandTotal > 0) {
                submitButton.disabled = false;
                submitButton.classList.remove('opacity-50', 'cursor-not-allowed');
                submitButton.classList.add('hover:bg-red-700');
            } else {
                submitButton.disabled = true;
                submitButton.classList.add('opacity-50', 'cursor-not-allowed');
                submitButton.classList.remove('hover:bg-red-700');
            }
        }

        document.addEventListener('DOMContentLoaded', () => {
            document.getElementById('qty-basreng').addEventListener('input', updateOrderSummary);
            document.getElementById('qty-makaroni').addEventListener('input', updateOrderSummary);
            updateOrderSummary(); // Panggil saat memuat halaman

            const orderForm = document.getElementById('order-form');
            if (orderForm) {
                orderForm.addEventListener('submit', handleOrderSubmit);
            }
        });

        function handleOrderSubmit(event) {
            event.preventDefault();

            const name = document.getElementById('customer-name').value.trim();
            const address = document.getElementById('customer-address').value.trim();
            const paymentMethod = document.querySelector('input[name="payment-method"]:checked');
            const grandTotal = parseInt(document.getElementById('grand-total-input').value) || 0;

            if (!name || !address) {
                showMessageBox("Mohon lengkapi Nama dan Alamat pengiriman.", 'warning');
                return;
            }
            if (!paymentMethod) {
                showMessageBox("Mohon pilih Metode Pembayaran.", 'warning');
                return;
            }
            if (grandTotal === 0) {
                showMessageBox("Keranjang belanja Anda kosong. Tambahkan produk dulu!", 'warning');
                return;
            }

            const basrengQty = parseInt(document.getElementById('qty-basreng').value) || 0;
            const makaroniQty = parseInt(document.getElementById('qty-makaroni').value) || 0;

            // Format Pesan WhatsApp
            let orderDetails = `*PESANAN BARU - FLAISY SNACK*\n\n`;
            orderDetails += `*Data Pelanggan:*\n`;
            orderDetails += `Nama: ${name}\n`;
            orderDetails += `Alamat: ${address}\n\n`;

            orderDetails += `*Detail Pesanan:*\n`;
            if (basrengQty > 0) {
                orderDetails += `- Basreng Pedas: ${basrengQty} pack (Rp ${(basrengQty * 5000).toLocaleString('id-ID')})\n`;
            }
            if (makaroniQty > 0) {
                orderDetails += `- Makaroni Pedas: ${makaroniQty} pack (Rp ${(makaroniQty * 5000).toLocaleString('id-ID')})\n`;
            }

            orderDetails += `\n*Total Biaya:*\n`;
            orderDetails += `Rp ${grandTotal.toLocaleString('id-ID')}\n\n`;
            orderDetails += `*Metode Pembayaran:*\n`;
            orderDetails += `${paymentMethod.value}\n\n`;

            orderDetails += `Mohon konfirmasi pesanan ini. Terima kasih!`;

            // Encoding pesan untuk URL
            const encodedMessage = encodeURIComponent(orderDetails);
            const waLink = `https://wa.me/${adminWaNumber}?text=${encodedMessage}`;

            // Buka link WhatsApp
            window.open(waLink, '_blank');

            // Tampilkan pesan sukses
            showMessageBox("Pesanan berhasil dibuat! Anda akan dialihkan ke WhatsApp Admin untuk konfirmasi.", 'success');
        }

        // --- Logika Animasi Cabai ---

        function createChili() {
            const container = document.getElementById('chili-container');
            const chili = document.createElement('div');
            chili.classList.add('chili');
            chili.textContent = '🌶️'; // Emoji Cabai
            chili.style.left = `${Math.random() * 100}vw`;
            chili.style.animationDuration = `${Math.random() * 4 + 3}s`; // Durasi jatuh 3-7 detik
            chili.style.animationDelay = `${-Math.random() * 5}s`; // Mulai dari atas
            container.appendChild(chili);

            // Hapus elemen setelah animasi selesai
            setTimeout(() => {
                chili.remove();
            }, 7000); // Harus lebih lama dari durasi terlama
        }

        function startChiliAnimation() {
            // Hasilkan cabai baru setiap 300ms
            setInterval(createChili, 300);
        }

        // --- Logika Message Box (Pengganti Alert) ---
        function showMessageBox(message, type) {
            const box = document.getElementById('message-box');
            const iconEl = document.getElementById('message-icon');
            const textEl = document.getElementById('message-text');

            // Reset kelas
            box.className = "fixed bottom-4 right-4 p-4 rounded-xl shadow-2xl z-50 transform transition-all duration-300";
            iconEl.innerHTML = '';

            let bgColor = 'bg-blue-100 border-blue-400';
            let iconHtml = '<i data-lucide="info" class="w-6 h-6 text-blue-600"></i>';

            if (type === 'success') {
                bgColor = 'bg-green-100 border-green-400';
                iconHtml = '<i data-lucide="check-circle" class="w-6 h-6 text-green-600"></i>';
            } else if (type === 'warning') {
                bgColor = 'bg-yellow-100 border-yellow-400';
                iconHtml = '<i data-lucide="alert-triangle" class="w-6 h-6 text-yellow-600"></i>';
            } else if (type === 'error') {
                bgColor = 'bg-red-100 border-red-400';
                iconHtml = '<i data-lucide="x-octagon" class="w-6 h-6 text-red-600"></i>';
            }

            box.classList.add(bgColor, 'border', 'flex', 'items-center', 'space-x-3', 'translate-y-0');
            iconEl.innerHTML = iconHtml;
            textEl.textContent = message;

            // Render ulang ikon Lucide
            lucide.createIcons();

            // Sembunyikan setelah 5 detik
            setTimeout(() => {
                box.classList.remove('translate-y-0');
                box.classList.add('translate-y-20');
                setTimeout(() => {
                    box.classList.remove('flex', 'items-center', 'space-x-3');
                }, 300);
            }, 5000);
        }

        // --- Inisialisasi Utama ---
        window.onload = function () {
            initializeFirebase();
            startChiliAnimation();
        };

    </script>

    <!-- Message Box (Pengganti Alert/Confirm) -->
    <div id="message-box" class="fixed bottom-4 right-4 p-4 rounded-xl shadow-2xl z-50 transform translate-y-20 transition-all duration-300">
        <div id="message-icon"></div>
        <div id="message-text" class="text-sm font-medium"></div>
    </div>


    <!-- HEADER & HERO SECTION -->
    <header class="bg-rose-50 border-b border-rose-100 shadow-md sticky top-0 z-20">
        <div class="container mx-auto p-4 flex justify-between items-center max-w-7xl">
            <h1 class="text-3xl font-fredoka text-red-600 tracking-wider">Flaisy Snack</h1>
            <a href="#order-section" class="px-4 py-2 bg-red-600 text-white rounded-full font-semibold shadow-lg transition duration-300 hover:bg-red-700 hover:shadow-xl text-sm md:text-base">
                <i data-lucide="shopping-cart" class="w-4 h-4 inline mr-1"></i> Pesan Sekarang!
            </a>
        </div>
    </header>

    <main class="container mx-auto p-4 md:p-8 max-w-7xl">

        <!-- HEADLINE & TAGLINE -->
        <section class="text-center py-16 bg-rose-100 rounded-3xl shadow-xl mb-16 relative overflow-hidden">
            <!-- Overlay untuk membuat teks lebih menonjol dari animasi cabai -->
            <div class="absolute inset-0 bg-white bg-opacity-80 backdrop-blur-sm z-10 p-8 rounded-3xl">
                <h2 class="text-5xl md:text-7xl font-fredoka text-red-700 mb-4 animate-pulse">
                    Pedas Nikmat yang bikin ketagihan!
                </h2>
                <p class="text-xl md:text-2xl font-inter text-stone-600 italic">
                    "Cemilan renyah, pedas bikin nangih!"
                </p>
            </div>
            <!-- Gambar pendukung di latar belakang -->
            <img src="https://placehold.co/1200x400/FEE2E2/EF4444?text=Background+Aesthetic+Spicy" alt="Latar belakang estetik rasa pedas" class="absolute inset-0 w-full h-full object-cover opacity-30">
        </section>

        <!-- PRODUK -->
        <section id="product-section" class="mb-16">
            <h3 class="text-4xl font-fredoka text-center text-stone-800 mb-10">Produk Unggulan</h3>
            <div class="grid grid-cols-1 md:grid-cols-2 gap-8">

                <!-- Kartu Produk: Basreng Pedas -->
                <div class="bg-white p-6 rounded-3xl shadow-2xl transition duration-500 hover:scale-[1.02] border-4 border-rose-200">
                    <img src="https://o-cdf.oramiland.com/unsafe/cnc-magazine.oramiland.com/parenting/original_images/Screen_Shot_2024-05-24_at_18.47.39.png" onerror="this.onerror=null;this.src='https://placehold.co/600x400/EF4444/FFFFFF?text=Basreng+Pedas';" alt="Basreng Pedas Flaisy Snack" class="w-full h-64 object-cover rounded-2xl mb-4 shadow-inner">
                    <h4 class="text-3xl font-fredoka text-red-600 mb-2">Basreng Pedas</h4>
                    <p class="text-2xl font-bold text-red-700 mb-4">Rp 5.000 / pack</p>
                    <p class="text-stone-700 leading-relaxed">
                        Basreng (Bakso Goreng) renyah dengan bumbu cabai rahasia Flaisy Snack! 
                        Teksturnya yang kriuk di luar dan lembut di dalam, dipadukan dengan level pedas yang pas, 
                        dijamin bikin Anda tak bisa berhenti ngemil. *Waspada ketagihan!*
                    </p>
                    <a href="#order-section" class="mt-4 inline-block text-red-600 font-semibold hover:text-red-800">
                        <i data-lucide="chevrons-right" class="w-4 h-4 inline"></i> Mau Coba?
                    </a>
                </div>

                <!-- Kartu Produk: Makaroni Pedas -->
                <div class="bg-white p-6 rounded-3xl shadow-2xl transition duration-500 hover:scale-[1.02] border-4 border-rose-200">
                    <img src="https://down-id.img.susercontent.com/file/85682004a253e70891e328dd4e103c46" onerror="this.onerror=null;this.src='https://placehold.co/600x400/EF4444/FFFFFF?text=Makaroni+Pedas';" alt="Makaroni Pedas Flaisy Snack" class="w-full h-64 object-cover rounded-2xl mb-4 shadow-inner">
                    <h4 class="text-3xl font-fredoka text-red-600 mb-2">Makaroni Pedas</h4>
                    <p class="text-2xl font-bold text-red-700 mb-4">Rp 5.000 / pack</p>
                    <p class="text-stone-700 leading-relaxed">
                        Makaroni spiral yang digoreng hingga garing sempurna, lalu dibalut bumbu pedas manis yang nagih. 
                        Sensasi renyah di setiap gigitan dan rasa pedas yang membuat mata melek. Sempurna untuk teman begadang!
                    </p>
                    <a href="#order-section" class="mt-4 inline-block text-red-600 font-semibold hover:text-red-800">
                        <i data-lucide="chevrons-right" class="w-4 h-4 inline"></i> Langsung Order!
                    </a>
                </div>

            </div>
        </section>

        <!-- FORM PEMESANAN & PEMBAYARAN -->
        <section id="order-section" class="mb-16 bg-rose-50 p-6 md:p-10 rounded-3xl shadow-2xl border border-rose-200">
            <h3 class="text-4xl font-fredoka text-center text-stone-800 mb-8">Pesan dan Bayar</h3>
            
            <form id="order-form" class="grid grid-cols-1 lg:grid-cols-2 gap-8">
                <!-- Kolom Kiri: Data Pembeli & Pesanan -->
                <div>
                    <h4 class="text-2xl font-bold text-stone-700 mb-4 flex items-center"><i data-lucide="user" class="w-5 h-5 mr-2 text-red-600"></i> Data Pembeli & Pesanan</h4>
                    
                    <!-- Input Nama -->
                    <div class="mb-4">
                        <label for="customer-name" class="block text-sm font-medium text-stone-600 mb-1">Nama Lengkap</label>
                        <input type="text" id="customer-name" required class="w-full p-3 border border-rose-300 rounded-xl focus:ring-red-500 focus:border-red-500 transition duration-150">
                    </div>

                    <!-- Input Alamat -->
                    <div class="mb-6">
                        <label for="customer-address" class="block text-sm font-medium text-stone-600 mb-1">Alamat Pengiriman Lengkap</label>
                        <textarea id="customer-address" required rows="3" class="w-full p-3 border border-rose-300 rounded-xl focus:ring-red-500 focus:border-red-500 transition duration-150"></textarea>
                    </div>

                    <!-- Pilihan Pesanan -->
                    <h5 class="text-lg font-semibold text-stone-700 mb-3 flex items-center"><i data-lucide="package" class="w-4 h-4 mr-2 text-red-600"></i> Pilih Produk</h5>
                    
                    <!-- Basreng Qty -->
                    <div class="flex justify-between items-center mb-4 p-4 bg-white rounded-xl shadow-md border border-rose-100">
                        <span class="font-medium">Basreng Pedas (Rp 5.000)</span>
                        <input type="number" id="qty-basreng" value="0" min="0" class="w-20 p-2 text-center border border-rose-300 rounded-lg focus:ring-red-500 focus:border-red-500">
                    </div>

                    <!-- Makaroni Qty -->
                    <div class="flex justify-between items-center mb-6 p-4 bg-white rounded-xl shadow-md border border-rose-100">
                        <span class="font-medium">Makaroni Pedas (Rp 5.000)</span>
                        <input type="number" id="qty-makaroni" value="0" min="0" class="w-20 p-2 text-center border border-rose-300 rounded-lg focus:ring-red-500 focus:border-red-500">
                    </div>

                    <!-- Ringkasan Total -->
                    <div class="border-t-2 border-dashed border-red-300 pt-4 mt-4 bg-red-50 p-4 rounded-xl">
                        <p class="flex justify-between font-bold text-lg text-stone-800">
                            <span>Total Jumlah Pack:</span>
                            <span id="total-qty" class="text-red-600">0</span>
                        </p>
                        <p class="flex justify-between font-bold text-2xl text-stone-800 mt-2">
                            <span>Total Biaya:</span>
                            <span id="total-price" class="text-red-700">Rp 0</span>
                        </p>
                        <input type="hidden" id="grand-total-input" value="0">
                    </div>
                </div>

                <!-- Kolom Kanan: Metode Pembayaran -->
                <div>
                    <h4 class="text-2xl font-bold text-stone-700 mb-4 flex items-center"><i data-lucide="wallet" class="w-5 h-5 mr-2 text-red-600"></i> Pilih Metode Pembayaran</h4>
                    
                    <div class="space-y-4">
                        <!-- Grup Pembayaran -->
                        <div class="bg-white p-4 rounded-xl shadow-lg border border-rose-100">
                            <h5 class="font-semibold text-stone-700 mb-3">E-Wallet / QRIS (Scan untuk Pembayaran)</h5>
                            
                            <!-- Ikon E-Wallet yang RAPIH dan Estetik -->
                            <div class="grid grid-cols-5 gap-3 mb-4">
                                <!-- Menggunakan placeholder yang estetik -->
                                <img src="https://placehold.co/80x40/99f6e4/134e4a?text=DANA" alt="Logo DANA" class="w-full h-auto rounded-lg object-contain border border-teal-200 p-1">
                                <img src="https://placehold.co/80x40/a7f3d0/065f46?text=GoPay" alt="Logo GoPay" class="w-full h-auto rounded-lg object-contain border border-emerald-200 p-1">
                                <img src="https://placehold.co/80x40/fbcfe8/9d174d?text=OVO" alt="Logo OVO" class="w-full h-auto rounded-lg object-contain border border-pink-200 p-1">
                                <img src="https://placehold.co/80x40/ffe4e6/be123c?text=ShopeePay" alt="Logo ShopeePay" class="w-full h-auto rounded-lg object-contain border border-rose-200 p-1">
                                <img src="https://placehold.co/80x40/bae6fd/0369a1?text=QRIS" alt="Logo QRIS" class="w-full h-auto rounded-lg object-contain border border-sky-200 p-1">
                            </div>

                            <label class="flex items-center space-x-3 cursor-pointer p-2 rounded-lg hover:bg-rose-100 transition duration-150">
                                <input type="radio" name="payment-method" value="E-Wallet/QRIS" class="form-radio text-red-600 focus:ring-red-500">
                                <span class="text-sm font-medium">Bayar via E-Wallet (DANA, GoPay, OVO, ShopeePay) / QRIS</span>
                            </label>
                        </div>

                        <!-- Opsi COD -->
                        <div class="bg-white p-4 rounded-xl shadow-lg border border-rose-100">
                            <h5 class="font-semibold text-stone-700 mb-3">Cash / Bayar di Tempat</h5>
                             <!-- Ikon COD -->
                            <div class="grid grid-cols-5 gap-3 mb-4">
                                <img src="https://placehold.co/80x40/ddd6fe/4c1d95?text=COD" alt="Logo COD" class="w-full h-auto rounded-lg object-contain border border-violet-200 p-1 col-span-1">
                            </div>
                            <label class="flex items-center space-x-3 cursor-pointer p-2 rounded-lg hover:bg-rose-100 transition duration-150">
                                <input type="radio" name="payment-method" value="Cash On Delivery (COD) / Bayar di Tempat" class="form-radio text-red-600 focus:ring-red-500">
                                <span class="text-sm font-medium">Cash On Delivery (COD) / Bayar di Tempat</span>
                            </label>
                        </div>
                    </div>
                    
                    <!-- Tombol Call-to-Action -->
                    <button type="submit" id="order-submit-button" disabled class="w-full mt-8 p-4 bg-red-600 text-white text-xl font-bold rounded-xl shadow-lg transition duration-300 opacity-50 cursor-not-allowed">
                        <i data-lucide="send" class="w-5 h-5 inline mr-2"></i> Kirim Pesanan ke WhatsApp Admin
                    </button>
                    <p class="text-center text-sm text-stone-500 mt-2">Anda akan diarahkan ke WhatsApp untuk konfirmasi pembelian.</p>
                </div>
            </form>
        </section>


        <!-- TESTIMONIALS / KOMENTAR PELANGGAN -->
        <section id="testimonials-section" class="mb-16 bg-white p-6 md:p-10 rounded-3xl shadow-2xl border border-rose-200">
            <h3 class="text-4xl font-fredoka text-center text-stone-800 mb-8">Apa Kata Pelanggan Kami?</h3>

            <div class="grid grid-cols-1 md:grid-cols-3 gap-8">
                <!-- Kolom Kiri: Form Komentar -->
                <div class="md:col-span-1 p-6 bg-rose-50 rounded-xl shadow-inner border border-rose-100">
                    <h4 class="text-xl font-bold text-red-600 mb-4">Bagikan Pengalaman Anda!</h4>
                    <form id="comment-form">
                        <div class="mb-4">
                            <label for="comment-name" class="block text-sm font-medium text-stone-600 mb-1">Nama (Opsional)</label>
                            <input type="text" id="comment-name" class="w-full p-2 border border-rose-300 rounded-lg">
                        </div>
                        <div class="mb-4">
                            <label for="comment-text" class="block text-sm font-medium text-stone-600 mb-1">Komentar/Testimoni Anda</label>
                            <textarea id="comment-text" required rows="4" class="w-full p-2 border border-rose-300 rounded-lg focus:ring-red-500 focus:border-red-500"></textarea>
                        </div>
                        <button type="submit" class="w-full p-2 bg-red-500 text-white rounded-lg font-semibold hover:bg-red-600 transition duration-150">Kirim Komentar</button>
                    </form>
                </div>

                <!-- Kolom Kanan: Daftar Komentar -->
                <div class="md:col-span-2">
                    <h4 class="text-xl font-bold text-stone-700 mb-4">Kumpulan Komentar</h4>
                    <div id="testimonials-list" class="space-y-4 max-h-96 overflow-y-auto pr-2">
                        <!-- Komentar akan dimuat di sini oleh JavaScript (onSnapshot) -->
                        <p class="text-center text-stone-500 italic">Memuat data komentar...</p>
                    </div>
                </div>
            </div>
        </section>

    </main>

    <!-- FOOTER -->
    <footer class="bg-stone-800 p-8 text-white mt-16">
        <div class="container mx-auto max-w-7xl text-center">
            <p class="text-4xl font-fredoka text-rose-300 mb-4">
                Flaisy Snack — Teman Setia Waktu Santai!
            </p>
            <div class="border-t border-stone-700 pt-4 mt-4">
                <!-- Creator Text -->
                <p class="text-xs text-stone-400 mb-2">
                    Di buat Oleh Safira Dwi Natasya Dopas, Nur Aisyah, Bunga Azzahra_XII-I
                </p>
                <!-- Hak Cipta -->
                <p class="text-sm text-stone-500">&copy; 2024 Flaisy Snack. All rights reserved.</p>
            </div>
        </div>
    </footer>

    <!-- Inisialisasi Ikon Lucide -->
    <script>
        document.addEventListener('DOMContentLoaded', () => {
            lucide.createIcons();
        });
    </script>
</body>
</html>
