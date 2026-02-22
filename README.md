# VANTAGE
بع منتجاتك الرقمية بسهولة
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Vantage Ultimate | الذكاء الاصطناعي الرقمي</title>
    
    <!-- Scripts & Styles -->
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Cairo:wght@400;600;700;900&display=swap" rel="stylesheet">
    <script src="https://unpkg.com/lucide@latest"></script>
    
    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.0.1/firebase-app.js";
        import { getAuth, onAuthStateChanged, signInAnonymously, signOut, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.0.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, setDoc, updateDoc, collection, onSnapshot, query, addDoc, serverTimestamp, runTransaction } from "https://www.gstatic.com/firebasejs/11.0.1/firebase-firestore.js";

        // Gemini API Setup
        const apiKey = ""; // سيتم توفيره من البيئة المحيطة

        const firebaseConfig = JSON.parse(__firebase_config);
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'vantage-ai-99';

        // Global State
        let currentUser = null;
        let userData = { balance: 0, totalSales: 0, referrals: 0 };

        // --- Gemini AI Functions ---
        window.getAIAdvice = async () => {
            const btn = document.getElementById('ai-advice-btn');
            const originalText = btn.innerHTML;
            btn.innerHTML = '<div class="w-4 h-4 border-2 border-white border-t-transparent animate-spin rounded-full"></div> جاري التحليل...';
            btn.disabled = true;

            const prompt = `أنا بائع في منصة Vantage الرقمية. إجمالي مبيعاتي حالياً هو ${userData.totalSales} USDT. رصيدي المتاح هو ${userData.balance} USDT. قدم لي نصيحة واحدة قصيرة واحترافية (باللغة العربية) لزيادة أرباحي وجذب عملاء أكثر في جملة واحدة فقط.`;

            try {
                const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        contents: [{ parts: [{ text: prompt }] }]
                    })
                });
                
                const data = await response.json();
                const advice = data.candidates?.[0]?.content?.parts?.[0]?.text || "استمر في تقديم محتوى عالي الجودة لزيادة ثقة العملاء.";
                
                const adviceEl = document.getElementById('ai-advice-box');
                adviceEl.innerHTML = `<div class="bg-yellow-400/10 border border-yellow-400/20 p-4 rounded-2xl text-yellow-600 text-xs font-bold animate-in fade-in slide-in-from-top-2">✨ نصيحة الذكاء الاصطناعي: ${advice}</div>`;
            } catch (error) {
                console.error("AI Error:", error);
            } finally {
                btn.innerHTML = originalText;
                btn.disabled = false;
            }
        };

        window.generateAIDescription = async () => {
            const name = document.getElementById('prod-name').value;
            if (!name) return alert("يرجى إدخال اسم المنتج أولاً");
            
            const btn = document.getElementById('ai-desc-btn');
            btn.disabled = true;
            btn.innerText = "✨ توليد وصف...";

            const prompt = `اكتب وصفاً تسويقياً قصيراً جداً (حد أقصى 15 كلمة) لمنتج رقمي اسمه: "${name}". الوصف يجب أن يكون جذاباً ومحفزاً للشراء باللغة العربية.`;

            try {
                const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        contents: [{ parts: [{ text: prompt }] }]
                    })
                });
                const data = await response.json();
                const desc = data.candidates?.[0]?.content?.parts?.[0]?.text || "منتج رقمي مميز متاح الآن.";
                document.getElementById('prod-desc').value = desc;
            } catch (err) {
                console.error(err);
            } finally {
                btn.disabled = false;
                btn.innerText = "✨ وصف ذكي";
            }
        };

        // --- Auth Logic ---
        window.initAuth = async () => {
            if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                await signInWithCustomToken(auth, __initial_auth_token);
            } else {
                await signInAnonymously(auth);
            }

            onAuthStateChanged(auth, async (user) => {
                if (user) {
                    currentUser = user;
                    await checkUserRecord(user);
                    listenToData(user.uid);
                    document.getElementById('auth-overlay').classList.add('hidden');
                }
            });
        };

        async function checkUserRecord(user) {
            const userRef = doc(db, 'artifacts', appId, 'users', user.uid, 'profile', 'data');
            const snap = await getDoc(userRef);
            if (!snap.exists()) {
                await setDoc(userRef, {
                    balance: 0,
                    totalSales: 0,
                    referrals: 0,
                    uid: user.uid,
                    refCode: 'VNTG-' + Math.random().toString(36).substring(7).toUpperCase()
                });
            }
        }

        function listenToData(uid) {
            onSnapshot(doc(db, 'artifacts', appId, 'users', uid, 'profile', 'data'), (doc) => {
                if (doc.exists()) {
                    userData = doc.data();
                    renderUI();
                }
            });
            
            const q = query(collection(db, 'artifacts', appId, 'public', 'data', 'products'));
            onSnapshot(q, (snapshot) => {
                const products = snapshot.docs.map(d => ({id: d.id, ...d.data()}));
                renderMarket(products);
            });
        }

        // --- Core Functions ---
        window.publishProduct = async (e) => {
            e.preventDefault();
            const btn = e.target.querySelector('button[type="submit"]');
            btn.disabled = true;
            btn.innerText = "جاري التأمين والرفع...";

            const formData = new FormData(e.target);
            try {
                await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'products'), {
                    name: formData.get('name'),
                    desc: formData.get('desc'),
                    price: parseFloat(formData.get('price')),
                    category: formData.get('category'),
                    sellerId: currentUser.uid,
                    verified: true,
                    timestamp: serverTimestamp()
                });
                alert("تم النشر بنجاح!");
                switchTab('market');
            } catch (err) {
                alert("خطأ في الاتصال");
            } finally {
                btn.disabled = false;
                btn.innerText = "نشر المنتج الآن";
            }
        };

        window.buyProduct = async (productId, price, sellerId) => {
            if (userData.balance < price) {
                alert("رصيد غير كافٍ");
                switchTab('wallet');
                return;
            }

            try {
                await runTransaction(db, async (transaction) => {
                    const buyerRef = doc(db, 'artifacts', appId, 'users', currentUser.uid, 'profile', 'data');
                    const sellerRef = doc(db, 'artifacts', appId, 'users', sellerId, 'profile', 'data');
                    const sellerNet = price * 0.80;
                    transaction.update(buyerRef, { balance: userData.balance - price });
                    const sellerSnap = await transaction.get(sellerRef);
                    const sData = sellerSnap.data() || { balance: 0, totalSales: 0 };
                    transaction.set(sellerRef, { 
                        balance: (sData.balance || 0) + sellerNet,
                        totalSales: (sData.totalSales || 0) + sellerNet
                    }, { merge: true });
                });
                alert("نجحت عملية الشراء!");
            } catch (e) {
                alert("فشلت العملية");
            }
        };

        window.simulateBinanceDeposit = async () => {
            const userRef = doc(db, 'artifacts', appId, 'users', currentUser.uid, 'profile', 'data');
            await updateDoc(userRef, { balance: (userData.balance || 0) + 100 });
            alert("تم إيداع 100 USDT عبر Binance Pay");
            closeModal();
        };

        function renderUI() {
            document.querySelectorAll('.balance-val').forEach(el => el.innerText = (userData.balance || 0).toFixed(2));
            document.getElementById('ref-code-display').innerText = userData.refCode || '...';
            document.getElementById('profit-net').innerText = (userData.totalSales || 0).toFixed(2);
        }

        function renderMarket(products) {
            const container = document.getElementById('market-grid');
            if (products.length === 0) {
                container.innerHTML = `<div class="col-span-full py-20 text-center opacity-40 font-bold">لا توجد منتجات</div>`;
                return;
            }
            container.innerHTML = products.map(p => `
                <div class="bg-white p-5 rounded-[2.2rem] shadow-sm border border-slate-50 flex flex-col gap-4">
                    <div class="flex items-center justify-between">
                        <div class="flex items-center gap-3">
                            <div class="w-12 h-12 bg-slate-50 rounded-2xl flex items-center justify-center text-xl">
                                ${p.category === 'book' ? '📚' : (p.category === 'course' ? '🎓' : '💻')}
                            </div>
                            <div>
                                <h4 class="font-bold text-slate-800 text-sm">${p.name}</h4>
                                <p class="text-[9px] text-slate-400 font-bold uppercase tracking-widest">${p.category}</p>
                            </div>
                        </div>
                        <span class="text-sm font-black text-slate-900">${p.price} <span class="text-[8px] text-yellow-600">USDT</span></span>
                    </div>
                    ${p.desc ? `<p class="text-[11px] text-slate-500 bg-slate-50 p-3 rounded-xl line-clamp-2 italic border border-slate-100/50">✨ ${p.desc}</p>` : ''}
                    <button onclick="buyProduct('${p.id}', ${p.price}, '${p.sellerId}')" class="w-full bg-slate-900 text-white py-3 rounded-2xl text-[11px] font-black hover:bg-yellow-400 hover:text-black transition-all">
                        شراء الآن
                    </button>
                </div>
            `).join('');
        }

        window.addEventListener('DOMContentLoaded', () => {
            initAuth();
            lucide.createIcons();
        });
    </script>

    <style>
        body { font-family: 'Cairo', sans-serif; background: #fcfcfd; user-select: none; -webkit-tap-highlight-color: transparent; }
        .tab-content { display: none; }
        .tab-content.active { display: block; animation: fadeInUp 0.4s ease-out; }
        @keyframes fadeInUp { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
        .sidebar { transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1); }
        input, select, textarea { outline: none; border: 2px solid transparent; background: #f8fafc; }
        input:focus, textarea:focus { border-color: #fbbf24; background: white; }
    </style>
</head>
<body class="pb-28">

    <!-- Auth Overlay -->
    <div id="auth-overlay" class="fixed inset-0 bg-slate-900 z-[200] flex flex-col items-center justify-center p-10 text-center">
        <div class="w-20 h-20 bg-yellow-400 rounded-3xl mb-6 shadow-2xl flex items-center justify-center rotate-12">
            <i data-lucide="shield-check" class="w-10 h-10 text-slate-900"></i>
        </div>
        <h1 class="text-3xl font-black text-white mb-2 italic tracking-tighter">VANTAGE</h1>
        <p class="text-slate-400 text-xs mb-8">جاري ربط نظام الذكاء الاصطناعي مع بوابة الدفع...</p>
        <div class="w-10 h-10 border-4 border-yellow-400/20 border-t-yellow-400 rounded-full animate-spin"></div>
    </div>

    <!-- Sidebar -->
    <div id="sidebar-overlay" class="fixed inset-0 bg-black/40 z-[100] hidden" onclick="toggleSidebar()"></div>
    <aside id="sidebar" class="fixed top-0 right-0 h-full w-72 bg-white z-[110] shadow-2xl sidebar translate-x-full">
        <div class="p-8 border-b flex justify-between items-center bg-slate-50/50">
            <span class="font-black text-xl italic text-yellow-500">VANTAGE</span>
            <button onclick="toggleSidebar()"><i data-lucide="x" class="text-slate-300"></i></button>
        </div>
        <div class="p-8 space-y-8">
            <div class="space-y-4">
                <h4 class="text-[10px] font-black text-slate-300 uppercase tracking-widest">التواصل والتحقق</h4>
                <a href="#" class="flex items-center gap-3 text-sm font-bold text-slate-600"><i data-lucide="instagram" class="w-4 h-4"></i> إنستقرام</a>
                <a href="#" class="flex items-center gap-3 text-sm font-bold text-slate-600"><i data-lucide="twitter" class="w-4 h-4"></i> إكس (تويتر)</a>
                <a href="#" class="flex items-center gap-3 text-sm font-bold text-slate-600"><i data-lucide="send" class="w-4 h-4"></i> قناة التليجرام</a>
            </div>
            <div class="p-5 bg-yellow-50 rounded-2xl border border-yellow-100">
                <p class="text-[10px] font-black text-yellow-700 uppercase mb-2">حالة الحساب ✨</p>
                <p class="text-[11px] text-yellow-600 leading-relaxed font-bold">حسابك مدعوم بميزات الذكاء الاصطناعي من فئة Enterprise.</p>
            </div>
        </div>
    </aside>

    <!-- Header -->
    <header class="bg-white/90 sticky top-0 z-50 px-6 py-4 border-b flex justify-between items-center backdrop-blur-md">
        <button onclick="toggleSidebar()" class="p-2.5 bg-slate-50 rounded-xl">
            <i data-lucide="menu" class="w-5 h-5 text-slate-700"></i>
        </button>
        <h1 class="text-lg font-black italic tracking-tighter">VANTAGE <span class="text-yellow-500 font-normal">AI</span></h1>
        <div class="bg-slate-900 px-3 py-1.5 rounded-full flex items-center gap-2">
            <i data-lucide="zap" class="w-3 h-3 text-yellow-400 fill-yellow-400"></i>
            <span class="text-[10px] font-black text-white balance-val">0.00</span>
        </div>
    </header>

    <!-- Main Content -->
    <main class="max-w-xl mx-auto p-5">
        
        <!-- Tab: Market -->
        <section id="market" class="tab-content active space-y-6">
            <div class="flex justify-between items-end">
                <div>
                    <h2 class="text-2xl font-black">المتجر الذكي</h2>
                    <p class="text-[10px] font-bold text-slate-400 uppercase tracking-widest mt-1">منتجات رقمية عالمية</p>
                </div>
                <div class="flex gap-2">
                    <div class="w-8 h-8 rounded-full bg-slate-100 flex items-center justify-center"><i data-lucide="search" class="w-4 h-4 text-slate-400"></i></div>
                    <div class="w-8 h-8 rounded-full bg-slate-100 flex items-center justify-center"><i data-lucide="filter" class="w-4 h-4 text-slate-400"></i></div>
                </div>
            </div>
            <div id="market-grid" class="grid gap-4">
                <!-- Products -->
            </div>
        </section>

        <!-- Tab: Add Product -->
        <section id="add" class="tab-content space-y-6">
            <div class="flex items-center gap-2">
                <h2 class="text-2xl font-black">نشر منتج جديد</h2>
                <span class="bg-indigo-100 text-indigo-600 text-[9px] px-2 py-0.5 rounded-md font-black">AI ENABLED ✨</span>
            </div>
            <form onsubmit="publishProduct(event)" class="bg-white p-6 rounded-[2.5rem] shadow-xl space-y-5 border border-slate-50">
                <div class="space-y-4">
                    <div class="relative">
                        <input id="prod-name" name="name" required placeholder="اسم المنتج الرقمي" class="w-full p-4 rounded-2xl font-bold text-sm">
                        <button type="button" id="ai-desc-btn" onclick="generateAIDescription()" class="absolute left-2 top-2 bottom-2 bg-yellow-400 text-slate-900 px-3 rounded-xl text-[9px] font-black shadow-sm">✨ وصف ذكي</button>
                    </div>
                    
                    <textarea id="prod-desc" name="desc" placeholder="وصف المنتج (يمكنك استخدام الذكاء الاصطناعي أعلاه لإنشائه)" class="w-full p-4 rounded-2xl font-bold text-sm h-24 resize-none"></textarea>

                    <div class="flex gap-3">
                        <input name="price" type="number" step="0.01" required placeholder="السعر" class="flex-1 p-4 rounded-2xl font-bold text-sm">
                        <select name="category" class="bg-slate-50 p-4 rounded-2xl font-bold text-sm text-slate-500">
                            <option value="book">كتاب</option>
                            <option value="course">دورة</option>
                            <option value="code">كود</option>
                        </select>
                    </div>
                </div>
                <div class="border-2 border-dashed border-slate-100 p-8 rounded-3xl text-center bg-slate-50/50">
                    <i data-lucide="file-code" class="w-8 h-8 mx-auto text-slate-300 mb-2"></i>
                    <p class="text-[10px] font-black text-slate-400 uppercase">ارفع الملف (ZIP, PDF, MP4)</p>
                </div>
                <button type="submit" class="w-full bg-slate-900 text-white font-black py-5 rounded-[2rem] shadow-lg active:scale-95 transition-all">نشر المنتج الآن</button>
            </form>
        </section>

        <!-- Tab: Profits -->
        <section id="profits" class="tab-content space-y-6">
            <h2 class="text-2xl font-black">لوحة الأرباح</h2>
            
            <!-- AI Advice Box -->
            <div id="ai-advice-box"></div>

            <div class="grid grid-cols-2 gap-4">
                <div class="bg-white p-6 rounded-[2rem] border border-slate-100 shadow-sm">
                    <p class="text-[10px] font-black text-slate-400 uppercase mb-2">إجمالي المبيعات</p>
                    <h3 id="profit-net" class="text-2xl font-black">0.00</h3>
                </div>
                <div class="bg-white p-6 rounded-[2rem] border border-slate-100 shadow-sm flex flex-col justify-between">
                    <p class="text-[10px] font-black text-slate-400 uppercase mb-2">مستشار النمو</p>
                    <button id="ai-advice-btn" onclick="getAIAdvice()" class="bg-indigo-600 text-white py-2 px-4 rounded-xl text-[10px] font-black flex items-center justify-center gap-2">✨ نصيحة ذكية</button>
                </div>
            </div>
            
            <div class="bg-gradient-to-br from-slate-900 to-slate-800 text-white p-8 rounded-[2.5rem] relative overflow-hidden shadow-2xl">
                <div class="relative z-10">
                    <p class="text-xs font-bold text-slate-400 mb-2">أرباحك المتاحة (80% من المبيعات)</p>
                    <h2 class="text-4xl font-black mb-8"><span class="balance-val">0.00</span> <span class="text-yellow-400 text-lg">USDT</span></h2>
                    <div class="flex gap-2">
                        <button class="flex-1 bg-white text-slate-900 font-black py-4 rounded-2xl text-xs">سحب الرصيد</button>
                        <button class="w-12 h-12 bg-white/10 rounded-2xl flex items-center justify-center border border-white/20"><i data-lucide="history" class="w-5 h-5"></i></button>
                    </div>
                </div>
                <i data-lucide="trending-up" class="absolute -bottom-10 -left-10 w-48 h-48 opacity-10 rotate-12"></i>
            </div>
        </section>

        <!-- Tab: Wallet -->
        <section id="wallet" class="tab-content space-y-6">
            <div class="bg-white p-10 rounded-[3rem] shadow-xl text-center border border-slate-50">
                <div class="w-20 h-20 bg-yellow-400 rounded-[2rem] flex items-center justify-center mx-auto mb-6 shadow-xl rotate-6">
                    <i data-lucide="bitcoin" class="text-slate-900 w-10 h-10"></i>
                </div>
                <p class="text-xs font-black text-slate-400 uppercase tracking-widest">رصيد الشراء</p>
                <h2 class="text-5xl font-black text-slate-900 mt-2 mb-8 balance-val">0.00</h2>
                <button onclick="openModal()" class="w-full bg-slate-900 text-white font-black py-5 rounded-3xl flex items-center justify-center gap-3">
                    شحن المحفظة <i data-lucide="arrow-right" class="w-4 h-4"></i>
                </button>
            </div>
            
            <div class="bg-indigo-600 p-8 rounded-[2.5rem] text-white relative overflow-hidden">
                <div class="relative z-10">
                    <div class="flex items-center gap-3 mb-4">
                        <i data-lucide="gift" class="text-indigo-200"></i>
                        <h3 class="font-black text-lg">شارك واربح</h3>
                    </div>
                    <div class="bg-white/10 p-5 rounded-2xl border border-white/20 flex justify-between items-center backdrop-blur-md">
                        <span id="ref-code-display" class="font-mono font-black text-xl tracking-tighter uppercase">...</span>
                        <button class="text-[10px] bg-white text-indigo-600 px-4 py-2 rounded-xl font-black uppercase">نسخ</button>
                    </div>
                </div>
                <div class="absolute -right-10 -bottom-10 w-40 h-40 bg-indigo-500 rounded-full blur-3xl opacity-50"></div>
            </div>
        </section>
    </main>

    <!-- Navigation -->
    <nav class="fixed bottom-6 inset-x-6 bg-slate-900/90 backdrop-blur-xl px-4 py-3 flex justify-around items-center z-[90] rounded-[2.5rem] shadow-2xl border border-white/10">
        <button onclick="switchTab('market')" class="nav-btn active flex flex-col items-center gap-1.5 text-slate-500 transition-all p-2">
            <i data-lucide="shopping-bag" class="w-5 h-5"></i>
            <span class="text-[9px] font-black">المتجر</span>
        </button>
        <button onclick="switchTab('add')" class="nav-btn flex flex-col items-center gap-1.5 text-slate-500 transition-all p-2">
            <i data-lucide="plus-circle" class="w-5 h-5"></i>
            <span class="text-[9px] font-black">نشر</span>
        </button>
        <button onclick="switchTab('profits')" class="nav-btn flex flex-col items-center gap-1.5 text-slate-500 transition-all p-2">
            <i data-lucide="bar-chart-3" class="w-5 h-5"></i>
            <span class="text-[9px] font-black">الأرباح</span>
        </button>
        <button onclick="switchTab('wallet')" class="nav-btn flex flex-col items-center gap-1.5 text-slate-500 transition-all p-2">
            <i data-lucide="wallet-2" class="w-5 h-5"></i>
            <span class="text-[9px] font-black">المحفظة</span>
        </button>
    </nav>

    <!-- Binance Modal -->
    <div id="deposit-modal" class="fixed inset-0 bg-slate-900/60 z-[150] hidden items-center justify-center p-6 backdrop-blur-sm">
        <div class="bg-white w-full max-w-sm rounded-[3rem] p-8 text-center space-y-6 relative animate-in zoom-in-95">
            <button onclick="closeModal()" class="absolute top-6 left-6 text-slate-300"><i data-lucide="x"></i></button>
            <div class="w-16 h-16 bg-yellow-50 rounded-2xl flex items-center justify-center mx-auto shadow-sm">
                <i data-lucide="lock" class="text-yellow-500 w-8 h-8"></i>
            </div>
            <h3 class="text-xl font-black">Binance API Bridge</h3>
            <div class="bg-slate-50 p-6 rounded-3xl border border-slate-100">
                <p class="text-[10px] text-slate-400 mb-2 font-bold uppercase tracking-widest">معرف المعاملة الآمن</p>
                <code class="text-[11px] font-mono break-all font-black text-slate-600">VNTG-PAY-NODE-881-AI-SECURE</code>
            </div>
            <button onclick="simulateBinanceDeposit()" class="w-full bg-slate-900 text-white py-5 rounded-3xl font-black shadow-xl">تأكيد شحن 100 USDT</button>
        </div>
    </div>

    <script>
        function switchTab(id) {
            document.querySelectorAll('.tab-content').forEach(t => t.classList.remove('active'));
            document.getElementById(id).classList.add('active');
            
            document.querySelectorAll('.nav-btn').forEach(b => {
                b.classList.remove('text-yellow-400', 'scale-110');
                b.classList.add('text-slate-500');
            });
            event.currentTarget?.classList?.add('text-yellow-400', 'scale-110');
            event.currentTarget?.classList?.remove('text-slate-500');
            
            lucide.createIcons();
        }

        function toggleSidebar() {
            document.getElementById('sidebar').classList.toggle('translate-x-full');
            document.getElementById('sidebar-overlay').classList.toggle('hidden');
        }

        function openModal() { document.getElementById('deposit-modal').style.display = 'flex'; lucide.createIcons(); }
        function closeModal() { document.getElementById('deposit-modal').style.display = 'none'; }
    </script>

</body>
</html>
