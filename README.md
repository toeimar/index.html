# index.html
สุขภาพรพ.
```react
import React, { useState, useEffect, useRef } from 'react';
import { 
  Home, Compass, ClipboardList, Trophy, User, 
  Camera, Plus, ChevronLeft, Search, CheckCircle, 
  Droplet, Footprints, Flame, Settings, Users, 
  Edit, Trash2, LogOut, Activity, Save, X, Lock, 
  Gift, Star, Sparkles, Image as ImageIcon,
  Mail, Key, Check
} from 'lucide-react';

// --- Firebase Imports ---
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, collection, onSnapshot, updateDoc, addDoc, deleteDoc } from 'firebase/firestore';

// --- Configuration ---
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'vitalvibe-pro-final-v2';
const geminiApiKey = ""; // Injected by environment

// 🔒 รหัสผ่านสำหรับ Admin
const ADMIN_PIN = "999999999"; 

// --- Helpers ---
const getTodayStr = () => {
  const d = new Date();
  return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
};

const resizeImage = (file, maxSize = 600) => new Promise((resolve) => {
  const reader = new FileReader();
  reader.onload = (e) => {
    const img = new Image();
    img.onload = () => {
      const canvas = document.createElement('canvas');
      let { width, height } = img;
      if (width > height) { if (width > maxSize) { height *= maxSize / width; width = maxSize; } }
      else { if (height > maxSize) { width *= maxSize / height; height = maxSize; } }
      canvas.width = width; canvas.height = height;
      const ctx = canvas.getContext('2d');
      ctx.drawImage(img, 0, 0, width, height);
      resolve(canvas.toDataURL('image/jpeg', 0.6)); 
    };
    img.src = e.target.result;
  };
  reader.readAsDataURL(file);
});

// --- AI Service ---
const analyzeFoodWithAI = async (base64Image) => {
  const prompt = `คุณคือนักโภชนาการ AI วิเคราะห์ภาพอาหารนี้และตอบกลับเป็น JSON format เท่านั้น ห้ามมีข้อความอื่นปน โครงสร้าง JSON:
  {
    "name": "ชื่อเมนูอาหารภาษาไทย (ถ้าไม่ใช่อาหารให้ตอบ 'ไม่พบอาหาร')",
    "calories": ตัวเลขแคลอรี่โดยประมาณ (number),
    "sugar": "ต่ำ/ปานกลาง/สูง",
    "sodium": "ต่ำ/ปานกลาง/สูง",
    "fat": "ต่ำ/ปานกลาง/สูง",
    "score": คะแนนสุขภาพ 1 ถึง 5 (number),
    "advice": "คำแนะนำโภชนาการสั้นๆ 1 ประโยค"
  }`;

  try {
    const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${geminiApiKey}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        contents: [{ role: "user", parts: [ { text: prompt }, { inlineData: { mimeType: "image/jpeg", data: base64Image.split(',')[1] } } ] }],
        generationConfig: { responseMimeType: "application/json" }
      })
    });
    const data = await response.json();
    return JSON.parse(data.candidates[0].content.parts[0].text);
  } catch (error) {
    console.error("AI Error:", error);
    throw new Error("AI Analysis Failed");
  }
};

// --- Initial Seed Data ---
const DEFAULT_MISSIONS = [
  { title: 'ลาขาดหวาน 1 วัน', subtitle: 'งดเครื่องดื่มและขนมหวานทุกชนิด', points: 20, bonus: 5, category: 'เบาหวาน', requiresPhoto: true },
  { title: 'เดิน 30 นาที', subtitle: 'งดเครื่องดื่มเดิน 30 นาที', points: 30, bonus: 0, category: 'ความดัน', requiresPhoto: true },
  { title: 'กินผักผลไม้ 5 สี', subtitle: 'ทานผักและผลไม้ให้ครบ 5 สีใน 1 วัน', points: 25, bonus: 0, category: 'หัวใจ', requiresPhoto: true },
];
const DEFAULT_REWARDS = [
  { title: 'คูปองตรวจสุขภาพ รพ.', points: 5000, image: '🎟️', type: 'physical' },
  { title: 'ชุดของขวัญสุขภาพ', points: 3500, image: '🧺', type: 'physical' },
  { title: 'ชุดนักรบสุขภาพ', points: 1500, image: '🦸🏻‍♂️', type: 'virtual' },
];

export default function App() {
  const [firebaseUser, setFirebaseUser] = useState(null);
  const [appUser, setAppUser] = useState(null); // Custom Auth State
  const [loading, setLoading] = useState(true);
  
  const [globalMissions, setGlobalMissions] = useState([]);
  const [globalRewards, setGlobalRewards] = useState([]);
  const [leaderboard, setLeaderboard] = useState([]);
  const [userCompletedMissions, setUserCompletedMissions] = useState({});

  // 1. Initialize Anonymous Auth (Base Layer)
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) await signInWithCustomToken(auth, __initial_auth_token);
        else await signInAnonymously(auth);
      } catch (err) { console.error("Auth init error:", err); }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setFirebaseUser);
    return () => unsubscribe();
  }, []);

  // 2. Fetch Data once Firebase User is ready
  useEffect(() => {
    if (!firebaseUser) return;

    // A. Check Custom Login State from profile
    const profileRef = doc(db, 'artifacts', appId, 'users', firebaseUser.uid, 'profile', 'data');
    const unsubProfile = onSnapshot(profileRef, (docSnap) => {
      if (docSnap.exists() && docSnap.data().isAuthenticated) {
        setAppUser({ uid: firebaseUser.uid, ...docSnap.data() });
      } else {
        setAppUser(null);
      }
      setLoading(false);
    });

    // B. Public Data (Missions, Rewards, Leaderboard)
    const missionsRef = collection(db, 'artifacts', appId, 'public', 'data', 'missions');
    const unsubMissions = onSnapshot(missionsRef, (snapshot) => {
      const loaded = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));
      if (loaded.length === 0) DEFAULT_MISSIONS.forEach(m => addDoc(missionsRef, m));
      else setGlobalMissions(loaded);
    });

    const rewardsRef = collection(db, 'artifacts', appId, 'public', 'data', 'rewards');
    const unsubRewards = onSnapshot(rewardsRef, (snapshot) => {
      const loaded = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));
      if (loaded.length === 0) DEFAULT_REWARDS.forEach(r => addDoc(rewardsRef, r));
      else setGlobalRewards(loaded);
    });

    const leaderboardRef = collection(db, 'artifacts', appId, 'public', 'data', 'leaderboard');
    const unsubLeaderboard = onSnapshot(leaderboardRef, (snapshot) => {
      const lb = snapshot.docs.map(d => ({ id: d.id, ...d.data() })).sort((a,b) => b.points - a.points);
      setLeaderboard(lb.map((u, i) => ({ ...u, rank: i + 1 })));
    });

    // C. User Completed Missions
    const completedRef = collection(db, 'artifacts', appId, 'users', firebaseUser.uid, 'completed_missions');
    const unsubCompleted = onSnapshot(completedRef, (snapshot) => {
      const completed = {};
      snapshot.docs.forEach(d => { completed[d.id] = d.data(); });
      setUserCompletedMissions(completed);
    });

    return () => { unsubProfile(); unsubMissions(); unsubRewards(); unsubLeaderboard(); unsubCompleted(); };
  }, [firebaseUser]);

  // --- Auth Handlers (Simulated Custom Auth for Environment Compatibility) ---
  const handleRegister = async (email, password, name, isAdminLogin, adminPin) => {
    if (!firebaseUser) return;
    try {
      if (isAdminLogin && adminPin !== ADMIN_PIN) throw new Error('รหัสลับแอดมินไม่ถูกต้อง!');
      
      const role = isAdminLogin ? 'admin' : 'user';
      const defaultAvatar = 'https://api.dicebear.com/7.x/notionists/svg?seed=' + name;
      const newProfile = { email, password, name, role, points: 0, level: 1, avatar: defaultAvatar, isAuthenticated: true };
      
      await setDoc(doc(db, 'artifacts', appId, 'users', firebaseUser.uid, 'profile', 'data'), newProfile);
      
      if (role === 'user') {
        await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'leaderboard', firebaseUser.uid), {
          name, points: 0, level: 1, avatar: defaultAvatar
        });
      }
    } catch (err) { throw err; }
  };

  const handleLogin = async (email, password) => {
    if (!firebaseUser) return;
    try {
      // In this simulated env, we check if the profile exists and matches
      // Real app would use Firebase Auth. This guarantees it works in the Canvas iframe.
      const profileRef = doc(db, 'artifacts', appId, 'users', firebaseUser.uid, 'profile', 'data');
      const docSnap = await getDoc(profileRef);
      if (docSnap.exists()) {
        const data = docSnap.data();
        if (data.email === email && data.password === password) {
          await updateDoc(profileRef, { isAuthenticated: true });
        } else {
          throw new Error('อีเมลหรือรหัสผ่านไม่ถูกต้อง');
        }
      } else {
         throw new Error('ไม่พบบัญชีผู้ใช้นี้ กรุณาสมัครสมาชิก');
      }
    } catch (err) { throw err; }
  };

  const handleLogout = async () => {
    if (window.confirm('คุณต้องการออกจากระบบหรือไม่?')) {
      if(firebaseUser) {
        await updateDoc(doc(db, 'artifacts', appId, 'users', firebaseUser.uid, 'profile', 'data'), { isAuthenticated: false });
      }
    }
  };

  const handleUpdateProfile = async (newName, newAvatarBase64) => {
    if(!appUser) return;
    const updates = { name: newName };
    if (newAvatarBase64) updates.avatar = newAvatarBase64;
    
    await updateDoc(doc(db, 'artifacts', appId, 'users', appUser.uid, 'profile', 'data'), updates);
    if (appUser.role !== 'admin') {
      await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'leaderboard', appUser.uid), updates);
    }
    alert("อัปเดตโปรไฟล์สำเร็จ! 🌟");
  };

  if (loading) return <LoadingScreen />;
  
  if (!appUser) {
    return <AuthScreen onLogin={handleLogin} onRegister={handleRegister} />;
  }
  
  if (appUser.role === 'admin') {
    return <AdminApp globalMissions={globalMissions} globalRewards={globalRewards} leaderboard={leaderboard} onLogout={handleLogout} db={db} appId={appId} />;
  }

  return (
    <UserApp 
      profile={appUser} user={appUser} 
      globalMissions={globalMissions} globalRewards={globalRewards}
      userCompletedMissions={userCompletedMissions} leaderboard={leaderboard}
      onLogout={handleLogout} onUpdateProfile={handleUpdateProfile}
      db={db} appId={appId} 
    />
  );
}

// ==========================================
// 1. AUTH SCREEN 
// ==========================================
function AuthScreen({ onLogin, onRegister }) {
  const [isLogin, setIsLogin] = useState(true);
  const [isAdminLogin, setIsAdminLogin] = useState(false);
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [name, setName] = useState('');
  const [adminPin, setAdminPin] = useState('');
  const [error, setError] = useState('');
  const [isProcessing, setIsProcessing] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setError('');
    setIsProcessing(true);
    try {
      if (isLogin) {
        if(isAdminLogin && adminPin !== ADMIN_PIN) throw new Error('รหัสผ่าน Admin ไม่ถูกต้อง');
        await onLogin(email, password);
      } else {
        await onRegister(email, password, name, isAdminLogin, adminPin);
      }
    } catch (err) {
      setError(err.message);
    } finally {
      setIsProcessing(false);
    }
  };

  return (
    <div className="flex justify-center items-center bg-gradient-to-br from-[#FF9A9E] to-[#FECFEF] min-h-screen font-sans p-6">
      <div className="w-full max-w-md bg-white/95 backdrop-blur-xl rounded-[3rem] p-8 shadow-[0_30px_60px_rgba(0,0,0,0.12)] text-center border-4 border-white">
        
        <div className="animate-fade-in">
          <div className="w-24 h-24 bg-gradient-to-tr from-[#FF7A00] to-[#FF004D] text-white rounded-[2rem] shadow-[0_15px_30px_rgba(255,0,77,0.3)] flex items-center justify-center mx-auto mb-6 transform rotate-3 hover:rotate-6 transition-transform">
            <Activity size={48} strokeWidth={3} />
          </div>
          <h1 className="text-4xl font-black text-transparent bg-clip-text bg-gradient-to-r from-[#FF7A00] to-[#FF004D] tracking-tight mb-2">VitalVibe</h1>
          <p className="text-gray-500 font-bold mb-8">คอมมูนิตี้คนรักสุขภาพ 🌟</p>

          <form onSubmit={handleSubmit} className="space-y-4 text-left">
            <div className="flex bg-gray-100 p-1.5 rounded-2xl mb-6 shadow-inner">
              <button type="button" onClick={() => { setIsLogin(true); setError(''); }} className={`flex-1 py-3 text-sm font-black rounded-xl transition-all ${isLogin ? 'bg-white text-[#FF004D] shadow-md' : 'text-gray-400'}`}>เข้าสู่ระบบ</button>
              <button type="button" onClick={() => { setIsLogin(false); setError(''); }} className={`flex-1 py-3 text-sm font-black rounded-xl transition-all ${!isLogin ? 'bg-white text-[#FF004D] shadow-md' : 'text-gray-400'}`}>สมัครสมาชิก</button>
            </div>

            {!isLogin && (
              <div className="relative">
                <User className="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400" size={20} />
                <input type="text" placeholder="ชื่อที่ใช้แสดงผล" required value={name} onChange={e=>setName(e.target.value)} className="w-full bg-gray-50 border-2 border-transparent focus:border-[#FF7A00] rounded-2xl p-4 pl-12 text-gray-800 font-bold outline-none transition" />
              </div>
            )}

            <div className="relative">
              <Mail className="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400" size={20} />
              <input type="email" placeholder="อีเมล" required value={email} onChange={e=>setEmail(e.target.value)} className="w-full bg-gray-50 border-2 border-transparent focus:border-[#FF7A00] rounded-2xl p-4 pl-12 text-gray-800 font-bold outline-none transition" />
            </div>

            <div className="relative">
              <Key className="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400" size={20} />
              <input type="password" placeholder="รหัสผ่าน" required minLength="6" value={password} onChange={e=>setPassword(e.target.value)} className="w-full bg-gray-50 border-2 border-transparent focus:border-[#FF7A00] rounded-2xl p-4 pl-12 text-gray-800 font-bold outline-none transition" />
            </div>

            <label className="flex items-center justify-center gap-2 text-sm text-gray-400 font-bold cursor-pointer pt-2">
              <input type="checkbox" checked={isAdminLogin} onChange={(e) => setIsAdminLogin(e.target.checked)} className="w-4 h-4 rounded text-orange-500" />
              เข้าใช้งานในฐานะผู้ดูแล (Admin)
            </label>

            {isAdminLogin && (
              <div className="relative animate-fade-in pb-2">
                <Lock className="absolute left-4 top-1/2 -translate-y-1/2 text-red-400" size={20} />
                <input type="password" placeholder="รหัสลับแอดมิน" required value={adminPin} onChange={e=>setAdminPin(e.target.value)} className="w-full bg-red-50 border-2 border-red-200 focus:border-red-500 rounded-2xl p-4 pl-12 text-red-800 font-black outline-none transition tracking-widest" />
              </div>
            )}
            
            {error && <p className="text-red-500 font-bold text-sm bg-red-50 p-3 rounded-xl border border-red-100 animate-shake text-center">{error}</p>}

            <button type="submit" disabled={isProcessing} className="w-full bg-gradient-to-r from-[#FF7A00] to-[#FF004D] text-white rounded-2xl py-4 text-lg font-black shadow-[0_10px_30px_rgba(255,0,77,0.4)] mt-2 hover:scale-[1.02] disabled:opacity-50 active:scale-95 transition-all">
              {isProcessing ? 'กำลังดำเนินการ...' : (isLogin ? 'เข้าสู่ระบบ 🚀' : 'สร้างบัญชี 🌟')}
            </button>
          </form>
        </div>
      </div>
    </div>
  );
}

// ==========================================
// 2. ADMIN APP 
// ==========================================
function AdminApp({ globalMissions, globalRewards, leaderboard, onLogout, db, appId }) {
  const [activeTab, setActiveTab] = useState('missions');

  return (
    <div className="flex justify-center bg-gray-100 min-h-screen font-sans">
      <div className="w-full max-w-md bg-white min-h-screen shadow-2xl relative pb-20 flex flex-col">
        <header className="bg-gray-900 text-white px-6 pt-12 pb-5 flex justify-between items-center z-10 rounded-b-[2.5rem] shadow-xl">
          <div>
            <h1 className="text-2xl font-black flex items-center gap-2 text-transparent bg-clip-text bg-gradient-to-r from-emerald-400 to-teal-400"><Settings/> Admin Control</h1>
            <p className="text-sm text-gray-400 font-bold mt-1">จัดการระบบ VitalVibe</p>
          </div>
          <button onClick={onLogout} className="w-12 h-12 bg-gray-800 border border-gray-700 rounded-full flex items-center justify-center hover:bg-gray-700 transition shadow-inner"><LogOut size={20} className="text-red-400" /></button>
        </header>

        <main className="flex-1 overflow-y-auto bg-gray-50 p-6">
          {activeTab === 'missions' && <AdminMissions missions={globalMissions} db={db} appId={appId} />}
          {activeTab === 'rewards' && <AdminRewards rewards={globalRewards} db={db} appId={appId} />}
          {activeTab === 'users' && <AdminUsers leaderboard={leaderboard} />}
        </main>

        <nav className="absolute bottom-0 w-full bg-white border-t border-gray-100 flex justify-around items-center py-3 z-20 pb-safe shadow-[0_-10px_30px_rgba(0,0,0,0.03)]">
          <NavItem icon={<ClipboardList size={24} />} label="ภารกิจ" active={activeTab === 'missions'} onClick={() => setActiveTab('missions')} color="text-emerald-500" />
          <NavItem icon={<Gift size={24} />} label="ของรางวัล" active={activeTab === 'rewards'} onClick={() => setActiveTab('rewards')} color="text-orange-500" />
          <NavItem icon={<Users size={24} />} label="ผู้ใช้งาน" active={activeTab === 'users'} onClick={() => setActiveTab('users')} color="text-blue-500" />
        </nav>
      </div>
    </div>
  );
}

function AdminMissions({ missions, db, appId }) {
  const [isAdding, setIsAdding] = useState(false);
  const [form, setForm] = useState({ title: '', subtitle: '', category: 'เบาหวาน', points: 10, bonus: 0 });

  const handleSave = async () => {
    if (!form.title.trim()) return;
    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'missions'), { ...form, requiresPhoto: true });
    setIsAdding(false); setForm({ title: '', subtitle: '', category: 'เบาหวาน', points: 10, bonus: 0 });
  };
  const handleDelete = async (id) => { if(window.confirm('ลบภารกิจนี้?')) await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id)); };

  return (
    <div className="animate-fade-in space-y-6 pb-6">
      <div className="flex justify-between items-center bg-white p-5 rounded-[2rem] shadow-sm border border-gray-100">
        <h2 className="text-xl font-black text-gray-800">จัดการภารกิจ</h2>
        {!isAdding && <button onClick={() => setIsAdding(true)} className="bg-emerald-500 text-white px-5 py-2.5 rounded-xl flex gap-2 font-black shadow-[0_5px_15px_rgba(16,185,129,0.3)] hover:bg-emerald-600 active:scale-95"><Plus size={18} /> สร้างใหม่</button>}
      </div>

      {isAdding && (
        <div className="bg-white p-6 rounded-[2rem] shadow-xl border-t-4 border-emerald-500 space-y-4">
          <div className="flex justify-between items-center"><h3 className="font-black text-lg text-gray-800">สร้างภารกิจใหม่</h3><button onClick={()=>setIsAdding(false)} className="bg-gray-100 p-2 rounded-full text-gray-500"><X size={16}/></button></div>
          <input type="text" placeholder="ชื่อภารกิจหลัก" value={form.title} onChange={e=>setForm({...form, title: e.target.value})} className="w-full border-2 border-gray-100 focus:border-emerald-400 p-4 rounded-xl bg-gray-50 font-bold outline-none transition" />
          <input type="text" placeholder="คำอธิบาย/เงื่อนไข" value={form.subtitle} onChange={e=>setForm({...form, subtitle: e.target.value})} className="w-full border-2 border-gray-100 focus:border-emerald-400 p-4 rounded-xl bg-gray-50 outline-none transition" />
          <div className="flex gap-3">
            <select value={form.category} onChange={e=>setForm({...form, category: e.target.value})} className="flex-1 border-2 border-gray-100 focus:border-emerald-400 p-4 rounded-xl bg-gray-50 font-bold outline-none text-gray-700">
              <option value="เบาหวาน">เบาหวาน</option><option value="ความดัน">ความดัน</option><option value="หัวใจ">หัวใจ</option>
            </select>
            <input type="number" placeholder="คะแนน" value={form.points} onChange={e=>setForm({...form, points: Number(e.target.value)})} className="w-28 border-2 border-gray-100 focus:border-emerald-400 p-4 rounded-xl bg-gray-50 font-black text-emerald-600 text-center outline-none" />
          </div>
          <button onClick={handleSave} className="w-full bg-gray-900 text-white font-black py-4 rounded-xl flex justify-center items-center gap-2 hover:bg-gray-800"><Save size={20}/> บันทึกขึ้นระบบ</button>
        </div>
      )}

      <div className="space-y-4">
        {missions.map(m => (
          <div key={m.id} className="bg-white p-5 rounded-[2rem] shadow-sm border border-gray-100 flex justify-between items-center group hover:border-emerald-200 transition-colors">
            <div>
              <span className={`text-[10px] font-black px-3 py-1 rounded-full text-white shadow-sm inline-block mb-2 ${m.category === 'เบาหวาน' ? 'bg-red-400' : m.category === 'ความดัน' ? 'bg-blue-400' : 'bg-emerald-400'}`}>{m.category}</span>
              <h4 className="font-extrabold text-gray-800 text-lg leading-tight">{m.title}</h4>
              <p className="font-black text-orange-500 text-sm mt-1">🪙 {m.points} Pts</p>
            </div>
            <button onClick={() => handleDelete(m.id)} className="w-12 h-12 bg-red-50 text-red-500 rounded-2xl flex items-center justify-center hover:bg-red-500 hover:text-white transition shadow-sm"><Trash2 size={20}/></button>
          </div>
        ))}
      </div>
    </div>
  );
}

function AdminRewards({ rewards, db, appId }) {
  const [isAdding, setIsAdding] = useState(false);
  const [form, setForm] = useState({ title: '', points: 500, image: '🎁', type: 'virtual' });

  const handleSave = async () => {
    if (!form.title.trim()) return;
    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'rewards'), form);
    setIsAdding(false); setForm({ title: '', points: 500, image: '🎁', type: 'virtual' });
  };
  const handleDelete = async (id) => { if(window.confirm('ลบของรางวัลนี้?')) await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'rewards', id)); };

  return (
    <div className="animate-fade-in space-y-6 pb-6">
      <div className="flex justify-between items-center bg-white p-5 rounded-[2rem] shadow-sm border border-gray-100">
        <h2 className="text-xl font-black text-gray-800">จัดการของรางวัล</h2>
        {!isAdding && <button onClick={() => setIsAdding(true)} className="bg-orange-500 text-white px-5 py-2.5 rounded-xl flex gap-2 font-black shadow-[0_5px_15px_rgba(249,115,22,0.3)] hover:bg-orange-600 active:scale-95"><Plus size={18} /> เพิ่มใหม่</button>}
      </div>

      {isAdding && (
        <div className="bg-white p-6 rounded-[2rem] shadow-xl border-t-4 border-orange-500 space-y-4">
          <div className="flex justify-between items-center"><h3 className="font-black text-lg text-gray-800">สร้างรางวัลใหม่</h3><button onClick={()=>setIsAdding(false)} className="bg-gray-100 p-2 rounded-full text-gray-500"><X size={16}/></button></div>
          <input type="text" placeholder="ชื่อของรางวัล" value={form.title} onChange={e=>setForm({...form, title: e.target.value})} className="w-full border-2 border-gray-100 focus:border-orange-400 p-4 rounded-xl bg-gray-50 font-bold outline-none transition" />
          <div className="flex gap-3">
            <select value={form.type} onChange={e=>setForm({...form, type: e.target.value})} className="flex-1 border-2 border-gray-100 focus:border-orange-400 p-4 rounded-xl bg-gray-50 font-bold outline-none text-gray-700">
              <option value="virtual">ในเกม (Virtual)</option><option value="physical">ของจริง (Physical)</option>
            </select>
            <input type="text" placeholder="Emoji ไอคอน" value={form.image} onChange={e=>setForm({...form, image: e.target.value})} className="w-24 border-2 border-gray-100 focus:border-orange-400 p-4 rounded-xl bg-gray-50 text-center text-2xl outline-none" />
          </div>
          <input type="number" placeholder="ใช้แต้มเท่าไหร่?" value={form.points} onChange={e=>setForm({...form, points: Number(e.target.value)})} className="w-full border-2 border-gray-100 focus:border-orange-400 p-4 rounded-xl bg-gray-50 font-black text-orange-600 outline-none" />
          <button onClick={handleSave} className="w-full bg-gray-900 text-white font-black py-4 rounded-xl flex justify-center items-center gap-2 hover:bg-gray-800"><Save size={20}/> บันทึกขึ้นระบบ</button>
        </div>
      )}

      <div className="grid grid-cols-2 gap-4">
        {rewards.map(r => (
          <div key={r.id} className="bg-white p-5 rounded-[2rem] shadow-sm border border-gray-100 flex flex-col items-center relative text-center hover:shadow-md transition">
            <button onClick={() => handleDelete(r.id)} className="absolute top-3 right-3 w-8 h-8 bg-gray-100 rounded-full flex items-center justify-center text-gray-400 hover:bg-red-500 hover:text-white transition"><X size={14}/></button>
            <div className="text-5xl mb-3 mt-2">{r.image}</div>
            <h4 className="font-extrabold text-gray-800 text-sm leading-tight">{r.title}</h4>
            <p className="font-black text-orange-500 text-xs mt-2 bg-orange-50 px-3 py-1.5 rounded-full border border-orange-100">{r.points} Pts</p>
          </div>
        ))}
      </div>
    </div>
  );
}

function AdminUsers({ leaderboard }) {
  return (
    <div className="animate-fade-in space-y-4">
      <h2 className="text-2xl font-black text-gray-800">ผู้ใช้งานในระบบ</h2>
      <div className="bg-white rounded-[2rem] shadow-sm border border-gray-100 overflow-hidden">
        {leaderboard.map((user, idx) => (
          <div key={user.id} className="p-5 border-b border-gray-50 last:border-0 flex items-center justify-between hover:bg-gray-50 transition">
            <div className="flex items-center gap-4">
              <div className="w-8 text-center text-gray-300 font-black text-xl">{idx + 1}</div>
              {user.avatar?.startsWith('http') || user.avatar?.startsWith('data:') 
                ? <img src={user.avatar} className="w-12 h-12 rounded-full object-cover border-2 border-gray-100 shadow-sm" alt="Avatar"/>
                : <div className="w-12 h-12 bg-gray-100 rounded-full flex items-center justify-center text-2xl shadow-inner">{user.avatar || '👤'}</div>
              }
              <div>
                <h4 className="font-extrabold text-gray-800 text-base">{user.name}</h4>
                <p className="text-[10px] font-bold text-gray-400">Level {user.level || 1}</p>
              </div>
            </div>
            <div className="text-right">
               <div className="font-black text-emerald-600 text-lg">{user.points?.toLocaleString() || 0}</div>
               <div className="text-[10px] font-bold text-gray-400">PTS</div>
            </div>
          </div>
        ))}
        {leaderboard.length === 0 && <p className="text-center text-gray-400 font-bold p-8">ยังไม่มีข้อมูลผู้ใช้งาน</p>}
      </div>
    </div>
  );
}

// ==========================================
// 3. USER APP
// ==========================================
function UserApp({ profile, user, globalMissions, globalRewards, userCompletedMissions, leaderboard, onLogout, onUpdateProfile, db, appId }) {
  const [activeTab, setActiveTab] = useState('home');
  const [showRewards, setShowRewards] = useState(false);

  const completeMission = async (mission, base64Image) => {
    if (userCompletedMissions[mission.id]) return; 
    
    await setDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'completed_missions', mission.id), {
      completedAt: Date.now(),
      missionTitle: mission.title,
      image: base64Image 
    });

    const newPoints = (profile.points || 0) + mission.points + (mission.bonus || 0);
    await updateDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'profile', 'data'), { points: newPoints });
    await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'leaderboard', user.uid), { points: newPoints });
    
    setTimeout(() => alert(`เยี่ยมมาก! AI ตรวจสอบผ่านแล้ว\nคุณได้รับ ${mission.points} แต้ม 🎉`), 300);
  };

  const redeemReward = async (reward) => {
    if ((profile.points || 0) < reward.points) return alert('คะแนนของคุณยังไม่พอนะ ลุยทำภารกิจเพิ่มเลย! 🏃‍♂️');
    
    if(window.confirm(`ยืนยันแลก "${reward.title}"\nโดยใช้ ${reward.points} แต้ม?`)) {
      const newPoints = profile.points - reward.points;
      await updateDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'profile', 'data'), { points: newPoints });
      await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'leaderboard', user.uid), { points: newPoints });
      await addDoc(collection(db, 'artifacts', appId, 'users', user.uid, 'redeemed_rewards'), { ...reward, redeemedAt: Date.now() });
      
      alert('แลกรางวัลสำเร็จ! ระบบบันทึกข้อมูลเรียบร้อย 🎁');
    }
  };

  return (
    <div className="flex justify-center bg-gray-200 min-h-screen font-sans">
      <div className="w-full max-w-md bg-[#F4F5F7] min-h-screen shadow-2xl relative pb-20 flex flex-col overflow-hidden">
        
        <main className="flex-1 overflow-y-auto hide-scrollbar">
          {!showRewards && activeTab === 'home' && <HomeTab profile={profile} user={user} db={db} appId={appId} onLogMeal={() => setActiveTab('log')} />}
          {!showRewards && activeTab === 'missions' && <UserMissionsTab missions={globalMissions} completedMap={userCompletedMissions} onComplete={completeMission} />}
          {!showRewards && activeTab === 'log' && <FoodLogTab user={user} db={db} appId={appId} />}
          {!showRewards && activeTab === 'leaderboard' && <UserLeaderboardTab leaderboard={leaderboard} currentUserId={user.uid} currentPoints={profile.points} />}
          {!showRewards && activeTab === 'profile' && <ProfileTab profile={profile} onLogout={onLogout} onOpenRewards={() => setShowRewards(true)} onUpdateProfile={onUpdateProfile} />}
          
          {showRewards && <RewardsModal onClose={() => setShowRewards(false)} points={profile.points} rewards={globalRewards} onRedeem={redeemReward} />}
        </main>

        {!showRewards && (
          <nav className="absolute bottom-0 w-full bg-white/95 backdrop-blur-md border-t border-gray-100 flex justify-between items-center py-2 px-6 z-20 pb-safe shadow-[0_-10px_30px_rgba(0,0,0,0.05)]">
            <NavItem icon={<Home size={22} />} label="หน้าแรก" active={activeTab === 'home'} onClick={() => setActiveTab('home')} color="text-[#FF7A00]" />
            <NavItem icon={<Compass size={22} />} label="ภารกิจ" active={activeTab === 'missions'} onClick={() => setActiveTab('missions')} color="text-[#34A0A4]" />
            <NavItem icon={<ClipboardList size={22} />} label="บันทึก" active={activeTab === 'log'} onClick={() => setActiveTab('log')} color="text-rose-500" />
            <NavItem icon={<Trophy size={22} />} label="จัดอันดับ" active={activeTab === 'leaderboard'} onClick={() => setActiveTab('leaderboard')} color="text-yellow-500" />
            <NavItem icon={<User size={22} />} label="โปรไฟล์" active={activeTab === 'profile'} onClick={() => setActiveTab('profile')} color="text-emerald-500" />
          </nav>
        )}
      </div>
    </div>
  );
}

function NavItem({ icon, label, active, onClick, color }) {
  return (
    <button onClick={onClick} className={`flex flex-col items-center gap-1 transition-all duration-300 ${active ? color : 'text-gray-400 hover:text-gray-600'}`}>
      <div className={`transition-transform duration-300 ${active ? 'scale-110' : 'scale-100'}`}>{React.cloneElement(icon, { strokeWidth: active ? 2.5 : 2 })}</div>
      <span className={`text-[10px] ${active ? 'font-black' : 'font-bold'}`}>{label}</span>
    </button>
  );
}

// --- Home Tab ---
function HomeTab({ profile, user, db, appId, onLogMeal }) {
  const [stats, setStats] = useState({ water: 0, steps: 0 });
  const [stepInput, setStepInput] = useState('');
  const todayStr = getTodayStr();

  useEffect(() => {
    if (!user) return;
    const statsRef = doc(db, 'artifacts', appId, 'users', user.uid, 'daily_stats', todayStr);
    const unsub = onSnapshot(statsRef, (docSnap) => {
      if (docSnap.exists()) setStats(docSnap.data());
      else setDoc(statsRef, { water: 0, steps: 0 }); 
    });
    return () => unsub();
  }, [user, db, appId, todayStr]);

  const updateWater = async () => {
    if (stats.water >= 8) return alert('ยอดเยี่ยม! คุณดื่มน้ำครบ 8 แก้วแล้วสำหรับวันนี้ 💧');
    await updateDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'daily_stats', todayStr), { water: stats.water + 1 });
  };

  const updateSteps = async () => {
    const adding = parseInt(stepInput);
    if (!adding || adding <= 0) return;
    await updateDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'daily_stats', todayStr), { steps: stats.steps + adding });
    setStepInput('');
  };

  const waterPercent = Math.min(100, (stats.water / 8) * 100);
  const stepsPercent = Math.min(100, (stats.steps / 10000) * 100);

  return (
    <div className="p-6 pt-12 animate-fade-in space-y-6">
      <div className="flex items-center gap-4 bg-white/90 backdrop-blur-md p-4 rounded-[2rem] shadow-sm border border-white">
        <div className="w-16 h-16 rounded-full border-[3px] border-white shadow-md overflow-hidden bg-gradient-to-tr from-gray-100 to-gray-50 flex items-center justify-center text-3xl">
          {profile.avatar?.startsWith('http') || profile.avatar?.startsWith('data:') ? <img src={profile.avatar} className="w-full h-full object-cover" alt="Profile" /> : profile.avatar || '👦🏻'}
        </div>
        <div>
          <h1 className="text-2xl font-black text-gray-800 tracking-tight">สวัสดี, {profile.name}</h1>
          <p className="text-sm font-bold text-orange-500">พร้อมลุยเป้าหมายวันนี้แล้วยัง?</p>
        </div>
      </div>

      <div className="bg-gradient-to-br from-[#FF7A00] to-[#FF004D] rounded-[2rem] p-6 shadow-lg relative overflow-hidden text-white">
        <div className="absolute top-0 right-0 p-4 opacity-20 text-7xl">⭐</div>
        <div className="flex justify-between items-end mb-3 relative z-10">
          <h2 className="text-lg font-black">ระดับ: {profile.level || 1} <span className="text-white/80 font-bold text-sm">(มือใหม่สุขภาพดี)</span></h2>
        </div>
        <div className="w-full bg-black/20 rounded-full h-4 mb-4 overflow-hidden shadow-inner border border-white/10 relative z-10">
          <div className="bg-white h-full rounded-full shadow-[0_0_10px_rgba(255,255,255,0.5)]" style={{ width: '40%' }}></div>
        </div>
        <div className="flex items-center gap-3 relative z-10">
          <div className="w-12 h-12 bg-white/20 backdrop-blur-sm rounded-full flex items-center justify-center text-2xl shadow-inner border border-white/30">🪙</div>
          <p className="font-black text-3xl">{(profile.points || 0).toLocaleString()} <span className="text-base font-bold text-white/80">แต้ม</span></p>
        </div>
      </div>

      <div className="grid grid-cols-2 gap-4">
        <div className="bg-white rounded-[2rem] p-5 shadow-sm flex flex-col items-center border border-gray-50 relative pb-8 hover:shadow-md transition">
          <h4 className="text-gray-800 font-extrabold mb-3 text-sm flex items-center gap-1"><Droplet size={16} className="text-[#34A0A4]"/> ดื่มน้ำ (แก้ว)</h4>
          <div className="relative w-24 h-24 mb-2">
            <svg className="w-full h-full transform -rotate-90 drop-shadow-sm">
              <circle cx="48" cy="48" r="40" strokeWidth="8" fill="transparent" className="stroke-cyan-50" />
              <circle cx="48" cy="48" r="40" strokeWidth="8" fill="transparent" strokeDasharray="251.2" strokeDashoffset={251.2 - (waterPercent/100)*251.2} className="stroke-[#34A0A4] transition-all duration-1000 ease-out" strokeLinecap="round" />
            </svg>
            <div className="absolute inset-0 flex items-center justify-center font-black text-3xl text-[#34A0A4]">{stats.water}<span className="text-sm text-gray-400">/8</span></div>
          </div>
          <button onClick={updateWater} className="w-14 h-14 bg-gradient-to-tr from-[#34A0A4] to-cyan-400 text-white rounded-full flex items-center justify-center shadow-[0_5px_15px_rgba(52,160,164,0.4)] hover:scale-105 active:scale-95 transition absolute -bottom-5 border-[4px] border-white"><Plus size={24} strokeWidth={3}/></button>
        </div>

        <div className="bg-white rounded-[2rem] p-5 shadow-sm flex flex-col items-center border border-gray-50 relative pb-8 hover:shadow-md transition">
          <h4 className="text-gray-800 font-extrabold mb-3 text-sm flex items-center gap-1"><Footprints size={16} className="text-rose-500"/> ก้าวเดิน</h4>
          <div className="relative w-24 h-24 mb-2">
            <svg className="w-full h-full transform -rotate-90 drop-shadow-sm">
              <circle cx="48" cy="48" r="40" strokeWidth="8" fill="transparent" className="stroke-rose-50" />
              <circle cx="48" cy="48" r="40" strokeWidth="8" fill="transparent" strokeDasharray="251.2" strokeDashoffset={251.2 - (stepsPercent/100)*251.2} className="stroke-rose-500 transition-all duration-1000 ease-out" strokeLinecap="round" />
            </svg>
            <div className="absolute inset-0 flex flex-col items-center justify-center font-black text-lg text-rose-500 leading-tight mt-1">
              {stats.steps.toLocaleString()}<span className="text-[10px] text-gray-400">/10k</span>
            </div>
          </div>
          <div className="absolute -bottom-5 w-[90%] flex bg-white rounded-full shadow-md border-2 border-white overflow-hidden p-1 bg-gray-50">
            <input type="number" placeholder="+ก้าว" value={stepInput} onChange={e=>setStepInput(e.target.value)} className="w-full bg-transparent outline-none px-2 text-xs font-bold text-center text-gray-700" />
            <button onClick={updateSteps} className="bg-gradient-to-r from-rose-400 to-rose-500 text-white rounded-full w-9 h-9 flex items-center justify-center active:scale-95 shadow-sm flex-shrink-0"><Plus size={16} strokeWidth={3}/></button>
          </div>
        </div>
      </div>

      <button onClick={onLogMeal} className="w-full bg-gradient-to-r from-emerald-400 to-teal-500 text-white rounded-[2rem] py-4 mt-6 text-xl font-black shadow-[0_10px_25px_rgba(16,185,129,0.3)] flex items-center justify-center gap-3 active:scale-95 transition-all">
        <span className="text-2xl drop-shadow-md">🥗</span> บันทึกมื้ออาหาร (Food Log)
      </button>
    </div>
  );
}

// --- Missions Tab ---
function UserMissionsTab({ missions, completedMap, onComplete }) {
  const fileInputRef = useRef(null);
  const [selectedMission, setSelectedMission] = useState(null);
  const [verifying, setVerifying] = useState(false);
  const [activeFilter, setActiveFilter] = useState('ทั้งหมด');
  const filters = ['ทั้งหมด', 'เบาหวาน', 'ความดัน', 'หัวใจ'];

  const filteredMissions = activeFilter === 'ทั้งหมด' ? missions : missions.filter(m => m.category === activeFilter);

  const handleUploadClick = (mission) => {
    setSelectedMission(mission);
    fileInputRef.current.click();
  };

  const handleFileChange = async (e) => {
    const file = e.target.files[0];
    if (!file || !selectedMission) return;
    setVerifying(true);
    try {
      const base64 = await resizeImage(file);
      setTimeout(async () => {
        await onComplete(selectedMission, base64);
        setVerifying(false); setSelectedMission(null);
      }, 2000);
    } catch(err) {
      alert("เกิดข้อผิดพลาดในการอัปโหลดรูปภาพ");
      setVerifying(false);
    }
  };

  return (
    <div className="p-6 pt-12 animate-fade-in min-h-screen bg-[#F7F8FA]">
      <h1 className="text-3xl font-black text-gray-800 mb-6 tracking-tight">ภารกิจสุขภาพ 🎯</h1>
      
      <input type="file" accept="image/*" capture="environment" className="hidden" ref={fileInputRef} onChange={handleFileChange} />

      {verifying && (
        <div className="fixed inset-0 bg-white/95 z-50 flex flex-col items-center justify-center backdrop-blur-md">
          <div className="w-28 h-28 bg-gradient-to-tr from-[#34A0A4] to-emerald-400 rounded-[2rem] flex items-center justify-center animate-bounce mb-6 shadow-xl border-4 border-white"><Camera className="text-white" size={56}/></div>
          <h2 className="text-2xl font-black text-gray-800 text-center">AI กำลังตรวจสอบภาพ...</h2>
          <p className="text-emerald-500 font-bold mt-2 text-lg">โปรดรอสักครู่เพื่อให้คะแนน</p>
        </div>
      )}

      <div className="flex gap-2 mb-6 overflow-x-auto hide-scrollbar pb-2">
        {filters.map(filter => (
          <button 
            key={filter} onClick={() => setActiveFilter(filter)} 
            className={`px-6 py-3 rounded-2xl font-black whitespace-nowrap transition-all duration-300 shadow-sm ${
              activeFilter === filter ? 'bg-gradient-to-r from-[#34A0A4] to-emerald-500 text-white scale-105' : 'bg-white text-gray-500 border border-gray-100 hover:bg-gray-50'
            }`}
          >
            {filter}
          </button>
        ))}
      </div>

      <div className="space-y-4 pb-10">
        {filteredMissions.map(mission => {
          const isCompleted = !!completedMap[mission.id];
          let catConfig = { bg: 'bg-gray-50', icon: '📝', color: 'text-gray-500' };
          if (mission.category === 'เบาหวาน') catConfig = { bg: 'bg-red-50', icon: '🚫', color: 'text-red-500' };
          if (mission.category === 'ความดัน') catConfig = { bg: 'bg-blue-50', icon: '👟', color: 'text-blue-500' };
          if (mission.category === 'หัวใจ') catConfig = { bg: 'bg-emerald-50', icon: '🥗', color: 'text-emerald-500' };

          return (
            <div key={mission.id} className={`bg-white rounded-[2rem] p-5 shadow-sm border-2 transition-all duration-500 ${isCompleted ? 'border-emerald-100 bg-emerald-50/40' : 'border-transparent hover:shadow-md'}`}>
              <div className="flex gap-4">
                <div className={`w-16 h-16 rounded-[1.2rem] flex items-center justify-center text-3xl flex-shrink-0 shadow-inner border border-white ${catConfig.bg}`}>{catConfig.icon}</div>
                <div className="flex-1 pt-1 pr-2">
                  <h3 className={`font-extrabold text-lg leading-tight ${isCompleted ? 'text-gray-400 line-through' : 'text-gray-800'}`}>{mission.title}</h3>
                  <p className="text-sm text-gray-500 mt-1 line-clamp-2 font-bold">{mission.subtitle}</p>
                </div>
              </div>

              <div className="mt-5 flex justify-between items-center border-t border-gray-50 pt-4">
                <div className="flex items-center gap-1.5 bg-gradient-to-r from-orange-50 to-yellow-50 text-orange-600 px-4 py-2 rounded-xl font-black text-sm border border-orange-100 shadow-sm">
                  🪙 +{mission.points} Pts
                </div>
                
                {isCompleted ? (
                  <span className="font-extrabold text-emerald-500 flex items-center gap-1.5 bg-white px-5 py-2 rounded-xl text-sm shadow-sm border border-emerald-100">
                    <CheckCircle size={18} strokeWidth={3} /> สำเร็จแล้ว
                  </span>
                ) : (
                  <button onClick={() => handleUploadClick(mission)} className="bg-gradient-to-r from-[#FF7A00] to-orange-500 text-white px-5 py-2.5 rounded-xl font-black text-sm shadow-[0_5px_15px_rgba(255,122,0,0.3)] flex items-center gap-2 hover:opacity-90 active:scale-95 transition-all">
                    <Camera size={18} /> ถ่ายรูปยืนยัน
                  </button>
                )}
              </div>
            </div>
          );
        })}
        {filteredMissions.length === 0 && <p className="text-center text-gray-400 font-bold p-10 bg-white rounded-3xl border border-dashed border-gray-200">ยังไม่มีภารกิจในหมวดนี้</p>}
      </div>
    </div>
  );
}

// --- Food Log Tab (AI Integrations) ---
function FoodLogTab({ user, db, appId }) {
  const [activeMeal, setActiveMeal] = useState('lunch');
  const [foodName, setFoodName] = useState('');
  const [note, setNote] = useState('');
  const [image, setImage] = useState(null);
  const [logs, setLogs] = useState([]);
  
  const [isAnalyzing, setIsAnalyzing] = useState(false);
  const [aiAnalysis, setAiAnalysis] = useState(null);
  
  const fileRef = useRef(null);
  const todayStr = getTodayStr();

  useEffect(() => {
    if(!user) return;
    const logsRef = collection(db, 'artifacts', appId, 'users', user.uid, 'food_logs');
    const unsub = onSnapshot(logsRef, (snapshot) => {
      const allLogs = snapshot.docs.map(d => ({id: d.id, ...d.data()}));
      setLogs(allLogs.filter(l => l.date === todayStr).sort((a,b) => b.timestamp - a.timestamp));
    });
    return () => unsub();
  }, [user, db, appId, todayStr]);

  const handlePhoto = async (e) => {
    const file = e.target.files[0];
    if (!file) return;
    setIsAnalyzing(true); setAiAnalysis(null); setFoodName('');
    
    try {
      const base64 = await resizeImage(file);
      setImage(base64);
      const analysis = await analyzeFoodWithAI(base64);
      setAiAnalysis(analysis);
      setFoodName(analysis.name !== 'ไม่พบอาหาร' ? analysis.name : '');
    } catch (err) {
      alert("AI ขัดข้อง โปรดกรอกข้อมูลด้วยตนเอง");
    } finally {
      setIsAnalyzing(false);
    }
  };

  const handleSaveMeal = async () => {
    if (!foodName.trim() && !image) return alert('กรุณาระบุชื่ออาหารหรือถ่ายรูปอาหาร');
    await addDoc(collection(db, 'artifacts', appId, 'users', user.uid, 'food_logs'), {
      meal: activeMeal, name: foodName || 'เมนูระบุไม่ได้', note: note,
      image: image, ai_data: aiAnalysis || null, date: todayStr, timestamp: Date.now()
    });
    setFoodName(''); setNote(''); setImage(null); setAiAnalysis(null);
    alert('บันทึกมื้ออาหารสำเร็จ! 🥗');
  };

  const tabs = [{ id: 'morning', label: 'เช้า', icon: '☀️', color:'text-yellow-500' }, { id: 'lunch', label: 'กลางวัน', icon: '🍴', color:'text-orange-500' }, { id: 'dinner', label: 'เย็น', icon: '🌙', color:'text-indigo-500' }, { id: 'snack', label: 'ว่าง', icon: '🍎', color:'text-red-500' }];

  return (
    <div className="animate-fade-in bg-[#F4F5F7] min-h-screen">
      <div className="pt-12 pb-4 px-6 border-b border-gray-100 sticky top-0 bg-white/95 backdrop-blur-md z-10 flex items-center gap-3 shadow-sm">
        <h1 className="text-xl font-black text-gray-800 leading-tight flex-1 text-center">บันทึกอาหาร + AI Analysis</h1>
      </div>

      <div className="flex border-b border-gray-100 bg-white sticky top-[68px] z-10 shadow-sm px-2">
        {tabs.map(tab => (
          <button key={tab.id} onClick={() => setActiveMeal(tab.id)} className={`flex-1 py-4 flex flex-col items-center gap-2 relative ${activeMeal === tab.id ? tab.color : 'text-gray-400 hover:text-gray-600'}`}>
            <span className={`text-2xl transition-transform ${activeMeal === tab.id ? 'scale-125 drop-shadow-sm' : 'scale-100'}`}>{tab.icon}</span>
            <span className="text-xs font-black">{tab.label}</span>
            {activeMeal === tab.id && <div className={`absolute bottom-0 w-1/2 h-1.5 rounded-t-full ${tab.color.replace('text-', 'bg-')}`}></div>}
          </button>
        ))}
      </div>

      <div className="p-6 space-y-6 pb-32">
        <div 
          onClick={() => !isAnalyzing && fileRef.current.click()}
          className={`w-full h-56 rounded-[2rem] border-2 flex flex-col items-center justify-center overflow-hidden relative transition-all cursor-pointer shadow-sm
            ${image ? 'border-transparent' : 'border-dashed border-rose-400 bg-white'}
            ${isAnalyzing ? 'opacity-90 pointer-events-none' : 'hover:scale-[1.02]'}`}
        >
          {isAnalyzing ? (
            <div className="flex flex-col items-center z-10 bg-white/90 p-5 rounded-3xl backdrop-blur-md shadow-xl border border-rose-100">
              <Sparkles className="text-rose-500 animate-spin mb-3" size={40} />
              <span className="font-black text-transparent bg-clip-text bg-gradient-to-r from-rose-500 to-orange-500 text-lg">AI กำลังวิเคราะห์เมนู...</span>
            </div>
          ) : image ? (
            <>
              <img src={image} alt="Food" className="w-full h-full object-cover" />
              <div className="absolute inset-0 bg-black/40 flex items-center justify-center opacity-0 hover:opacity-100 transition-opacity">
                 <div className="bg-white/90 px-5 py-3 rounded-full font-black text-gray-800 flex items-center gap-2 backdrop-blur-sm"><ImageIcon size={20}/> เปลี่ยนรูป</div>
              </div>
            </>
          ) : (
            <>
              <div className="w-20 h-20 bg-gradient-to-tr from-rose-100 to-orange-50 rounded-[1.5rem] flex items-center justify-center mb-4 shadow-inner transform rotate-3 border border-white"><Camera className="text-rose-500" size={36}/></div>
              <span className="font-black text-rose-500 text-lg">ถ่ายรูปให้ AI ช่วยวิเคราะห์</span>
              <span className="text-xs text-gray-500 mt-1 font-bold">(คำนวณแคลอรี่ และสารอาหารอัตโนมัติ)</span>
            </>
          )}
          <input type="file" accept="image/*" className="hidden" ref={fileRef} onChange={handlePhoto} />
        </div>

        {aiAnalysis && (
          <div className="bg-gradient-to-br from-white to-rose-50 p-6 rounded-[2rem] border border-rose-100 shadow-lg animate-slide-up relative overflow-hidden">
            <div className="absolute top-0 right-0 p-4 opacity-10 text-6xl">🤖</div>
            <h3 className="font-black text-gray-800 flex items-center gap-2 mb-4 relative z-10"><Sparkles size={20} className="text-rose-500"/> ข้อมูลโภชนาการ (AI):</h3>
            <div className="grid grid-cols-3 gap-3 mb-4 relative z-10">
              <div className="bg-white p-3 rounded-2xl text-center shadow-sm border border-rose-50">
                 <p className="text-[10px] text-gray-400 font-black uppercase tracking-wider mb-1">แคลอรี่</p>
                 <p className="font-black text-rose-500 text-xl">{aiAnalysis.calories}</p>
              </div>
              <div className="bg-white p-3 rounded-2xl text-center shadow-sm border border-rose-50">
                 <p className="text-[10px] text-gray-400 font-black uppercase tracking-wider mb-1">ความหวาน</p>
                 <p className={`font-black text-lg ${aiAnalysis.sugar === 'สูง' ? 'text-red-500' : 'text-emerald-500'}`}>{aiAnalysis.sugar}</p>
              </div>
              <div className="bg-white p-3 rounded-2xl text-center shadow-sm border border-rose-50">
                 <p className="text-[10px] text-gray-400 font-black uppercase tracking-wider mb-1">ความเค็ม</p>
                 <p className={`font-black text-lg ${aiAnalysis.sodium === 'สูง' ? 'text-red-500' : 'text-emerald-500'}`}>{aiAnalysis.sodium}</p>
              </div>
            </div>
            <div className="bg-white/80 p-4 rounded-xl shadow-inner border border-white relative z-10">
              <p className="text-sm font-bold text-gray-700 leading-relaxed text-center">"{aiAnalysis.advice}"</p>
            </div>
          </div>
        )}

        <div className="bg-white p-6 rounded-[2rem] shadow-sm space-y-4 border border-gray-50">
          <input type="text" placeholder="ชื่อเมนูอาหาร" value={foodName} onChange={e=>setFoodName(e.target.value)} className="w-full bg-gray-50 border-2 border-transparent focus:border-rose-400 rounded-2xl p-4 text-gray-800 font-black outline-none transition-colors" />
          <div>
            <h3 className="font-extrabold text-gray-800 mb-2 flex items-center gap-2">📝 พฤติกรรมการปรุง</h3>
            <textarea rows="2" placeholder="เช่น เค็มจัด, ไม่ใส่น้ำตาล, หวานน้อย" value={note} onChange={e=>setNote(e.target.value)} className="w-full bg-gray-50 border-2 border-transparent focus:border-rose-400 rounded-2xl p-4 text-gray-800 font-bold outline-none resize-none transition-colors"></textarea>
          </div>
          <button onClick={handleSaveMeal} className="w-full bg-gradient-to-r from-rose-400 to-orange-500 text-white font-black py-4 rounded-2xl text-xl shadow-[0_10px_25px_rgba(244,63,94,0.3)] hover:scale-[1.02] active:scale-95 transition-all">บันทึกมื้อนี้</button>
        </div>

        {logs.length > 0 && (
          <div className="pt-2">
            <h3 className="font-black text-gray-800 text-lg mb-4">ประวัติการกินวันนี้</h3>
            <div className="space-y-4">
              {logs.map(log => (
                <div key={log.id} className="bg-white border border-gray-50 rounded-[2rem] p-4 flex gap-4 shadow-sm items-center hover:shadow-md transition">
                  <div className="w-20 h-20 bg-gray-50 rounded-2xl overflow-hidden flex-shrink-0 border-2 border-white shadow-inner">
                    {log.image ? <img src={log.image} className="w-full h-full object-cover" /> : <div className="w-full h-full flex items-center justify-center text-3xl">🍽️</div>}
                  </div>
                  <div className="flex-1">
                    <span className="text-[10px] font-black text-rose-500 bg-rose-50 px-3 py-1 rounded-full uppercase inline-block mb-1 border border-rose-100">{tabs.find(t=>t.id === log.meal)?.label}</span>
                    <h4 className="font-extrabold text-gray-800 text-lg leading-tight mt-1">{log.name}</h4>
                    {log.ai_data && <p className="text-xs font-black text-orange-500 mt-1 bg-orange-50 inline-block px-2 py-0.5 rounded-md">🔥 {log.ai_data.calories} kcal</p>}
                  </div>
                </div>
              ))}
            </div>
          </div>
        )}
      </div>
    </div>
  );
}

// --- Leaderboard Tab ---
function UserLeaderboardTab({ leaderboard, currentUserId, currentPoints }) {
  const userRank = leaderboard.find(u => u.id === currentUserId)?.rank || '-';
  const top5 = leaderboard.slice(0, 5);

  return (
    <div className="pt-12 pb-32 animate-fade-in bg-[#F4F5F7] min-h-screen relative overflow-hidden">
      {/* Background Decor */}
      <div className="absolute top-[-10%] left-[-10%] w-64 h-64 bg-emerald-200 rounded-full blur-3xl opacity-30"></div>
      <div className="absolute top-[20%] right-[-10%] w-64 h-64 bg-yellow-200 rounded-full blur-3xl opacity-30"></div>

      <div className="text-center px-6 mb-8 relative z-10">
        <h1 className="text-4xl font-black text-gray-900 tracking-tight drop-shadow-sm mb-3">กระดานผู้นำ</h1>
        <div className="bg-white inline-flex items-center gap-2 px-5 py-2 rounded-full shadow-sm border border-gray-100">
          <span className="animate-pulse text-lg">🏆</span>
          <p className="text-gray-600 font-bold text-sm">อัปเดตทุกสัปดาห์</p>
        </div>
      </div>

      <div className="px-6 space-y-4 relative z-10">
        {top5.map((user) => (
          <div key={user.id} className={`bg-white/95 backdrop-blur-sm p-4 rounded-3xl shadow-sm flex items-center relative overflow-hidden transition-all hover:scale-[1.02] border-2 ${user.id === currentUserId ? 'border-[#34A0A4]' : 'border-transparent'}`}>
            {user.rank === 1 && <div className="absolute top-0 left-0 w-2 h-full bg-gradient-to-b from-yellow-300 to-yellow-500"></div>}
            {user.rank === 2 && <div className="absolute top-0 left-0 w-2 h-full bg-gradient-to-b from-gray-300 to-gray-400"></div>}
            {user.rank === 3 && <div className="absolute top-0 left-0 w-2 h-full bg-gradient-to-b from-orange-300 to-orange-500"></div>}
            
            <div className={`w-10 font-black text-3xl text-center mr-2 ${user.rank <= 3 ? 'text-gray-800' : 'text-gray-300'}`}>{user.rank}</div>
            <div className="w-16 h-16 bg-gradient-to-tr from-gray-100 to-white rounded-full flex items-center justify-center text-4xl mr-4 overflow-hidden border-[3px] border-white shadow-md">
              {user.avatar?.startsWith('http') || user.avatar?.startsWith('data:') ? <img src={user.avatar} className="w-full h-full object-cover"/> : user.avatar || '👤'}
            </div>
            <div className="flex-1">
              <h4 className="font-extrabold text-gray-800 text-lg">{user.name} {user.id === currentUserId && <span className="text-xs text-white bg-[#34A0A4] px-2 py-0.5 rounded-full ml-1 shadow-sm">คุณ</span>}</h4>
              <p className="font-black text-gray-600 text-base mt-1">{(user.points||0).toLocaleString()} <span className="text-xs font-bold text-gray-400">แต้ม</span></p>
            </div>
            <div className="w-14 text-center drop-shadow-lg">
              {user.rank === 1 && <span className="text-5xl">🥇</span>}
              {user.rank === 2 && <span className="text-5xl">🥈</span>}
              {user.rank === 3 && <span className="text-5xl">🥉</span>}
            </div>
          </div>
        ))}
      </div>

      <div className="fixed bottom-[5.5rem] left-1/2 -translate-x-1/2 w-full max-w-md px-4 z-20 pb-4 pointer-events-none">
        <div className="bg-gradient-to-r from-[#1E3A8A] to-[#3B82F6] rounded-[2rem] p-4 flex items-center text-white shadow-[0_20px_40px_rgba(59,130,246,0.4)] pointer-events-auto border-4 border-white/20 backdrop-blur-md">
          <div className="w-16 h-16 bg-white rounded-full flex items-center justify-center text-4xl mr-4 overflow-hidden border-2 border-[#60A5FA] shadow-inner">
             {leaderboard.find(u=>u.id===currentUserId)?.avatar?.startsWith('http') || leaderboard.find(u=>u.id===currentUserId)?.avatar?.startsWith('data:') ? <img src={leaderboard.find(u=>u.id===currentUserId)?.avatar} className="w-full h-full object-cover"/> : leaderboard.find(u=>u.id===currentUserId)?.avatar || '👦🏻'}
          </div>
          <div className="flex-1 flex justify-between items-end">
            <div>
              <p className="text-xs text-blue-100 font-black uppercase tracking-wider">อันดับของคุณ</p>
              <h3 className="text-3xl font-black leading-none mt-1 drop-shadow-sm">ที่ {userRank}</h3>
            </div>
            <div className="text-right">
              <p className="text-xs text-blue-100 font-black uppercase tracking-wider">คะแนนรวม</p>
              <p className="font-black mt-1 text-xl text-yellow-300 drop-shadow-md">{(currentPoints||0).toLocaleString()}</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

// --- Profile & Rewards & Edit Profile ---
function ProfileTab({ profile, onLogout, onOpenRewards, onUpdateProfile }) {
  const [isEditing, setIsEditing] = useState(false);
  const [editName, setEditName] = useState(profile.name);
  const [editAvatar, setEditAvatar] = useState(profile.avatar);
  const fileRef = useRef(null);

  const handleAvatarChange = async (e) => {
    const file = e.target.files[0];
    if (file) {
      const base64 = await resizeImage(file, 200); 
      setEditAvatar(base64);
    }
  };

  const handleSaveProfile = () => {
    if(!editName.trim()) return alert("กรุณาใส่ชื่อ");
    onUpdateProfile(editName, editAvatar);
    setIsEditing(false);
  };

  return (
    <div className="p-6 pt-12 animate-fade-in bg-[#F4F5F7] min-h-screen">
      
      <div className="bg-white rounded-[2rem] p-6 shadow-sm mb-6 relative overflow-hidden border border-gray-50 flex flex-col items-center">
        <div className="absolute top-0 right-0 p-4 opacity-5 text-8xl">👑</div>
        <div className="w-28 h-28 rounded-full border-[4px] border-white shadow-xl overflow-hidden bg-gray-100 flex items-center justify-center text-5xl mb-4 relative z-10">
          {profile.avatar?.startsWith('http') || profile.avatar?.startsWith('data:') ? <img src={profile.avatar} className="w-full h-full object-cover" /> : profile.avatar || '👦🏻'}
        </div>
        <h2 className="text-3xl font-black text-gray-800 relative z-10">{profile.name}</h2>
        <p className="text-emerald-500 font-black bg-emerald-50 border border-emerald-100 px-4 py-1.5 rounded-full mt-2 text-sm shadow-sm">Level {profile.level || 1} Explorer</p>
        
        <button onClick={() => setIsEditing(true)} className="mt-5 bg-gray-100 text-gray-700 px-6 py-3 rounded-full font-black text-sm hover:bg-gray-200 transition-colors flex items-center gap-2 shadow-sm">
          <Edit size={16}/> แก้ไขโปรไฟล์ส่วนตัว
        </button>
      </div>

      {isEditing && (
        <div className="fixed inset-0 bg-black/60 z-50 flex items-center justify-center p-6 backdrop-blur-sm animate-fade-in">
          <div className="bg-white rounded-[2.5rem] p-8 w-full max-w-sm shadow-2xl relative border-4 border-white/20">
            <button onClick={() => setIsEditing(false)} className="absolute top-5 right-5 text-gray-400 hover:text-gray-800 bg-gray-100 p-2 rounded-full"><X size={20}/></button>
            <h3 className="text-2xl font-black text-gray-800 mb-6 text-center">ตั้งค่าโปรไฟล์</h3>
            
            <div className="flex flex-col items-center mb-6">
              <div className="w-28 h-28 rounded-full border-4 border-gray-100 overflow-hidden mb-3 relative group cursor-pointer shadow-inner" onClick={() => fileRef.current.click()}>
                 {editAvatar?.startsWith('http') || editAvatar?.startsWith('data:') ? <img src={editAvatar} className="w-full h-full object-cover" /> : <div className="w-full h-full flex items-center justify-center text-5xl">{editAvatar || '👦🏻'}</div>}
                 <div className="absolute inset-0 bg-black/50 flex items-center justify-center opacity-0 group-hover:opacity-100 transition"><Camera className="text-white" size={32}/></div>
              </div>
              <p className="text-xs font-black text-emerald-500 bg-emerald-50 px-3 py-1 rounded-full">แตะเพื่อเปลี่ยนรูป</p>
              <input type="file" accept="image/*" className="hidden" ref={fileRef} onChange={handleAvatarChange} />
            </div>

            <div className="space-y-4">
              <input type="text" value={editName} onChange={e=>setEditName(e.target.value)} placeholder="ชื่อแสดงผล" className="w-full bg-gray-50 border-2 border-transparent focus:border-emerald-400 rounded-2xl p-4 text-gray-800 font-black outline-none text-center text-lg transition-colors" />
              <button onClick={handleSaveProfile} className="w-full bg-gradient-to-r from-emerald-400 to-[#34A0A4] text-white font-black py-4 rounded-2xl shadow-[0_10px_20px_rgba(52,160,164,0.3)] hover:scale-[1.02] active:scale-95 transition-all text-lg mt-2">บันทึกข้อมูล</button>
            </div>
          </div>
        </div>
      )}

      <div className="text-center mb-6 mt-8">
        <h1 className="text-2xl font-black text-gray-900">สรุปผลรายสัปดาห์</h1>
      </div>

      <div className="space-y-6">
        <div className="bg-white rounded-[2rem] p-6 shadow-sm border border-gray-50 relative overflow-hidden">
          <div className="absolute top-0 right-0 p-4 opacity-5 text-6xl">📊</div>
          <h3 className="font-extrabold text-gray-800 text-lg mb-6 relative z-10">การประเมิน</h3>
          <div className="flex justify-around relative z-10">
            <EvaluationItem icon="🍬" label="หวาน" score={4} />
            <div className="w-px bg-gray-100"></div>
            <EvaluationItem icon="🧂" label="เค็ม" score={5} />
            <div className="w-px bg-gray-100"></div>
            <EvaluationItem icon="💧" label="ไขมัน" score={3} />
          </div>
        </div>

        <button onClick={onOpenRewards} className="w-full bg-gradient-to-r from-[#FF7A00] to-[#FF004D] text-white rounded-[2rem] py-4 text-xl font-black shadow-[0_10px_25px_rgba(255,122,0,0.3)] flex items-center justify-center gap-3 mt-4 hover:scale-[1.02] active:scale-95 transition-all">
          <span className="text-2xl drop-shadow-md">🎁</span> ร้านค้าแลกรางวัล
        </button>

        <button onClick={onLogout} className="w-full bg-white text-red-500 font-black border-2 border-red-50 rounded-[2rem] hover:bg-red-50 transition-colors py-4 flex justify-center items-center gap-2 shadow-sm">
           <LogOut size={20}/> ออกจากระบบ
        </button>
      </div>
    </div>
  );
}

function EvaluationItem({ icon, label, score }) {
  return (
    <div className="flex flex-col items-center">
      <span className="text-4xl mb-2 drop-shadow-sm">{icon}</span>
      <span className="font-extrabold text-gray-800 mb-2">{label}</span>
      <div className="flex gap-1">
        {[1, 2, 3, 4, 5].map(star => <Star key={star} size={14} className={star <= score ? 'fill-yellow-400 text-yellow-400 drop-shadow-sm' : 'fill-gray-100 text-gray-100'} />)}
      </div>
    </div>
  );
}

function RewardsModal({ onClose, points, rewards, onRedeem }) {
  const [tab, setTab] = useState('virtual');
  const virtualRewards = rewards.filter(r => r.type === 'virtual');
  const physicalRewards = rewards.filter(r => r.type === 'physical');

  return (
    <div className="absolute inset-0 bg-[#F4F5F7] z-50 flex flex-col animate-slide-up">
      <div className="pt-12 pb-4 px-6 bg-white/95 backdrop-blur-md border-b border-gray-100 flex items-center shadow-sm">
        <button onClick={onClose} className="p-3 -ml-2 text-gray-800 bg-gray-100 rounded-full hover:bg-gray-200 transition"><ChevronLeft size={24} /></button>
        <h1 className="text-xl font-black text-gray-800 flex-1 text-center pr-10">แลกของรางวัล</h1>
      </div>

      <div className="text-center py-8 bg-white border-b border-gray-100 mb-6 shadow-sm relative overflow-hidden">
        <div className="absolute top-0 right-0 p-4 opacity-5 text-8xl">🎁</div>
        <p className="text-gray-500 font-black uppercase tracking-wider mb-2 relative z-10">คะแนนคงเหลือของคุณ</p>
        <div className="flex items-center justify-center gap-3 relative z-10">
          <div className="w-12 h-12 bg-gradient-to-br from-yellow-200 to-yellow-400 rounded-full flex items-center justify-center text-2xl shadow-md border-2 border-white">🪙</div>
          <h2 className="text-6xl font-black text-gray-900 tracking-tight drop-shadow-sm">{(points||0).toLocaleString()}</h2>
        </div>
      </div>

      <div className="flex mx-6 bg-white rounded-2xl p-1.5 mb-6 shadow-sm border border-gray-100">
        <button onClick={() => setTab('virtual')} className={`flex-1 py-3 text-sm font-black rounded-xl transition-all ${tab === 'virtual' ? 'bg-[#FF7A00] text-white shadow-md' : 'text-gray-400 hover:bg-gray-50'}`}>ในเกม (Virtual)</button>
        <button onClick={() => setTab('physical')} className={`flex-1 py-3 text-sm font-black rounded-xl transition-all ${tab === 'physical' ? 'bg-[#FF7A00] text-white shadow-md' : 'text-gray-400 hover:bg-gray-50'}`}>ของจริง (Physical)</button>
      </div>

      <div className="flex-1 overflow-y-auto px-6 pb-10 grid grid-cols-2 gap-4 place-content-start">
        {(tab === 'virtual' ? virtualRewards : physicalRewards).map(r => (
           <div key={r.id} className="bg-white border border-gray-100 rounded-[2rem] p-4 shadow-sm flex flex-col items-center text-center hover:shadow-md transition">
             <div className={`w-full aspect-square ${tab === 'virtual' ? 'bg-gradient-to-br from-blue-50 to-indigo-50' : 'bg-gradient-to-br from-orange-50 to-red-50'} rounded-[1.5rem] flex items-center justify-center text-6xl mb-4 shadow-inner border border-white`}>{r.image}</div>
             <h4 className="font-extrabold text-gray-800 text-sm mb-auto leading-tight">{r.title}</h4>
             <button onClick={() => onRedeem(r)} className={`w-full mt-4 ${tab === 'virtual' ? 'bg-gradient-to-r from-blue-400 to-indigo-500' : 'bg-gradient-to-r from-orange-400 to-red-500'} text-white font-black py-3 rounded-xl text-xs shadow-md hover:opacity-90 active:scale-95 transition`}>
               แลกเลย {r.points.toLocaleString()}
             </button>
           </div>
        ))}
        {(tab === 'virtual' ? virtualRewards : physicalRewards).length === 0 && (
          <div className="col-span-2 text-center text-gray-400 font-bold py-10 bg-white rounded-3xl border border-dashed border-gray-200">ยังไม่มีของรางวัลในหมวดนี้</div>
        )}
      </div>
    </div>
  );
}

function LoadingScreen() {
  return <div className="flex h-screen items-center justify-center bg-[#F4F5F7]"><Activity className="text-[#FF004D] animate-pulse drop-shadow-md" size={64} strokeWidth={2.5} /></div>;
}


```
