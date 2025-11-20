import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInAnonymously, 
  onAuthStateChanged, 
  signInWithCustomToken 
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  doc, 
  setDoc, 
  onSnapshot 
} from 'firebase/firestore';
import { 
  Users, 
  Instagram, 
  Clock, 
  AlertTriangle, 
  CheckCircle, 
  Lock, 
  Trophy 
} from 'lucide-react';

// --- Firebase Başlatma ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

export default function ZeronScrim() {
  const [user, setUser] = useState(null);
  const [slots, setSlots] = useState({}); // Dolu slotları tutar
  const [mySlot, setMySlot] = useState(null); // Benim aldığım slot numarası
  const [modalOpen, setModalOpen] = useState(false);
  const [selectedSlot, setSelectedSlot] = useState(null);
  const [teamName, setTeamName] = useState('');
  const [instagram, setInstagram] = useState('');
  const [loading, setLoading] = useState(false);
  const [timeUntilReset, setTimeUntilReset] = useState('');

  // Türkiye saatine göre bugünün tarihini al (Gün/Ay/Yıl)
  const getTodayDateString = () => {
    return new Date().toLocaleDateString('tr-TR');
  };

  // --- 1. Adım: Kimlik Doğrulama ---
  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    const unsub = onAuthStateChanged(auth, (u) => setUser(u));
    return () => unsub();
  }, []);

  // --- 2. Adım: Canlı Veri Dinleme (Real-Time) ---
  useEffect(() => {
    if (!user) return;

    // Veritabanı yolu: artifacts -> [appId] -> public -> data -> zeron_bookings
    // İsim "zeron" olarak güncellendi
    const bookingsRef = collection(db, 'artifacts', appId, 'public', 'data', 'zeron_bookings');

    // Snapshot listener: Veritabanında bir değişiklik olduğu an burası çalışır
    const unsubscribe = onSnapshot(bookingsRef, (snapshot) => {
      const todayStr = getTodayDateString();
      const currentSlots = {};
      let userBookedSlot = null;

      snapshot.docs.forEach((doc) => {
        const data = doc.data();
        // Sadece BUGÜNÜN tarihine sahip kayıtları göster
        if (data.date === todayStr) {
          currentSlots[data.slotId] = data;
          
          // Eğer bu kaydı yapan şu anki kullanıcıysa, slot numarasını kaydet
          if (data.userId === user.uid) {
            userBookedSlot = data.slotId;
          }
        }
      });

      setSlots(currentSlots);
      setMySlot(userBookedSlot);
    });

    return () => unsubscribe();
  }, [user]);

  // --- 3. Adım: Geri Sayım Sayacı ---
  useEffect(() => {
    const timer = setInterval(() => {
      const now = new Date();
      const tomorrow = new Date(now);
      tomorrow.setDate(tomorrow.getDate() + 1);
      tomorrow.setHours(0, 0, 0, 0);
      
      const diff = tomorrow - now;
      const h = Math.floor((diff / (1000 * 60 * 60)) % 24);
      const m = Math.floor((diff / (1000 * 60)) % 60);
      const s = Math.floor((diff / 1000) % 60);
      
      setTimeUntilReset(`${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`);
    }, 1000);
    return () => clearInterval(timer);
  }, []);

  // Slot Seçimi
  const handleSlotClick = (slotNum) => {
    // Eğer zaten doluysa işlem yapma
    if (slots[slotNum]) return;

    // Eğer kullanıcının zaten bir slotu varsa uyarı ver ve engelle
    if (mySlot) {
      return; 
    }

    setSelectedSlot(slotNum);
    setModalOpen(true);
  };

  // Rezervasyon İşlemi
  const confirmBooking = async (e) => {
    e.preventDefault();
    if (!teamName || !instagram) return;

    setLoading(true);
    try {
      const slotIdStr = selectedSlot.toString();
      // Koleksiyon ismi zeron_bookings olarak ayarlandı
      const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'zeron_bookings', slotIdStr);
      
      await setDoc(docRef, {
        slotId: selectedSlot,
        teamName: teamName.toUpperCase(),
        instagram: instagram,
        userId: user.uid,
        date: getTodayDateString(), // Kritik nokta: Bugünün tarihiyle kaydet
        timestamp: Date.now()
      });
      
      setModalOpen(false);
      setTeamName('');
      setInstagram('');
    } catch (error) {
      console.error("Hata:", error);
      alert("Bir hata oluştu, lütfen tekrar dene.");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen bg-[#050505] text-white font-sans selection:bg-red-600 selection:text-white pb-12">
      
      {/* --- NAVBAR --- */}
      <nav className="sticky top-0 z-40 bg-[#0a0a0a]/90 backdrop-blur-md border-b border-white/10 shadow-2xl shadow-red-900/5">
        <div className="max-w-6xl mx-auto px-4 h-20 flex items-center justify-between">
          <div className="flex items-center gap-3">
            <div className="w-10 h-10 bg-red-600 rounded flex items-center justify-center rotate-3 shadow-lg shadow-red-600/40">
              <Trophy size={24} className="text-white" />
            </div>
            <div>
              <h1 className="text-2xl font-black tracking-tighter italic">
                ZERON <span className="text-red-600">ORG.</span>
              </h1>
              <p className="text-[10px] text-gray-500 tracking-[0.2em] font-bold">PROFESSIONAL SCRIMS</p>
            </div>
          </div>

          {/* Sayaç - Masaüstü */}
          <div className="hidden md:flex items-center gap-3 bg-black/40 px-4 py-2 rounded-full border border-white/5">
            <Clock size={16} className="text-red-500 animate-pulse" />
            <div className="text-sm font-mono text-gray-300">
              SIFIRLANMAYA: <span className="text-white font-bold text-lg">{timeUntilReset}</span>
            </div>
          </div>
        </div>
      </nav>

      {/* --- MOBİL SAYAÇ --- */}
      <div className="md:hidden bg-red-900/20 border-b border-red-900/30 py-2 text-center">
         <span className="text-xs font-mono text-red-200">
           Sıfırlanmaya: <strong>{timeUntilReset}</strong>
         </span>
      </div>

      {/* --- İÇERİK --- */}
      <main className="max-w-6xl mx-auto px-4 mt-8">
        
        {/* Durum Mesajı */}
        {mySlot && (
          <div className="mb-8 p-4 bg-green-900/20 border border-green-500/30 rounded-xl flex items-center gap-4 animate-in slide-in-from-top-4 duration-500">
            <div className="w-10 h-10 rounded-full bg-green-500/20 flex items-center justify-center flex-shrink-0">
              <CheckCircle className="text-green-500" />
            </div>
            <div>
              <h3 className="font-bold text-green-400">Kaydınız Başarılı!</h3>
              <p className="text-sm text-green-200/60">
                Bugünlük <strong>Slot {mySlot}</strong> takımınız adına ayrıldı. Yarın tekrar kayıt olabilirsiniz.
              </p>
            </div>
          </div>
        )}

        {/* Kurallar */}
        <div className="mb-6 flex items-start gap-2 text-xs text-gray-500 bg-[#0a0a0a] p-3 rounded border border-white/5">
          <AlertTriangle size={14} className="text-red-500 mt-0.5" />
          <p>
            Her takım günde sadece <strong className="text-white">1 SLOT</strong> alabilir. 
            Gece 00:00'da tüm slotlar otomatik sıfırlanır.
          </p>
        </div>

        {/* SLOT GRID */}
        <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-5 gap-3 md:gap-4">
          {Array.from({ length: 25 }, (_, i) => i + 1).map((num) => {
            const slotInfo = slots[num]; // Bu slotun verisi
            const isTaken = !!slotInfo; // Dolu mu?
            const isMine = mySlot === num; // Benim mi?
            const isDisabled = isTaken || (mySlot !== null && !isMine); // Tıklanabilir mi?

            return (
              <button
                key={num}
                onClick={() => !isDisabled && handleSlotClick(num)}
                disabled={isDisabled}
                className={`
                  relative h-32 rounded-xl border flex flex-col items-center justify-center p-3 text-center transition-all duration-300 group overflow-hidden
                  ${isTaken 
                    ? 'bg-[#111] border-white/5 cursor-default' 
                    : isDisabled
                      ? 'bg-[#0a0a0a] border-white/5 opacity-50 cursor-not-allowed'
                      : 'bg-[#0a0a0a] border-white/10 hover:border-red-500 hover:bg-red-900/10 hover:shadow-[0_0_30px_rgba(220,38,38,0.15)] cursor-pointer'
                  }
                  ${isMine ? 'ring-2 ring-green-500 bg-green-900/10' : ''}
                `}
              >
                {/* Arkaplan Numarası */}
                <span className={`absolute top-2 left-3 text-4xl font-black opacity-[0.03] select-none ${isTaken ? 'text-white' : 'text-red-500'}`}>
                  {num < 10 ? `0${num}` : num}
                </span>

                {/* Slot Durumu */}
                {isTaken ? (
                  <div className="w-full space-y-2 z-10">
                    <div className="flex justify-center mb-1">
                      {isMine ? (
                         <span className="bg-green-600/20 text-green-400 text-[10px] px-2 py-0.5 rounded font-bold tracking-wider border border-green-600/30">
                           SİZİN SLOT
                         </span>
                      ) : (
                        <Lock size={16} className="text-gray-600" />
                      )}
                    </div>
                    <div className="font-bold text-white text-sm md:text-base truncate uppercase tracking-tight">
                      {slotInfo.teamName}
                    </div>
                    <div className="flex items-center justify-center gap-1 text-gray-500 text-xs">
                      <Instagram size={12} />
                      <span className="truncate max-w-[100px]">@{slotInfo.instagram.replace('@', '')}</span>
                    </div>
                  </div>
                ) : (
                  <>
                    <div className="absolute top-3 right-3 w-2 h-2 rounded-full bg-green-500 shadow-[0_0_10px_#22c55e] animate-pulse"></div>
                    <span className="text-gray-400 font-bold text-lg group-hover:text-white group-hover:scale-110 transition-transform">
                      SLOT {num}
                    </span>
                    <span className="text-[10px] text-gray-600 mt-1 group-hover:text-red-400">
                      ALMAK İÇİN TIKLA
                    </span>
                  </>
                )}
              </button>
            );
          })}
        </div>
      </main>

      <footer className="mt-12 text-center text-gray-600 text-xs pb-8">
        <p>ZERON ORGANİZASYON © {new Date().getFullYear()}</p>
      </footer>

      {/* --- MODAL --- */}
      {modalOpen && (
        <div className="fixed inset-0 z-50 flex items-center justify-center p-4 bg-black/90 backdrop-blur-sm animate-in fade-in duration-200">
          <div className="bg-[#0f0f0f] border border-white/10 w-full max-w-sm rounded-2xl overflow-hidden shadow-2xl">
            
            <div className="bg-red-600 px-6 py-4 flex justify-between items-center">
              <h3 className="text-white font-bold">SLOT {selectedSlot} ONAYLA</h3>
              <button onClick={() => setModalOpen(false)} className="text-white/80 hover:text-white text-xl">&times;</button>
            </div>

            <form onSubmit={confirmBooking} className="p-6 space-y-4">
              <div>
                <label className="text-xs font-bold text-gray-500 uppercase block mb-1.5">Takım İsmi</label>
                <div className="relative">
                  <Users className="absolute left-3 top-3 text-gray-500" size={18} />
                  <input 
                    type="text" 
                    required
                    maxLength={18}
                    className="w-full bg-black border border-white/10 rounded-lg py-2.5 pl-10 pr-4 text-white placeholder-gray-700 focus:outline-none focus:border-red-500 focus:ring-1 focus:ring-red-500 transition-all"
                    placeholder="ZERON ESPORTS"
                    value={teamName}
                    onChange={(e) => setTeamName(e.target.value)}
                  />
                </div>
              </div>

              <div>
                <label className="text-xs font-bold text-gray-500 uppercase block mb-1.5">Instagram Adresi</label>
                <div className="relative">
                  <Instagram className="absolute left-3 top-3 text-gray-500" size={18} />
                  <input 
                    type="text" 
                    required
                    className="w-full bg-black border border-white/10 rounded-lg py-2.5 pl-10 pr-4 text-white placeholder-gray-700 focus:outline-none focus:border-red-500 focus:ring-1 focus:ring-red-500 transition-all"
                    placeholder="kullaniciadi"
                    value={instagram}
                    onChange={(e) => setInstagram(e.target.value)}
                  />
                </div>
              </div>

              <button 
                type="submit" 
                disabled={loading}
                className="w-full bg-white text-black font-bold py-3 rounded-lg hover:bg-gray-200 transition-colors mt-2 flex items-center justify-center gap-2"
              >
                {loading ? 'İşleniyor...' : 'SLOTU KAPAT'}
              </button>
            </form>

          </div>
        </div>
      )}
    </div>
  );
}
￼
