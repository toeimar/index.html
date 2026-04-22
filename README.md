```react
import React, { useState, useEffect, useRef } from 'react';
import { 
  Home, Compass, ClipboardList, Trophy, User, 
  Camera, Plus, ChevronLeft, Search, CheckCircle, 
  Droplet, Footprints, Flame, Settings, Users, 
  Edit, Trash2, LogOut, Activity, Save, X, Lock, 
  Gift, Star, Sparkles, Image as ImageIcon,
  Mail, Key, Smartphone, AlertCircle, Eye, Calendar, Repeat
} from 'lucide-react';

// --- Firebase Imports ---
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, getDoc, collection, onSnapshot, updateDoc, addDoc, deleteDoc, query, where } from 'firebase/firestore';

// --- Configuration ---
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'vitalvibe-live-v11';
const geminiApiKey = ""; // Injected by environment

// 🔒 รหัสผ่านสำหรับ Admin (ห้ามโชว์บนหน้าเว็บ)
const ADMIN_PIN = "kira5642"; 

// --- Helpers ---
const getTodayStr = () => {
  const d = new Date();
  return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
};

const cleanEmail = (email) => email.trim().toLowerCase().replace(/[^a-z0-9]/g, '_');

// บีบอัดภาพก่อนส่งขึ้นระบบ
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
  const prompt = `คุณคือนักโภชนาการ AI วิเคราะห์ภาพอาหารนี้และตอบกลับเป็น JSON format เท่านั้น:
  {
    "name": "ชื่อเมนูอาหาร (ถ้าไม่ใช่อาหารให้ตอบ 'ไม่พบอาหาร')",
    "calories": ตัวเลขแคลอรี่โดยประมาณ (number),
    "sugar": "ต่ำ/ปานกลาง/สูง",
    "sodium": "ต่ำ/ปานกลาง/สูง",
    "fat": "ต่ำ/ปานกลาง/สูง",
    "advice": "คำแนะนำสั้นๆ 1 ประโยค"
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
    throw new Error("AI Analysis Failed");
  }
};

// --- Seed Data ---
const DEFAULT_MISSIONS = [
  { title: 'ลาขาดหวาน 1 วัน', subtitle: 'งดเครื่องดื่มและขนมหวานทุกชนิด', points: 20, category: 'เบาหวาน', maxPerDay: 1, expiresAt: '' },
  { title: 'เดิน 30 นาที', subtitle: 'เดินออกกำลังกายต่อเนื่อง 30 นาที', points: 30, category: 'ความดัน', maxPerDay: 1, expiresAt: '' },
  { title: 'กินผักผลไม้ 5 สี', subtitle: 'ทานผักและผลไม้ให้หลากหลาย', points: 25, category: 'หัวใจ', maxPerDay: 3, expiresAt: '' },
];
const DEFAULT_REWARDS = [
  { title: 'คูปองตรวจสุขภาพ รพ.', points: 5000, image: '🎟️', type: 'physical' },
  { title: 'ชุดของขวัญสุขภาพ', points: 3500, image: '🧺', type: 'physical' },
  { title: 'แก้วน้ำเก็บอุณหภูมิ', points: 1500, image: '🥤', type: 'physical' },
];

export default function App() {
  const [firebaseReady, setFirebaseReady] = useState(false);
  const [appUser, setAppUser] = useState(null);
  const [allProfiles, setAllProfiles] = useState([]);
  
  const [globalMissions, setGlobalMissions] = useState([]);
  const [globalRewards, setGlobalRewards] = useState([]);
  const [leaderboard, setLeaderboard] = useState([]);
  const [userProgress, setUserProgress] = useState([]); // Track mission completions
  
  // Custom Modals
  const [toast, setToast] = useState(null);
  const [confirmDialog, setConfirmDialog] = useState(null);
  const [promptDialog, setPromptDialog] = useState(null);

  const showToast = (message, type = 'success') => {
    setToast({ message, type });
    setTimeout(() => setToast(null), 3000);
  };

  // 1. Initialize DB Connection
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) await signInWithCustomToken(auth, __initial_auth_token);
        else await signInAnonymously(auth);
      } catch (err) { console.error("Firebase init:", err); }
    };
    initAuth();
    const unsub = onAuthStateChanged(auth, (user) => { if(user) setFirebaseReady(true); });
    return () => unsub();
  }, []);

  // 2. Fetch Data
  useEffect(() => {
    if (!firebaseReady) return;

    const missionsRef = collection(db, 'artifacts', appId, 'public', 'data', 'missions');
    const unsubM = onSnapshot(missionsRef, (snap) => {
      const loaded = snap.docs.map(d => ({ id: d.id, ...d.data() }));
      if (loaded.length === 0) DEFAULT_MISSIONS.forEach(m => addDoc(missionsRef, m));
      else setGlobalMissions(loaded);
    });

    const rewardsRef = collection(db, 'artifacts', appId, 'public', 'data', 'rewards');
    const unsubR = onSnapshot(rewardsRef, (snap) => {
      const loaded = snap.docs.map(d => ({ id: d.id, ...d.data() }));
      if (loaded.length === 0) DEFAULT_REWARDS.forEach(r => addDoc(rewardsRef, r));
      else setGlobalRewards(loaded);
    });

    const profilesRef = collection(db, 'artifacts', appId, 'public', 'data', 'profiles');
    const unsubP = onSnapshot(profilesRef, (snap) => {
      const loadedProfiles = snap.docs.map(d => d.data());
      setAllProfiles(loadedProfiles);
      
      const ranked = loadedProfiles.filter(u => u.role === 'user').sort((a,b) => b.points - a.points).map((u, i) => ({ ...u, rank: i + 1 }));
      setLeaderboard(ranked);

      // Restore session logic
      try {
        const savedSession = localStorage.getItem('vv_uid_v10');
        if (savedSession && !appUser) {
          const userObj = loadedProfiles.find(u => u.uid === savedSession);
          if (userObj) setAppUser(userObj);
        }
      } catch(e) {}

      // Keep appUser fresh
      setAppUser(current => {
        if (!current) return null;
        const updated = loadedProfiles.find(u => u.uid === current.uid);
        return updated || current;
      });
    });

    return () => { unsubM(); unsubR(); unsubP(); };
  }, [firebaseReady]);

  // Load User Progress (Missions)
  useEffect(() => {
    if (!appUser?.uid || appUser.role === 'admin') return;
    const progressRef = collection(db, 'artifacts', appId, 'public', 'data', `progress_${appUser.uid}`);
    const unsubProgress = onSnapshot(progressRef, (snapshot) => {
      setUserProgress(snapshot.docs.map(d => d.data()));
    });
    return () => unsubProgress();
  }, [appUser?.uid]);

  // --- Auth System ---
  const handleRegister = async (email, password, name, isAdminLogin, adminPin) => {
    if (isAdminLogin && adminPin !== ADMIN_PIN) {
      showToast('รหัสลับผู้ดูแลระบบไม่ถูกต้อง!', 'error');
      throw new Error('รหัสลับผู้ดูแลระบบไม่ถูกต้อง!');
    }
    
    const eid = cleanEmail(email);
    const authRef = doc(db, 'artifacts', appId, 'public', 'data', 'auth', eid);
    const docSnap = await getDoc(authRef);
    if (docSnap.exists()) {
      showToast('อีเมลนี้ถูกใช้งานแล้ว กรุณาล็อคอิน', 'error');
      throw new Error('อีเมลนี้ถูกใช้งานแล้ว');
    }

    const uid = 'USR_' + Date.now() + Math.random().toString(36).substring(2, 8);
    const role = isAdminLogin ? 'admin' : 'user';
    const avatar = 'https://api.dicebear.com/7.x/notionists/svg?seed=' + encodeURIComponent(name);

    await setDoc(authRef, { email: eid, password, uid, role });
    const profile = { uid, name, role, points: 0, level: 1, avatar };
    await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'profiles', uid), profile);

    setAppUser(profile);
    try { localStorage.setItem('vv_uid_v10', uid); } catch(e){}
    showToast('สร้างบัญชีสำเร็จ! ยินดีต้อนรับ 🎉');
  };

  const handleLogin = async (email, password) => {
    const eid = cleanEmail(email);
    const authRef = doc(db, 'artifacts', appId, 'public', 'data', 'auth', eid);
    const docSnap = await getDoc(authRef);
    
    if (!docSnap.exists()) {
      showToast('ไม่พบบัญชีนี้ กรุณาสมัครสมาชิก', 'error');
      throw new Error('ไม่พบบัญชีนี้');
    }
    const authData = docSnap.data();
    if (authData.password !== password) {
      showToast('รหัสผ่านไม่ถูกต้อง', 'error');
      throw new Error('รหัสผ่านไม่ถูกต้อง');
    }

    const profileSnap = await getDoc(doc(db, 'artifacts', appId, 'public', 'data', 'profiles', authData.uid));
    if (profileSnap.exists()) {
      const profile = profileSnap.data();
      setAppUser(profile);
      try { localStorage.setItem('vv_uid_v10', profile.uid); } catch(e){}
      showToast('เข้าสู่ระบบสำเร็จ! 🚀');
    }
  };

  const handleLogout = () => {
    setConfirmDialog({
      text: 'คุณต้องการออกจากระบบหรือไม่?',
      onConfirm: () => {
        setAppUser(null);
        try { localStorage.removeItem('vv_uid_v10'); } catch(e){}
        showToast('ออกจากระบบเรียบร้อย');
      }
    });
  };

  const handleUpdateProfile = async (newName, newAvatar) => {
    if (!appUser) return;
    const updates = { name: newName };
    if (newAvatar) updates.avatar = newAvatar;
    await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'profiles', appUser.uid), updates);
    showToast("อัปเดตโปรไฟล์สำเร็จ! 🌟");
  };

  if (!firebaseReady) return <LoadingScreen />;

  return (
    <div className="relative bg-gray-900 font-sans h-screen w-full flex justify-center">
      <div className="w-full max-w-md bg-white h-full relative overflow-hidden shadow-2xl flex flex-col">
        {!appUser ? (
          <AuthScreen onLogin={handleLogin} onRegister={handleRegister} />
        ) : appUser.role === 'admin' ? (
          <AdminApp 
            missions={globalMissions} rewards={globalRewards} leaderboard={leaderboard} 
            onLogout={handleLogout} db={db} appId={appId} 
            setConfirmDialog={setConfirmDialog} setPromptDialog={setPromptDialog} showToast={showToast}
          />
        ) : (
          <UserApp 
            profile={appUser} missions={globalMissions} rewards={globalRewards} leaderboard={leaderboard}
            userProgress={userProgress}
            onLogout={handleLogout} onUpdateProfile={handleUpdateProfile}
            db={db} appId={appId} setConfirmDialog={setConfirmDialog} showToast={showToast}
          />
        )}

        {/* Global Overlays */}
        {toast && (
          <div className="absolute top-10 left-1/2 -translate-x-1/2 z-[999] animate-slide-down">
            <div className={`px-6 py-3 rounded-full font-black shadow-2xl flex items-center gap-2 text-white tracking-wide border-2 ${toast.type === 'error' ? 'bg-red-500 border-red-400' : 'bg-gray-900 border-gray-700'}`}>
              {toast.type === 'error' ? <AlertCircle size={20}/> : <CheckCircle size={20}/>} {toast.message}
            </div>
          </div>
        )}

        {confirmDialog && (
          <div className="absolute inset-0 z-[1000] bg-black/60 backdrop-blur-md flex items-center justify-center p-6 animate-fade-in">
            <div className="bg-white rounded-[2.5rem] p-8 w-full text-center shadow-2xl border-[4px] border-white/20">
              <h3 className="text-2xl font-black text-gray-800 mb-8 leading-tight">{confirmDialog.text}</h3>
              <div className="flex gap-4">
                <button onClick={() => setConfirmDialog(null)} className="flex-1 py-4 bg-gray-100 text-gray-600 font-black rounded-[1.5rem] hover:bg-gray-200 transition">ยกเลิก</button>
                <button onClick={() => { confirmDialog.onConfirm(); setConfirmDialog(null); }} className="flex-1 py-4 bg-red-500 text-white font-black rounded-[1.5rem] shadow-[0_10px_25px_rgba(239,68,68,0.4)] hover:scale-[1.02] active:scale-95 transition">ยืนยัน</button>
              </div>
            </div>
          </div>
        )}

        {promptDialog && (
          <div className="absolute inset-0 z-[1000] bg-black/60 backdrop-blur-md flex items-center justify-center p-6 animate-fade-in">
            <div className="bg-white rounded-[2.5rem] p-8 w-full shadow-2xl border-[4px] border-white/20">
              <h3 className="text-2xl font-black text-gray-800 mb-2">{promptDialog.title}</h3>
              <p className="text-sm font-bold text-gray-500 mb-6">{promptDialog.description}</p>
              <input 
                type="number" autoFocus id="prompt-input" defaultValue={promptDialog.defaultValue} 
                className="w-full bg-gray-50 border-2 border-emerald-400 rounded-2xl p-4 text-center text-3xl font-black text-emerald-600 outline-none mb-6 shadow-inner" 
              />
              <div className="flex gap-4">
                <button onClick={() => setPromptDialog(null)} className="flex-1 py-4 bg-gray-100 text-gray-600 font-black rounded-[1.5rem] hover:bg-gray-200 transition">ยกเลิก</button>
                <button onClick={() => { 
                  const val = document.getElementById('prompt-input').value;
                  promptDialog.onConfirm(val); 
                  setPromptDialog(null); 
                }} className="flex-1 py-4 bg-gradient-to-r from-emerald-400 to-teal-500 text-white font-black rounded-[1.5rem] shadow-[0_10px_25px_rgba(16,185,129,0.4)] hover:scale-[1.02] active:scale-95 transition">บันทึก</button>
              </div>
            </div>
          </div>
        )}
      </div>
    </div>
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
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    try {
      if (isLogin) await onLogin(email, password);
      else {
        if (!name.trim()) throw new Error('กรุณากรอกชื่อของคุณ');
        await onRegister(email, password, name, isAdminLogin, adminPin);
      }
    } catch (err) {
      // Handled by toast in main component
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="h-full w-full bg-gradient-to-br from-[#FF9A9E] via-[#FECFEF] to-[#A1C4FD] p-6 relative overflow-y-auto">
      <div className="absolute top-[-5%] left-[-10%] w-64 h-64 bg-orange-300 rounded-full blur-3xl opacity-50 pointer-events-none"></div>
      <div className="absolute bottom-[10%] right-[-10%] w-64 h-64 bg-blue-300 rounded-full blur-3xl opacity-50 pointer-events-none"></div>

      <div className="bg-white/95 backdrop-blur-xl rounded-[3rem] p-8 shadow-[0_20px_50px_rgba(0,0,0,0.1)] text-center border-[4px] border-white relative z-10 mt-6">
        <div className="w-24 h-24 bg-gradient-to-tr from-[#FF7A00] to-[#FF004D] text-white rounded-[2.5rem] shadow-[0_15px_30px_rgba(255,0,77,0.3)] flex items-center justify-center mx-auto mb-6 transform rotate-3">
          <Activity size={48} strokeWidth={3} />
        </div>
        <h1 className="text-4xl font-black text-transparent bg-clip-text bg-gradient-to-r from-[#FF7A00] to-[#FF004D] tracking-tight mb-2">VitalVibe</h1>
        <p className="text-gray-500 font-bold mb-8">คอมมูนิตี้คนรักสุขภาพ 🌟</p>

        <form onSubmit={handleSubmit} className="space-y-4 text-left">
          <div className="flex bg-gray-100 p-1.5 rounded-[1.2rem] mb-6 shadow-inner">
            <button type="button" onClick={() => setIsLogin(true)} className={`flex-1 py-3.5 text-sm font-black rounded-xl transition-all ${isLogin ? 'bg-white text-[#FF004D] shadow-md' : 'text-gray-400 hover:text-gray-600'}`}>เข้าสู่ระบบ</button>
            <button type="button" onClick={() => setIsLogin(false)} className={`flex-1 py-3.5 text-sm font-black rounded-xl transition-all ${!isLogin ? 'bg-white text-[#FF004D] shadow-md' : 'text-gray-400 hover:text-gray-600'}`}>สร้างบัญชีใหม่</button>
          </div>

          {!isLogin && (
            <div className="relative animate-fade-in">
              <User className="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400" size={20} />
              <input type="text" placeholder="ชื่อที่ใช้แสดงผล" required value={name} onChange={e=>setName(e.target.value)} className="w-full bg-gray-50 border-2 border-transparent focus:border-[#FF7A00] rounded-2xl p-4 pl-12 text-gray-800 font-bold outline-none transition" />
            </div>
          )}

          <div className="relative">
            <Mail className="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400" size={20} />
            <input type="email" placeholder="อีเมล (E-mail)" required value={email} onChange={e=>setEmail(e.target.value)} className="w-full bg-gray-50 border-2 border-transparent focus:border-[#FF7A00] rounded-2xl p-4 pl-12 text-gray-800 font-bold outline-none transition" />
          </div>

          <div className="relative">
            <Key className="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400" size={20} />
            <input type="password" placeholder="รหัสผ่าน (Password)" required minLength="6" value={password} onChange={e=>setPassword(e.target.value)} className="w-full bg-gray-50 border-2 border-transparent focus:border-[#FF7A00] rounded-2xl p-4 pl-12 text-gray-800 font-bold outline-none transition" />
          </div>

          {!isLogin && (
            <label className="flex items-center justify-center gap-2 text-sm text-gray-400 font-bold cursor-pointer pt-2">
              <input type="checkbox" checked={isAdminLogin} onChange={(e) => setIsAdminLogin(e.target.checked)} className="w-4 h-4 rounded text-orange-500" />
              สมัครในฐานะผู้ดูแลระบบ (Admin)
            </label>
          )}

          {!isLogin && isAdminLogin && (
            <div className="relative animate-fade-in pb-2">
              <Lock className="absolute left-4 top-1/2 -translate-y-1/2 text-red-400" size={20} />
              <input type="password" placeholder="รหัสลับแอดมิน" required value={adminPin} onChange={e=>setAdminPin(e.target.value)} className="w-full bg-red-50 border-2 border-red-200 focus:border-red-500 rounded-2xl p-4 pl-12 text-red-800 font-black outline-none tracking-widest" />
            </div>
          )}
          
          <button type="submit" disabled={loading} className="w-full bg-gradient-to-r from-[#FF7A00] to-[#FF004D] text-white rounded-2xl py-4 text-lg font-black shadow-[0_10px_30px_rgba(255,0,77,0.4)] mt-2 hover:scale-[1.02] disabled:opacity-50 active:scale-95 transition-all">
            {loading ? 'กำลังตรวจสอบ...' : (isLogin ? 'เข้าสู่ระบบ 🚀' : 'เริ่มใช้งานฟรี ✨')}
          </button>
        </form>
      </div>
    </div>
  );
}

// ==========================================
// 2. ADMIN APP 
// ==========================================
function AdminApp({ missions, rewards, leaderboard, onLogout, db, appId, setConfirmDialog, setPromptDialog, showToast }) {
  const [activeTab, setActiveTab] = useState('users');

  return (
    <div className="h-full flex flex-col bg-gray-50">
      <header className="bg-gray-900 text-white px-6 pt-12 pb-6 shadow-xl rounded-b-[2.5rem] z-10 relative overflow-hidden">
        <div className="absolute top-[-50%] right-[-10%] w-32 h-32 bg-emerald-500 rounded-full blur-3xl opacity-20"></div>
        <div className="flex justify-between items-center relative z-10">
          <div>
            <h1 className="text-2xl font-black text-transparent bg-clip-text bg-gradient-to-r from-emerald-400 to-teal-400 tracking-tight">Admin Control</h1>
            <p className="text-sm font-bold text-gray-400 mt-1">จัดการระบบทั้งหมด</p>
          </div>
          <button onClick={onLogout} className="w-12 h-12 bg-gray-800 border border-gray-700 rounded-full flex items-center justify-center hover:bg-gray-700 transition shadow-inner"><LogOut size={20} className="text-red-400" /></button>
        </div>
      </header>

      <main className="flex-1 overflow-y-auto p-6 pb-24">
        {activeTab === 'users' && <AdminUsers leaderboard={leaderboard} db={db} appId={appId} setPromptDialog={setPromptDialog} showToast={showToast} />}
        {activeTab === 'missions' && <AdminMissions missions={missions} db={db} appId={appId} setConfirmDialog={setConfirmDialog} showToast={showToast} />}
        {activeTab === 'rewards' && <AdminRewards rewards={rewards} db={db} appId={appId} setConfirmDialog={setConfirmDialog} showToast={showToast} />}
      </main>

      <nav className="absolute bottom-0 w-full bg-white/95 backdrop-blur-md border-t border-gray-100 flex justify-around items-center py-3 pb-safe shadow-[0_-10px_30px_rgba(0,0,0,0.05)] z-20">
        <NavItem icon={<Users size={24} />} label="ผู้ใช้งาน" active={activeTab === 'users'} onClick={() => setActiveTab('users')} color="text-blue-500" />
        <NavItem icon={<ClipboardList size={24} />} label="ภารกิจ" active={activeTab === 'missions'} onClick={() => setActiveTab('missions')} color="text-emerald-500" />
        <NavItem icon={<Gift size={24} />} label="รางวัล" active={activeTab === 'rewards'} onClick={() => setActiveTab('rewards')} color="text-orange-500" />
      </nav>
    </div>
  );
}

function AdminUsers({ leaderboard, db, appId, setPromptDialog, showToast }) {
  const [viewingUser, setViewingUser] = useState(null); 

  const handleEditPoints = (user) => {
    setPromptDialog({
      title: 'แก้ไขคะแนนสะสม',
      description: `ปรับคะแนนของ: ${user.name}`,
      defaultValue: user.points || 0,
      onConfirm: async (newVal) => {
        if (!isNaN(newVal) && newVal !== '') {
          await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'profiles', user.uid), { points: Number(newVal) });
          showToast('อัปเดตคะแนนเรียบร้อยแล้ว!');
        }
      }
    });
  };

  return (
    <div className="animate-fade-in space-y-4">
      <h2 className="text-2xl font-black text-gray-800">จัดการผู้ใช้งาน</h2>
      <p className="text-gray-500 font-bold text-sm bg-blue-50 p-4 rounded-2xl border border-blue-100 shadow-sm leading-relaxed">
        💡 กด <Eye size={14} className="inline text-blue-500"/> ดูข้อมูลสุขภาพ<br/>
        💡 กด <Edit size={14} className="inline text-emerald-500"/> ปรับเพิ่มลดคะแนน
      </p>
      
      <div className="bg-white rounded-[2rem] shadow-sm border border-gray-100 overflow-hidden">
        {leaderboard.map((user, idx) => (
          <div key={user.uid} className="p-4 border-b border-gray-50 last:border-0 flex items-center justify-between hover:bg-gray-50 transition">
            <div className="flex items-center gap-3">
              <div className="w-6 text-center text-gray-300 font-black text-lg">{idx + 1}</div>
              {user.avatar?.startsWith('http') || user.avatar?.startsWith('data:') ? <img src={user.avatar} className="w-12 h-12 rounded-[1rem] object-cover border border-gray-200 shadow-sm"/> : <div className="w-12 h-12 bg-gray-100 rounded-[1rem] flex items-center justify-center text-2xl shadow-inner">{user.avatar || '👤'}</div>}
              <div>
                <h4 className="font-extrabold text-gray-800 text-base leading-tight max-w-[100px] truncate">{user.name}</h4>
                <p className="text-[10px] font-bold text-emerald-500 bg-emerald-50 px-2 py-0.5 rounded-full inline-block mt-1">Lv.{user.level || 1}</p>
              </div>
            </div>
            <div className="flex items-center gap-2">
               <div className="text-right mr-1">
                 <div className="font-black text-blue-600 text-lg">{user.points?.toLocaleString() || 0}</div>
                 <div className="text-[9px] font-black text-gray-400 uppercase tracking-wider">Points</div>
               </div>
               <button onClick={() => setViewingUser(user)} className="w-10 h-10 flex items-center justify-center bg-blue-50 text-blue-500 hover:text-white hover:bg-blue-500 rounded-[1rem] shadow-sm transition active:scale-95">
                 <Eye size={16}/>
               </button>
               <button onClick={() => handleEditPoints(user)} className="w-10 h-10 flex items-center justify-center bg-emerald-50 text-emerald-500 hover:text-white hover:bg-emerald-500 rounded-[1rem] shadow-sm transition active:scale-95">
                 <Edit size={16}/>
               </button>
            </div>
          </div>
        ))}
      </div>

      {viewingUser && <UserHealthModal user={viewingUser} onClose={() => setViewingUser(null)} db={db} appId={appId} />}
    </div>
  );
}

// 🏥 Modal สำหรับแอดมินดูข้อมูลสุขภาพ 
function UserHealthModal({ user, onClose, db, appId }) {
  const [stats, setStats] = useState({ water: 0, steps: 0 });
  const [logs, setLogs] = useState([]);
  const todayStr = getTodayStr();

  useEffect(() => {
    const statsRef = doc(db, 'artifacts', appId, 'public', 'data', `stats_${user.uid}`, todayStr);
    const unsubStats = onSnapshot(statsRef, (docSnap) => {
      if (docSnap.exists()) setStats(docSnap.data());
    });

    const logsRef = collection(db, 'artifacts', appId, 'public', 'data', `logs_${user.uid}`);
    const unsubLogs = onSnapshot(logsRef, (snapshot) => {
      const allLogs = snapshot.docs.map(d => ({id: d.id, ...d.data()}));
      setLogs(allLogs.filter(l => l.date === todayStr).sort((a,b) => b.timestamp - a.timestamp));
    });

    return () => { unsubStats(); unsubLogs(); };
  }, [user.uid, db, appId, todayStr]);

  return (
    <div className="fixed inset-0 z-[2000] bg-black/60 backdrop-blur-sm flex items-center justify-center p-6 animate-fade-in">
      <div className="bg-white rounded-[2rem] w-full max-h-[85vh] flex flex-col shadow-2xl border-[4px] border-white/20">
        <div className="p-5 border-b border-gray-100 flex justify-between items-center bg-gray-50 rounded-t-[1.8rem]">
          <h3 className="text-lg font-black text-gray-800 flex items-center gap-2"><Activity size={18} className="text-blue-500"/> ข้อมูลสุขภาพวันนี้</h3>
          <button onClick={onClose} className="p-2 bg-gray-200 rounded-full text-gray-500 hover:bg-gray-300 transition"><X size={16}/></button>
        </div>
        
        <div className="p-6 overflow-y-auto">
          <div className="flex items-center gap-4 mb-6">
            {user.avatar?.startsWith('http') || user.avatar?.startsWith('data:') ? <img src={user.avatar} className="w-14 h-14 rounded-[1rem] object-cover border shadow-sm"/> : <div className="w-14 h-14 bg-gray-100 rounded-[1rem] flex items-center justify-center text-2xl shadow-inner">{user.avatar}</div>}
            <div>
              <h4 className="font-extrabold text-gray-800 text-lg">{user.name}</h4>
              <p className="text-xs font-bold text-gray-500">แต้มสะสม: <span className="text-orange-500">{user.points} Pts</span></p>
            </div>
          </div>

          <h4 className="font-bold text-gray-800 mb-3 flex items-center gap-2">สถิติวันนี้ ({todayStr})</h4>
          <div className="grid grid-cols-2 gap-3 mb-6">
            <div className="bg-cyan-50 p-4 rounded-[1.5rem] border border-cyan-100 text-center">
              <Droplet size={24} className="text-[#34A0A4] mx-auto mb-2"/>
              <p className="text-[10px] font-black text-gray-400 uppercase">ดื่มน้ำ (แก้ว)</p>
              <p className="font-black text-2xl text-[#34A0A4]">{stats.water}<span className="text-sm">/8</span></p>
            </div>
            <div className="bg-rose-50 p-4 rounded-[1.5rem] border border-rose-100 text-center">
              <Footprints size={24} className="text-rose-500 mx-auto mb-2"/>
              <p className="text-[10px] font-black text-gray-400 uppercase">ก้าวเดิน</p>
              <p className="font-black text-2xl text-rose-500">{stats.steps}</p>
            </div>
          </div>

          <h4 className="font-bold text-gray-800 mb-3">บันทึกอาหารวันนี้</h4>
          <div className="space-y-3">
            {logs.map(log => (
              <div key={log.id} className="bg-gray-50 border border-gray-100 rounded-[1.5rem] p-3 flex gap-3 shadow-sm items-center">
                <div className="w-12 h-12 bg-white rounded-xl overflow-hidden flex-shrink-0 border">
                  {log.image ? <img src={log.image} className="w-full h-full object-cover" /> : <div className="w-full h-full flex items-center justify-center text-xl">🍽️</div>}
                </div>
                <div className="flex-1">
                  <h5 className="font-extrabold text-gray-800 text-sm leading-tight">{log.name}</h5>
                  {log.ai_data && <p className="text-[10px] font-black text-orange-500 mt-1 bg-orange-50 inline-block px-2 py-0.5 rounded-md border border-orange-100">🔥 {log.ai_data.calories} kcal</p>}
                </div>
              </div>
            ))}
            {logs.length === 0 && <p className="text-center text-xs font-bold text-gray-400 py-4">ยังไม่มีการบันทึกอาหาร</p>}
          </div>
        </div>
      </div>
    </div>
  );
}

function AdminMissions({ missions, db, appId, setConfirmDialog, showToast }) {
  const [isAdding, setIsAdding] = useState(false);
  const [form, setForm] = useState({ title: '', subtitle: '', category: 'เบาหวาน', points: 10, maxPerDay: 1, expiresAt: '' });

  const handleSave = async () => {
    if (!form.title.trim()) return showToast('กรุณากรอกชื่อภารกิจ', 'error');
    
    // Add logic to save mission
    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'missions'), { 
      ...form, requiresPhoto: true 
    });
    
    setIsAdding(false); 
    setForm({ title: '', subtitle: '', category: 'เบาหวาน', points: 10, maxPerDay: 1, expiresAt: '' });
    showToast('เพิ่มภารกิจสำเร็จ');
  };

  const handleDelete = (id) => {
    setConfirmDialog({ text: 'ลบภารกิจนี้ออกจากระบบใช่หรือไม่?', onConfirm: async () => {
        await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id));
        showToast('ลบภารกิจแล้ว');
    }});
  };

  return (
    <div className="animate-fade-in space-y-6 pb-6">
      <div className="flex justify-between items-center bg-white p-5 rounded-[2rem] shadow-sm border border-gray-100">
        <h2 className="text-xl font-black text-gray-800">จัดการภารกิจ</h2>
        {!isAdding && <button onClick={() => setIsAdding(true)} className="bg-emerald-500 text-white px-5 py-2.5 rounded-xl flex gap-2 font-black shadow-[0_5px_15px_rgba(16,185,129,0.3)] hover:scale-105 active:scale-95 transition"><Plus size={18} /> สร้างใหม่</button>}
      </div>

      {isAdding && (
        <div className="bg-white p-6 rounded-[2.5rem] shadow-xl border-[4px] border-emerald-100 space-y-4 animate-slide-down">
          <div className="flex justify-between items-center"><h3 className="font-black text-lg text-gray-800">ภารกิจใหม่</h3><button onClick={()=>setIsAdding(false)} className="bg-gray-100 p-2 rounded-full text-gray-500"><X size={16}/></button></div>
          <input type="text" placeholder="ชื่อภารกิจหลัก" value={form.title} onChange={e=>setForm({...form, title: e.target.value})} className="w-full border-2 border-gray-100 focus:border-emerald-400 p-4 rounded-xl bg-gray-50 font-bold outline-none transition" />
          <input type="text" placeholder="คำอธิบาย/เงื่อนไข" value={form.subtitle} onChange={e=>setForm({...form, subtitle: e.target.value})} className="w-full border-2 border-gray-100 focus:border-emerald-400 p-4 rounded-xl bg-gray-50 font-bold outline-none transition" />
          
          <div className="grid grid-cols-2 gap-3">
            <div>
              <label className="text-[10px] font-black text-gray-500 uppercase ml-2">หมวดหมู่</label>
              <select value={form.category} onChange={e=>setForm({...form, category: e.target.value})} className="w-full border-2 border-gray-100 p-4 rounded-xl bg-gray-50 font-bold outline-none">
                <option value="เบาหวาน">เบาหวาน</option><option value="ความดัน">ความดัน</option><option value="หัวใจ">หัวใจ</option>
              </select>
            </div>
            <div>
              <label className="text-[10px] font-black text-gray-500 uppercase ml-2">แต้มที่จะได้</label>
              <input type="number" placeholder="แต้ม" value={form.points} onChange={e=>setForm({...form, points: Number(e.target.value)})} className="w-full border-2 border-gray-100 p-4 rounded-xl bg-gray-50 font-black text-emerald-600 text-center outline-none" />
            </div>
          </div>

          <div className="grid grid-cols-2 gap-3">
            <div>
              <label className="text-[10px] font-black text-gray-500 uppercase ml-2">ทำได้กี่ครั้งต่อวัน?</label>
              <div className="relative">
                <Repeat className="absolute left-3 top-1/2 -translate-y-1/2 text-gray-400" size={16} />
                <input type="number" min="1" value={form.maxPerDay} onChange={e=>setForm({...form, maxPerDay: Number(e.target.value)})} className="w-full border-2 border-gray-100 p-4 pl-10 rounded-xl bg-gray-50 font-bold outline-none" />
              </div>
            </div>
            <div>
              <label className="text-[10px] font-black text-gray-500 uppercase ml-2">วันหมดอายุ (ถ้ามี)</label>
              <div className="relative">
                <Calendar className="absolute left-3 top-1/2 -translate-y-1/2 text-gray-400" size={16} />
                <input type="date" value={form.expiresAt} onChange={e=>setForm({...form, expiresAt: e.target.value})} className="w-full border-2 border-gray-100 p-4 pl-10 rounded-xl bg-gray-50 font-bold outline-none text-xs" />
              </div>
            </div>
          </div>

          <button onClick={handleSave} className="w-full bg-gray-900 text-white font-black py-4 rounded-xl flex justify-center items-center gap-2 hover:bg-gray-800 shadow-md mt-2">บันทึกขึ้นระบบ</button>
        </div>
      )}

      <div className="space-y-4">
        {missions.map(m => (
          <div key={m.id} className="bg-white p-5 rounded-[2rem] shadow-sm border border-gray-100 flex justify-between items-center group hover:border-emerald-200 transition-colors">
            <div>
              <span className={`text-[10px] font-black px-3 py-1 rounded-full text-white shadow-sm inline-block mb-2 ${m.category === 'เบาหวาน' ? 'bg-red-400' : m.category === 'ความดัน' ? 'bg-blue-400' : 'bg-emerald-400'}`}>{m.category}</span>
              <h4 className="font-extrabold text-gray-800 text-lg leading-tight">{m.title}</h4>
              <div className="flex items-center gap-3 mt-1">
                <p className="font-black text-orange-500 text-sm">🪙 {m.points} Pts</p>
                <p className="text-xs text-gray-400 font-bold flex items-center gap-1"><Repeat size={12}/> {m.maxPerDay || 1}/วัน</p>
                {m.expiresAt && <p className="text-[10px] text-red-400 font-bold bg-red-50 px-2 py-0.5 rounded-md flex items-center gap-1"><Calendar size={10}/> {m.expiresAt}</p>}
              </div>
            </div>
            <button onClick={() => handleDelete(m.id)} className="w-12 h-12 bg-red-50 text-red-500 rounded-2xl flex items-center justify-center hover:bg-red-500 hover:text-white transition shadow-sm"><Trash2 size={20}/></button>
          </div>
        ))}
      </div>
    </div>
  );
}

function AdminRewards({ rewards, db, appId, setConfirmDialog, showToast }) {
  const [isAdding, setIsAdding] = useState(false);
  const [form, setForm] = useState({ title: '', points: 500, image: '🎁', type: 'physical' });

  const handleSave = async () => {
    if (!form.title.trim()) return showToast('กรุณากรอกชื่อรางวัล', 'error');
    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'rewards'), form);
    setIsAdding(false); setForm({ title: '', points: 500, image: '🎁', type: 'physical' });
    showToast('เพิ่มของรางวัลสำเร็จ');
  };

  const handleDelete = (id) => {
    setConfirmDialog({ text: 'ลบของรางวัลนี้ใช่หรือไม่?', onConfirm: async () => {
        await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'rewards', id));
        showToast('ลบของรางวัลแล้ว');
    }});
  };

  return (
    <div className="animate-fade-in space-y-6 pb-6">
      <div className="flex justify-between items-center bg-white p-5 rounded-[2rem] shadow-sm border border-gray-100">
        <h2 className="text-xl font-black text-gray-800">จัดการของรางวัล</h2>
        {!isAdding && <button onClick={() => setIsAdding(true)} className="bg-orange-500 text-white px-5 py-2.5 rounded-xl flex gap-2 font-black shadow-[0_5px_15px_rgba(249,115,22,0.3)] hover:scale-105 active:scale-95 transition"><Plus size={18} /> เพิ่มใหม่</button>}
      </div>

      {isAdding && (
        <div className="bg-white p-6 rounded-[2.5rem] shadow-xl border-t-4 border-orange-500 space-y-4 animate-slide-down">
          <div className="flex justify-between items-center"><h3 className="font-black text-lg text-gray-800">สร้างรางวัลใหม่</h3><button onClick={()=>setIsAdding(false)} className="bg-gray-100 p-2 rounded-full text-gray-500"><X size={16}/></button></div>
          <input type="text" placeholder="ชื่อของรางวัล" value={form.title} onChange={e=>setForm({...form, title: e.target.value})} className="w-full border-2 border-gray-100 focus:border-orange-400 p-4 rounded-xl bg-gray-50 font-bold outline-none transition" />
          <div className="flex gap-3">
            <select value={form.type} onChange={e=>setForm({...form, type: e.target.value})} className="flex-1 border-2 border-gray-100 focus:border-orange-400 p-4 rounded-xl bg-gray-50 font-bold outline-none text-gray-700">
              <option value="physical">ของจริง (Physical)</option><option value="virtual">ในเกม (Virtual)</option>
            </select>
            <input type="text" placeholder="Emoji ไอคอน" value={form.image} onChange={e=>setForm({...form, image: e.target.value})} className="w-24 border-2 border-gray-100 focus:border-orange-400 p-4 rounded-xl bg-gray-50 text-center text-2xl outline-none" />
          </div>
          <input type="number" placeholder="ใช้แต้มเท่าไหร่?" value={form.points} onChange={e=>setForm({...form, points: Number(e.target.value)})} className="w-full border-2 border-gray-100 focus:border-orange-400 p-4 rounded-xl bg-gray-50 font-black text-orange-600 outline-none" />
          <button onClick={handleSave} className="w-full bg-gray-900 text-white font-black py-4 rounded-xl flex justify-center items-center gap-2 hover:bg-gray-800 shadow-md">บันทึกขึ้นระบบ</button>
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

// ==========================================
// 3. USER APP
// ==========================================
function UserApp({ profile, missions, rewards, leaderboard, userProgress, onLogout, onUpdateProfile, db, appId, setConfirmDialog, showToast }) {
  const [activeTab, setActiveTab] = useState('home');
  const [showRewardsModal, setShowRewardsModal] = useState(false);
  const [successModal, setSuccessModal] = useState(null);

  useEffect(() => { window.scrollTo(0, 0); }, [activeTab]);

  const completeMission = async (mission, base64Image) => {
    const todayStr = getTodayStr();
    // Count how many times this user completed this mission today
    const completedToday = userProgress.filter(p => p.missionId === mission.id && p.date === todayStr).length;
    const maxAllowed = mission.maxPerDay || 1;

    if (completedToday >= maxAllowed) {
      return showToast(`ภารกิจนี้ทำครบโควต้าของวันนี้แล้ว (${maxAllowed}/${maxAllowed} ครั้ง)`, 'error');
    }
    
    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', `progress_${profile.uid}`), {
      missionId: mission.id, date: todayStr, timestamp: Date.now(), image: base64Image 
    });

    const newPoints = (profile.points || 0) + mission.points;
    await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'profiles', profile.uid), { points: newPoints });
    showToast(`เยี่ยมมาก! ยืนยันภาพสำเร็จ ได้รับ ${mission.points} แต้ม 🎉`);
  };

  const redeemReward = (reward) => {
    if ((profile.points || 0) < reward.points) return showToast('คะแนนไม่พอนะ ลุยทำภารกิจเพิ่มเลย! 🏃‍♂️', 'error');
    
    setConfirmDialog({
      text: `ยืนยันแลก "${reward.title}" ด้วย ${reward.points} แต้ม?`,
      onConfirm: async () => {
        const newPoints = profile.points - reward.points;
        await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'profiles', profile.uid), { points: newPoints });
        // โชว์หน้าต่างแคปจอ
        setSuccessModal(reward);
        setShowRewardsModal(false);
      }
    });
  };

  return (
    <div className="h-full flex flex-col bg-[#F4F7F6]">
      <main className="flex-1 overflow-y-auto pb-24">
        {!showRewardsModal && activeTab === 'home' && <HomeTab profile={profile} db={db} appId={appId} onLogMeal={() => setActiveTab('log')} showToast={showToast} />}
        {!showRewardsModal && activeTab === 'missions' && <UserMissionsTab missions={missions} userProgress={userProgress} onComplete={completeMission} />}
        {!showRewardsModal && activeTab === 'log' && <FoodLogTab profile={profile} db={db} appId={appId} showToast={showToast} />}
        {!showRewardsModal && activeTab === 'leaderboard' && <UserLeaderboardTab leaderboard={leaderboard} currentUserId={profile.uid} />}
        {!showRewardsModal && activeTab === 'profile' && <ProfileTab profile={profile} onLogout={onLogout} onOpenRewards={() => setShowRewardsModal(true)} onUpdateProfile={onUpdateProfile} showToast={showToast} />}
        
        {showRewardsModal && <RewardsModal onClose={() => setShowRewardsModal(false)} points={profile.points} rewards={rewards} onRedeem={redeemReward} />}
      </main>

      {/* 🎉 Popup LINE Evidence (ไม่มี ID Line) */}
      {successModal && (
        <div className="absolute inset-0 z-[100] bg-black/80 flex items-center justify-center p-6 backdrop-blur-sm animate-fade-in">
          <div className="bg-white rounded-[3rem] p-8 text-center w-full border-[6px] border-white shadow-[0_0_50px_rgba(0,0,0,0.3)] relative overflow-hidden">
             <div className="absolute top-[-20%] left-[-20%] w-48 h-48 bg-orange-300 rounded-full blur-3xl opacity-50 pointer-events-none"></div>
             <div className="absolute bottom-[-20%] right-[-20%] w-48 h-48 bg-green-300 rounded-full blur-3xl opacity-50 pointer-events-none"></div>
             
             <div className="text-7xl mb-4 relative z-10 animate-bounce drop-shadow-md">🎉</div>
             <h2 className="text-4xl font-black text-transparent bg-clip-text bg-gradient-to-r from-orange-500 to-red-500 mb-2 relative z-10 tracking-tight">ยินดีด้วย!</h2>
             <p className="text-gray-800 font-black text-lg mb-2 relative z-10">คุณได้ทำการแลกรางวัล:</p>
             <p className="text-2xl font-black text-gray-900 mb-6 bg-white/80 py-4 rounded-2xl border border-gray-100 relative z-10 shadow-sm">{successModal.image} {successModal.title}</p>
             
             <div className="bg-gradient-to-br from-orange-50 to-orange-100 p-6 rounded-[2rem] border border-orange-200 mb-6 relative z-10 shadow-inner">
                <div className="flex justify-center mb-3"><Smartphone className="text-orange-500" size={40}/></div>
                <p className="text-orange-800 font-black text-lg leading-tight">📸 กรุณาแคปหน้าจอนี้</p>
                <div className="mt-4 bg-[#06C755] py-4 rounded-[1.2rem] shadow-[0_5px_15px_rgba(6,199,85,0.4)] flex justify-center items-center hover:scale-105 transition cursor-pointer">
                   <span className="text-white font-black text-lg tracking-wide">ส่งหลักฐานผ่าน LINE</span>
                </div>
             </div>
             
             <button onClick={() => setSuccessModal(null)} className="w-full bg-gray-900 text-white font-black py-4 rounded-2xl hover:bg-gray-800 active:scale-95 transition relative z-10 shadow-md text-lg">
               รับทราบ ปิดหน้าต่าง
             </button>
          </div>
        </div>
      )}

      {!showRewardsModal && (
        <nav className="absolute bottom-0 w-full bg-white/95 backdrop-blur-xl border-t border-gray-100 flex justify-between items-center py-2 px-6 z-20 pb-safe shadow-[0_-15px_40px_rgba(0,0,0,0.05)]">
          <NavItem icon={<Home size={24} />} label="หน้าแรก" active={activeTab === 'home'} onClick={() => setActiveTab('home')} color="text-[#FF7A00]" />
          <NavItem icon={<Compass size={24} />} label="ภารกิจ" active={activeTab === 'missions'} onClick={() => setActiveTab('missions')} color="text-[#34A0A4]" />
          <NavItem icon={<ClipboardList size={24} />} label="บันทึก" active={activeTab === 'log'} onClick={() => setActiveTab('log')} color="text-rose-500" />
          <NavItem icon={<Trophy size={24} />} label="จัดอันดับ" active={activeTab === 'leaderboard'} onClick={() => setActiveTab('leaderboard')} color="text-yellow-500" />
          <NavItem icon={<User size={24} />} label="โปรไฟล์" active={activeTab === 'profile'} onClick={() => setActiveTab('profile')} color="text-emerald-500" />
        </nav>
      )}
    </div>
  );
}

function NavItem({ icon, label, active, onClick, color }) {
  return (
    <button onClick={onClick} className={`flex flex-col items-center gap-1.5 transition-all duration-300 ${active ? color : 'text-gray-400 hover:text-gray-600'}`}>
      <div className={`transition-transform duration-300 ${active ? 'scale-110 drop-shadow-md' : 'scale-100'}`}>{React.cloneElement(icon, { strokeWidth: active ? 3 : 2 })}</div>
      <span className={`text-[10px] ${active ? 'font-black' : 'font-bold'}`}>{label}</span>
    </button>
  );
}

// --- Tab: Home ---
function HomeTab({ profile, db, appId, onLogMeal, showToast }) {
  const [stats, setStats] = useState({ water: 0, steps: 0 });
  const [stepInput, setStepInput] = useState('');
  const todayStr = getTodayStr();

  useEffect(() => {
    const statsRef = doc(db, 'artifacts', appId, 'public', 'data', `stats_${profile.uid}`, todayStr);
    const unsub = onSnapshot(statsRef, (docSnap) => {
      if (docSnap.exists()) setStats(docSnap.data());
      else setDoc(statsRef, { water: 0, steps: 0 }); 
    });
    return () => unsub();
  }, [profile.uid, db, appId, todayStr]);

  const updateWater = async () => {
    if (stats.water >= 8) return showToast('ยอดเยี่ยม! ดื่มน้ำครบ 8 แก้วแล้วสำหรับวันนี้ 💧', 'success');
    await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', `stats_${profile.uid}`, todayStr), { water: stats.water + 1 });
  };
  const updateSteps = async () => {
    const adding = parseInt(stepInput);
    if (!adding || adding <= 0) return;
    await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', `stats_${profile.uid}`, todayStr), { steps: stats.steps + adding });
    setStepInput('');
  };

  return (
    <div className="p-6 pt-12 animate-fade-in space-y-6">
      <div className="flex items-center gap-4 bg-white/90 backdrop-blur-md p-4 rounded-[2rem] shadow-sm border border-white hover:shadow-md transition">
        <div className="w-16 h-16 rounded-full border-[3px] border-white shadow-md overflow-hidden bg-gradient-to-tr from-gray-100 to-gray-50 flex items-center justify-center text-3xl flex-shrink-0">
          {profile.avatar?.startsWith('http') || profile.avatar?.startsWith('data:') ? <img src={profile.avatar} className="w-full h-full object-cover" alt="Profile" /> : profile.avatar}
        </div>
        <div className="min-w-0">
          <h1 className="text-2xl font-black text-gray-800 tracking-tight truncate">{profile.name}</h1>
          <p className="text-sm font-bold text-[#FF7A00]">พร้อมลุยเป้าหมายวันนี้แล้วยัง? 🔥</p>
        </div>
      </div>

      <div className="bg-gradient-to-br from-[#FF7A00] to-[#FF004D] rounded-[2rem] p-6 shadow-xl relative overflow-hidden text-white border border-white/20">
        <div className="absolute top-0 right-0 p-4 opacity-20 text-7xl">⭐</div>
        <div className="flex justify-between items-end mb-3 relative z-10">
          <h2 className="text-lg font-black">ระดับ: {profile.level || 1} <span className="text-white/80 font-bold text-sm">(สุขภาพดี)</span></h2>
        </div>
        <div className="w-full bg-black/20 rounded-full h-4 mb-4 overflow-hidden shadow-inner border border-white/10 relative z-10">
          <div className="bg-white h-full rounded-full shadow-[0_0_10px_rgba(255,255,255,0.5)]" style={{ width: '40%' }}></div>
        </div>
        <div className="flex items-center gap-3 relative z-10">
          <div className="w-12 h-12 bg-white/20 backdrop-blur-sm rounded-full flex items-center justify-center text-2xl shadow-inner border border-white/30">🪙</div>
          <p className="font-black text-4xl tracking-tight">{(profile.points || 0).toLocaleString()} <span className="text-base font-bold text-white/80">แต้ม</span></p>
        </div>
      </div>

      <div className="grid grid-cols-2 gap-4">
        <div className="bg-white rounded-[2rem] p-5 shadow-sm flex flex-col items-center border border-gray-50 relative pb-8 hover:shadow-md transition">
          <h4 className="text-gray-800 font-extrabold mb-3 text-sm flex items-center gap-1"><Droplet size={16} className="text-[#34A0A4]"/> ดื่มน้ำ (แก้ว)</h4>
          <div className="relative w-24 h-24 mb-2">
            <svg className="w-full h-full transform -rotate-90 drop-shadow-sm">
              <circle cx="48" cy="48" r="40" strokeWidth="8" fill="transparent" className="stroke-cyan-50" />
              <circle cx="48" cy="48" r="40" strokeWidth="8" fill="transparent" strokeDasharray="251.2" strokeDashoffset={251.2 - (Math.min(100,(stats.water/8)*100)/100)*251.2} className="stroke-[#34A0A4] transition-all duration-1000 ease-out" strokeLinecap="round" />
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
              <circle cx="48" cy="48" r="40" strokeWidth="8" fill="transparent" strokeDasharray="251.2" strokeDashoffset={251.2 - (Math.min(100,(stats.steps/10000)*100)/100)*251.2} className="stroke-rose-500 transition-all duration-1000 ease-out" strokeLinecap="round" />
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

      <button onClick={onLogMeal} className="w-full bg-gradient-to-r from-[#A67B5B] to-[#8B5A2B] text-white rounded-[2rem] py-5 mt-6 text-xl font-black shadow-[0_10px_25px_rgba(166,123,91,0.3)] flex items-center justify-center gap-3 hover:scale-[1.02] active:scale-95 transition-all">
        <span className="text-2xl drop-shadow-md">🥗</span> บันทึกมื้ออาหารวันนี้
      </button>
    </div>
  );
}

// --- Tab: Missions (With limits & expiry checks) ---
function UserMissionsTab({ missions, userProgress, onComplete }) {
  const fileInputRef = useRef(null);
  const [selectedMission, setSelectedMission] = useState(null);
  const [verifying, setVerifying] = useState(false);
  const [activeFilter, setActiveFilter] = useState('ทั้งหมด');
  const filters = ['ทั้งหมด', 'เบาหวาน', 'ความดัน', 'หัวใจ'];
  const todayStr = getTodayStr();

  // Filter valid missions (not expired)
  const validMissions = missions.filter(m => {
    if (m.expiresAt && m.expiresAt < todayStr) return false;
    if (activeFilter !== 'ทั้งหมด' && m.category !== activeFilter) return false;
    return true;
  });

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
      setVerifying(false);
    }
  };

  return (
    <div className="p-6 pt-12 animate-fade-in space-y-6">
      <h1 className="text-3xl font-black text-gray-800 tracking-tight">ภารกิจสุขภาพ 🎯</h1>
      <input type="file" accept="image/*" capture="environment" className="hidden" ref={fileInputRef} onChange={handleFileChange} />

      {verifying && (
        <div className="fixed inset-0 bg-white/95 z-50 flex flex-col items-center justify-center backdrop-blur-md">
          <div className="w-28 h-28 bg-gradient-to-tr from-[#34A0A4] to-emerald-400 rounded-[2.5rem] flex items-center justify-center animate-bounce mb-6 shadow-xl border-4 border-white"><Camera className="text-white" size={56}/></div>
          <h2 className="text-3xl font-black text-gray-800 text-center tracking-tight">AI กำลังตรวจภาพ...</h2>
          <p className="text-emerald-500 font-bold mt-2 text-lg">โปรดรอสักครู่เพื่อให้คะแนนคุณ</p>
        </div>
      )}

      <div className="flex gap-2 overflow-x-auto hide-scrollbar pb-2">
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

      <div className="space-y-4">
        {validMissions.map(mission => {
          const completedCount = userProgress.filter(p => p.missionId === mission.id && p.date === todayStr).length;
          const maxAllowed = mission.maxPerDay || 1;
          const isFullyCompleted = completedCount >= maxAllowed;

          let catConfig = { bg: 'bg-gray-50', icon: '📝', color: 'text-gray-500' };
          if (mission.category === 'เบาหวาน') catConfig = { bg: 'bg-red-50', icon: '🚫', color: 'text-red-500' };
          if (mission.category === 'ความดัน') catConfig = { bg: 'bg-blue-50', icon: '👟', color: 'text-blue-500' };
          if (mission.category === 'หัวใจ') catConfig = { bg: 'bg-emerald-50', icon: '🥗', color: 'text-emerald-500' };

          return (
            <div key={mission.id} className={`bg-white rounded-[2rem] p-5 shadow-sm border-2 transition-all duration-500 ${isFullyCompleted ? 'border-emerald-100 bg-emerald-50/40' : 'border-transparent hover:shadow-md'}`}>
              <div className="flex gap-4">
                <div className={`w-16 h-16 rounded-[1.5rem] flex items-center justify-center text-3xl flex-shrink-0 shadow-inner border border-white ${catConfig.bg}`}>{catConfig.icon}</div>
                <div className="flex-1 pt-1 pr-2">
                  <h3 className={`font-extrabold text-lg leading-tight ${isFullyCompleted ? 'text-gray-400 line-through' : 'text-gray-800'}`}>{mission.title}</h3>
                  <p className="text-sm text-gray-500 mt-1 line-clamp-2 font-bold">{mission.subtitle}</p>
                </div>
              </div>

              <div className="mt-5 flex justify-between items-center border-t border-gray-50 pt-4">
                <div className="flex items-center gap-1 bg-gradient-to-r from-orange-50 to-yellow-50 text-orange-600 px-3 py-1.5 rounded-xl font-black text-sm border border-orange-100 shadow-sm">
                  🪙 +{mission.points} Pts <span className="text-[10px] text-gray-400 bg-white px-2 py-0.5 rounded-md ml-1 border">ทำแล้ว {completedCount}/{maxAllowed}</span>
                </div>
                
                {isFullyCompleted ? (
                  <span className="font-extrabold text-emerald-500 flex items-center gap-1.5 bg-white px-4 py-2 rounded-xl text-sm shadow-sm border border-emerald-100">
                    <CheckCircle size={16} strokeWidth={3} /> สำเร็จแล้ว
                  </span>
                ) : (
                  <button onClick={() => handleUploadClick(mission)} className="bg-gradient-to-r from-[#FF7A00] to-orange-500 text-white px-5 py-2.5 rounded-xl font-black text-sm shadow-[0_5px_15px_rgba(255,122,0,0.3)] flex items-center gap-2 hover:opacity-90 active:scale-95 transition-all">
                    <Camera size={18} /> ยืนยันภาพ
                  </button>
                )}
              </div>
            </div>
          );
        })}
        {validMissions.length === 0 && <p className="text-center text-gray-400 font-bold p-10 bg-white rounded-3xl border border-dashed border-gray-200">ยังไม่มีภารกิจหมวดนี้</p>}
      </div>
    </div>
  );
}

// --- Tab: Food Log (AI) ---
function FoodLogTab({ profile, db, appId, showToast }) {
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
    const logsRef = collection(db, 'artifacts', appId, 'public', 'data', `logs_${profile.uid}`);
    const unsub = onSnapshot(logsRef, (snapshot) => {
      const allLogs = snapshot.docs.map(d => ({id: d.id, ...d.data()}));
      setLogs(allLogs.filter(l => l.date === todayStr).sort((a,b) => b.timestamp - a.timestamp));
    });
    return () => unsub();
  }, [profile.uid, db, appId, todayStr]);

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
      showToast("ระบบ AI ขัดข้อง กรุณากรอกข้อมูลเองครับ", "error");
    } finally {
      setIsAnalyzing(false);
    }
  };

  const handleSaveMeal = async () => {
    if (!foodName.trim() && !image) return showToast('กรุณาระบุชื่ออาหารหรือถ่ายรูป', 'error');
    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', `logs_${profile.uid}`), {
      meal: activeMeal, name: foodName || 'เมนูระบุไม่ได้', note: note,
      image: image, ai_data: aiAnalysis || null, date: todayStr, timestamp: Date.now()
    });
    setFoodName(''); setNote(''); setImage(null); setAiAnalysis(null);
    showToast('บันทึกมื้ออาหารสำเร็จ! 🥗');
  };

  const tabs = [{ id: 'morning', label: 'เช้า', icon: '☀️', color:'text-yellow-500' }, { id: 'lunch', label: 'กลางวัน', icon: '🍴', color:'text-orange-500' }, { id: 'dinner', label: 'เย็น', icon: '🌙', color:'text-indigo-500' }, { id: 'snack', label: 'ว่าง', icon: '🍎', color:'text-red-500' }];

  return (
    <div className="space-y-6 pb-6">
      <div className="pt-12 pb-4 px-6 border-b border-gray-100 sticky top-0 bg-white/95 backdrop-blur-md z-10 flex items-center gap-3 shadow-sm">
        <h1 className="text-xl font-black text-gray-800 leading-tight flex-1 text-center tracking-tight">บันทึกอาหารด้วย AI</h1>
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

      <div className="px-6 space-y-6">
        <div 
          onClick={() => !isAnalyzing && fileRef.current.click()}
          className={`w-full h-64 rounded-[2.5rem] border-2 flex flex-col items-center justify-center overflow-hidden relative transition-all cursor-pointer shadow-sm
            ${image ? 'border-transparent shadow-lg' : 'border-dashed border-rose-400 bg-white'}
            ${isAnalyzing ? 'opacity-90 pointer-events-none' : 'hover:scale-[1.02]'}`}
        >
          {isAnalyzing ? (
            <div className="flex flex-col items-center z-10 bg-white/90 p-6 rounded-3xl backdrop-blur-md shadow-xl border border-rose-100 text-center">
              <Sparkles className="text-rose-500 animate-spin mb-3" size={48} />
              <span className="font-black text-transparent bg-clip-text bg-gradient-to-r from-rose-500 to-orange-500 text-xl tracking-tight">AI กำลังวิเคราะห์เมนู...</span>
            </div>
          ) : image ? (
            <>
              <img src={image} alt="Food" className="w-full h-full object-cover" />
              <div className="absolute inset-0 bg-black/40 flex items-center justify-center opacity-0 hover:opacity-100 transition-opacity">
                 <div className="bg-white/95 px-6 py-3 rounded-full font-black text-gray-800 flex items-center gap-2 shadow-lg"><ImageIcon size={20}/> แตะเพื่อเปลี่ยนรูป</div>
              </div>
            </>
          ) : (
            <>
              <div className="w-24 h-24 bg-gradient-to-tr from-rose-100 to-orange-50 rounded-[2rem] flex items-center justify-center mb-4 shadow-inner transform rotate-3 border border-white"><Camera className="text-rose-500" size={40}/></div>
              <span className="font-black text-rose-500 text-xl tracking-tight">ถ่ายรูปให้ AI ช่วยวิเคราะห์</span>
              <span className="text-xs text-gray-500 mt-2 font-bold bg-gray-50 px-3 py-1 rounded-full">(แคลอรี่ หวาน เค็ม จะถูกประเมินอัตโนมัติ)</span>
            </>
          )}
          <input type="file" accept="image/*" className="hidden" ref={fileRef} onChange={handlePhoto} />
        </div>

        {aiAnalysis && (
          <div className="bg-gradient-to-br from-white to-rose-50 p-6 rounded-[2.5rem] border border-rose-100 shadow-lg animate-slide-up relative overflow-hidden">
            <div className="absolute top-0 right-0 p-4 opacity-10 text-8xl">🤖</div>
            <h3 className="font-black text-gray-800 flex items-center gap-2 mb-4 relative z-10"><Sparkles size={20} className="text-rose-500"/> ข้อมูลโภชนาการ (AI):</h3>
            <div className="grid grid-cols-3 gap-3 mb-4 relative z-10">
              <div className="bg-white p-3 rounded-2xl text-center shadow-sm border border-rose-50">
                 <p className="text-[10px] text-gray-400 font-black uppercase tracking-wider mb-1">แคลอรี่</p>
                 <p className="font-black text-rose-500 text-2xl">{aiAnalysis.calories}</p>
              </div>
              <div className="bg-white p-3 rounded-2xl text-center shadow-sm border border-rose-50">
                 <p className="text-[10px] text-gray-400 font-black uppercase tracking-wider mb-1">ความหวาน</p>
                 <p className={`font-black text-xl ${aiAnalysis.sugar === 'สูง' ? 'text-red-500' : 'text-emerald-500'}`}>{aiAnalysis.sugar}</p>
              </div>
              <div className="bg-white p-3 rounded-2xl text-center shadow-sm border border-rose-50">
                 <p className="text-[10px] text-gray-400 font-black uppercase tracking-wider mb-1">ความเค็ม</p>
                 <p className={`font-black text-xl ${aiAnalysis.sodium === 'สูง' ? 'text-red-500' : 'text-emerald-500'}`}>{aiAnalysis.sodium}</p>
              </div>
            </div>
            <div className="bg-white/90 p-4 rounded-2xl shadow-inner border border-white relative z-10 text-center">
              <p className="text-sm font-bold text-gray-700 leading-relaxed">"{aiAnalysis.advice}"</p>
            </div>
          </div>
        )}

        <div className="bg-white p-6 rounded-[2.5rem] shadow-sm space-y-4 border border-gray-50">
          <input type="text" placeholder="ระบุชื่อเมนูอาหารด้วยตนเอง" value={foodName} onChange={e=>setFoodName(e.target.value)} className="w-full bg-gray-50 border-2 border-transparent focus:border-rose-400 rounded-2xl p-4 text-gray-800 font-black outline-none transition-colors text-lg" />
          <div>
            <h3 className="font-extrabold text-gray-800 mb-2 flex items-center gap-2">📝 พฤติกรรมการปรุง</h3>
            <textarea rows="2" placeholder="เช่น เค็มจัด, ไม่ใส่น้ำตาล, หวานน้อย" value={note} onChange={e=>setNote(e.target.value)} className="w-full bg-gray-50 border-2 border-transparent focus:border-rose-400 rounded-2xl p-4 text-gray-800 font-bold outline-none resize-none transition-colors"></textarea>
          </div>
          <button onClick={handleSaveMeal} className="w-full bg-gradient-to-r from-rose-400 to-orange-500 text-white font-black py-5 rounded-2xl text-xl shadow-[0_10px_25px_rgba(244,63,94,0.3)] hover:scale-[1.02] active:scale-95 transition-all tracking-wide">บันทึกมื้อนี้เข้าสู่ระบบ</button>
        </div>

        {logs.length > 0 && (
          <div className="pt-4">
            <h3 className="font-black text-gray-800 text-xl mb-4 tracking-tight">ประวัติการกินวันนี้</h3>
            <div className="space-y-4">
              {logs.map(log => (
                <div key={log.id} className="bg-white border border-gray-50 rounded-[2rem] p-4 flex gap-4 shadow-sm items-center hover:shadow-md transition">
                  <div className="w-20 h-20 bg-gray-50 rounded-[1.5rem] overflow-hidden flex-shrink-0 border-2 border-white shadow-inner">
                    {log.image ? <img src={log.image} className="w-full h-full object-cover" /> : <div className="w-full h-full flex items-center justify-center text-3xl">🍽️</div>}
                  </div>
                  <div className="flex-1">
                    <span className="text-[10px] font-black text-rose-500 bg-rose-50 px-3 py-1 rounded-full uppercase inline-block mb-1 border border-rose-100 shadow-sm">{tabs.find(t=>t.id === log.meal)?.label}</span>
                    <h4 className="font-extrabold text-gray-800 text-lg leading-tight mt-1">{log.name}</h4>
                    {log.ai_data && <p className="text-xs font-black text-orange-500 mt-1 bg-orange-50 inline-block px-2 py-0.5 rounded-md border border-orange-100">🔥 {log.ai_data.calories} kcal</p>}
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

// --- Tab: Leaderboard ---
function UserLeaderboardTab({ leaderboard, currentUserId }) {
  const userRank = leaderboard.find(u => u.uid === currentUserId)?.rank || '-';
  const userPoints = leaderboard.find(u => u.uid === currentUserId)?.points || 0;
  const top5 = leaderboard.slice(0, 5);

  return (
    <div className="pt-12 pb-32 animate-fade-in bg-gradient-to-b from-[#F4F5F7] to-[#E6F3EC] min-h-screen relative overflow-hidden">
      <div className="absolute top-[-10%] left-[-10%] w-64 h-64 bg-emerald-200 rounded-full blur-3xl opacity-40"></div>
      <div className="absolute top-[20%] right-[-10%] w-64 h-64 bg-yellow-200 rounded-full blur-3xl opacity-40"></div>

      <div className="text-center px-6 mb-8 relative z-10">
        <h1 className="text-5xl font-black text-gray-900 tracking-tighter drop-shadow-sm mb-3">Leaderboard</h1>
        <div className="bg-white inline-flex items-center gap-2 px-5 py-2 rounded-full shadow-sm border border-gray-100">
          <span className="animate-pulse text-lg">🏆</span>
          <p className="text-gray-600 font-extrabold text-sm tracking-wide">อัปเดตทุกสัปดาห์ จากผู้ใช้งานจริง</p>
        </div>
      </div>

      <div className="px-6 space-y-4 relative z-10">
        {top5.map((user) => (
          <div key={user.uid} className={`bg-white/95 backdrop-blur-sm p-4 rounded-[2rem] shadow-sm flex items-center relative overflow-hidden transition-all hover:scale-[1.02] border-4 ${user.uid === currentUserId ? 'border-emerald-400 shadow-emerald-100' : 'border-white'}`}>
            {user.rank === 1 && <div className="absolute top-0 left-0 w-2 h-full bg-gradient-to-b from-yellow-300 to-yellow-500"></div>}
            {user.rank === 2 && <div className="absolute top-0 left-0 w-2 h-full bg-gradient-to-b from-gray-300 to-gray-400"></div>}
            {user.rank === 3 && <div className="absolute top-0 left-0 w-2 h-full bg-gradient-to-b from-orange-300 to-orange-500"></div>}
            
            <div className={`w-10 font-black text-3xl text-center mr-2 ${user.rank <= 3 ? 'text-gray-800' : 'text-gray-300'}`}>{user.rank}</div>
            <div className="w-16 h-16 bg-gradient-to-tr from-gray-100 to-white rounded-full flex items-center justify-center text-4xl mr-4 overflow-hidden border-[3px] border-white shadow-md flex-shrink-0">
              {user.avatar?.startsWith('http') || user.avatar?.startsWith('data:') ? <img src={user.avatar} className="w-full h-full object-cover"/> : user.avatar}
            </div>
            <div className="flex-1 min-w-0">
              <h4 className="font-extrabold text-gray-800 text-lg truncate flex items-center gap-2">
                {user.name} 
                {user.uid === currentUserId && <span className="text-[10px] text-white bg-emerald-500 px-2 py-0.5 rounded-full shadow-sm flex-shrink-0">คุณ</span>}
              </h4>
              <p className="font-black text-gray-500 text-base mt-1">{(user.points||0).toLocaleString()} <span className="text-[10px] font-black uppercase text-gray-400">Pts</span></p>
            </div>
            <div className="w-12 text-center drop-shadow-lg text-4xl">
              {user.rank === 1 && '🥇'} {user.rank === 2 && '🥈'} {user.rank === 3 && '🥉'}
            </div>
          </div>
        ))}
      </div>

      <div className="fixed bottom-[5.5rem] left-1/2 -translate-x-1/2 w-full max-w-md px-4 z-20 pb-4 pointer-events-none">
        <div className="bg-gradient-to-r from-[#1E3A8A] to-[#3B82F6] rounded-[2.5rem] p-5 flex items-center text-white shadow-[0_20px_40px_rgba(59,130,246,0.4)] pointer-events-auto border-[4px] border-white/20 backdrop-blur-md">
          <div className="w-16 h-16 bg-white rounded-full flex items-center justify-center text-4xl mr-4 overflow-hidden border-[3px] border-[#60A5FA] shadow-inner">
             {leaderboard.find(u=>u.uid===currentUserId)?.avatar?.startsWith('http') || leaderboard.find(u=>u.uid===currentUserId)?.avatar?.startsWith('data:') ? <img src={leaderboard.find(u=>u.uid===currentUserId)?.avatar} className="w-full h-full object-cover"/> : leaderboard.find(u=>u.uid===currentUserId)?.avatar || '👦🏻'}
          </div>
          <div className="flex-1 flex justify-between items-center">
            <div>
              <p className="text-[10px] text-blue-200 font-black uppercase tracking-widest mb-1">อันดับปัจจุบัน</p>
              <h3 className="text-3xl font-black leading-none drop-shadow-sm text-white"># {userRank}</h3>
            </div>
            <div className="text-right">
              <p className="text-[10px] text-blue-200 font-black uppercase tracking-widest mb-1">คะแนนรวม</p>
              <p className="font-black text-2xl text-yellow-300 drop-shadow-md">{(userPoints||0).toLocaleString()}</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

// --- Tab: Profile ---
function ProfileTab({ profile, onLogout, onOpenRewards, onUpdateProfile, showToast }) {
  const [isEditing, setIsEditing] = useState(false);
  const [editName, setEditName] = useState(profile.name);
  const [editAvatar, setEditAvatar] = useState(profile.avatar);
  const fileRef = useRef(null);

  // Mock AI Evaluation
  const evalMock = { sugar: 4, sodium: 5, fat: 3, advice: "สัปดาห์นี้ลดเค็มได้เยี่ยม! รักษาระดับนี้นะครับ 👏" };

  const handleAvatarChange = async (e) => {
    const file = e.target.files[0];
    if (file) {
      const base64 = await resizeImage(file, 200); 
      setEditAvatar(base64);
    }
  };

  const handleSaveProfile = () => {
    if(!editName.trim()) return showToast("กรุณาใส่ชื่อ", 'error');
    onUpdateProfile(editName, editAvatar);
    setIsEditing(false);
  };

  return (
    <div className="p-6 pt-12 animate-fade-in bg-[#F4F5F7] min-h-screen">
      
      {/* Header Profile */}
      <div className="bg-white rounded-[2.5rem] p-8 shadow-sm mb-6 relative overflow-hidden border border-gray-50 flex flex-col items-center">
        <div className="absolute top-0 right-0 p-4 opacity-5 text-8xl">👑</div>
        <div className="w-32 h-32 rounded-full border-[5px] border-white shadow-xl overflow-hidden bg-gray-100 flex items-center justify-center text-6xl mb-5 relative z-10">
          {profile.avatar?.startsWith('http') || profile.avatar?.startsWith('data:') ? <img src={profile.avatar} className="w-full h-full object-cover" /> : profile.avatar}
        </div>
        <h2 className="text-3xl font-black text-gray-800 relative z-10 text-center tracking-tight">{profile.name}</h2>
        <p className="text-emerald-500 font-black bg-emerald-50 border border-emerald-100 px-5 py-2 rounded-full mt-3 text-sm shadow-sm flex items-center gap-2">
          <Star size={16} className="fill-emerald-500"/> Level {profile.level || 1} Explorer
        </p>
        
        <button onClick={() => setIsEditing(true)} className="mt-6 bg-gray-900 text-white px-6 py-3.5 rounded-2xl font-black text-sm hover:bg-gray-800 transition-colors flex items-center gap-2 shadow-md active:scale-95">
          <Edit size={16}/> แก้ไขโปรไฟล์ส่วนตัว
        </button>
      </div>

      {/* Edit Profile Modal */}
      {isEditing && (
        <div className="fixed inset-0 bg-black/70 z-50 flex items-center justify-center p-6 backdrop-blur-sm animate-fade-in">
          <div className="bg-white rounded-[3rem] p-8 w-full max-w-sm shadow-2xl relative border-[4px] border-white/20">
            <button onClick={() => setIsEditing(false)} className="absolute top-6 right-6 text-gray-400 hover:text-gray-800 bg-gray-100 p-2 rounded-full transition"><X size={20}/></button>
            <h3 className="text-3xl font-black text-gray-800 mb-8 text-center tracking-tight">แก้ไขข้อมูล</h3>
            
            <div className="flex flex-col items-center mb-8">
              <div className="w-32 h-32 rounded-full border-4 border-gray-100 overflow-hidden mb-4 relative group cursor-pointer shadow-inner bg-gray-50" onClick={() => fileRef.current.click()}>
                 {editAvatar?.startsWith('http') || editAvatar?.startsWith('data:') ? <img src={editAvatar} className="w-full h-full object-cover" /> : <div className="w-full h-full flex items-center justify-center text-6xl">{editAvatar}</div>}
                 <div className="absolute inset-0 bg-black/50 flex flex-col items-center justify-center opacity-0 group-hover:opacity-100 transition"><Camera className="text-white mb-1" size={32}/><span className="text-[10px] text-white font-bold">เปลี่ยนรูป</span></div>
              </div>
              <input type="file" accept="image/*" className="hidden" ref={fileRef} onChange={handleAvatarChange} />
            </div>

            <div className="space-y-4">
              <div className="bg-gray-50 p-2 rounded-2xl border-2 border-transparent focus-within:border-emerald-400 transition-colors">
                <p className="text-[10px] font-black text-gray-400 uppercase px-3 pt-1">ชื่อแสดงผล</p>
                <input type="text" value={editName} onChange={e=>setEditName(e.target.value)} className="w-full bg-transparent p-2 text-gray-800 font-black outline-none text-xl text-center" />
              </div>
              <button onClick={handleSaveProfile} className="w-full bg-gradient-to-r from-emerald-400 to-[#34A0A4] text-white font-black py-4 rounded-2xl shadow-[0_10px_20px_rgba(52,160,164,0.3)] hover:scale-[1.02] active:scale-95 transition-all text-lg mt-4">บันทึกข้อมูล 💾</button>
            </div>
          </div>
        </div>
      )}

      {/* Weekly Evaluation */}
      <div className="text-center mb-6 mt-8">
        <h1 className="text-2xl font-black text-gray-900">สรุปผลรายสัปดาห์ (AI)</h1>
      </div>

      <div className="space-y-6">
        <div className="bg-white rounded-[2.5rem] p-6 shadow-sm border border-gray-50 relative overflow-hidden">
          <div className="absolute top-0 right-0 p-4 opacity-5 text-6xl">📊</div>
          
          <h3 className="font-extrabold text-gray-800 text-lg mb-6 relative z-10 flex items-center gap-2">การประเมินโภชนาการ</h3>
          <div className="flex justify-around relative z-10 mb-6 bg-gray-50 p-4 rounded-[1.5rem] border border-gray-100">
            <EvaluationItem icon="🍬" label="ลดหวาน" score={evalMock.sugar} />
            <div className="w-px bg-gray-200"></div>
            <EvaluationItem icon="🧂" label="ลดเค็ม" score={evalMock.sodium} />
            <div className="w-px bg-gray-200"></div>
            <EvaluationItem icon="💧" label="ลดมัน" score={evalMock.fat} />
          </div>

          <h3 className="font-extrabold text-gray-800 text-lg mb-4 relative z-10 flex items-center gap-2">AI แนะนำ <Sparkles size={16} className="text-orange-500"/></h3>
          <div className="bg-gradient-to-br from-blue-50 to-indigo-50 p-5 rounded-2xl rounded-tl-none relative ml-4 border border-blue-100 shadow-inner">
            <div className="absolute -left-3 top-0 w-4 h-4 bg-blue-50 transform rotate-45 border-l border-t border-blue-100"></div>
            <p className="text-gray-700 font-bold leading-relaxed">{evalMock.advice}</p>
          </div>
        </div>

        {/* Badges Section */}
        <div className="bg-white rounded-[2.5rem] p-6 shadow-sm border border-gray-50">
          <h3 className="font-extrabold text-gray-800 text-lg mb-5">รางวัลความสำเร็จ (Badges)</h3>
          <div className="flex justify-around text-center">
            <div className="flex flex-col items-center group">
              <div className="w-20 h-20 bg-gradient-to-tr from-blue-50 to-blue-100 rounded-[1.5rem] flex items-center justify-center text-4xl mb-3 shadow-inner group-hover:scale-110 transition-transform border border-white">🛡️</div>
              <span className="text-[11px] font-black text-gray-600">นักลดเค็มมือโปร</span>
            </div>
            <div className="flex flex-col items-center group">
              <div className="w-20 h-20 bg-gradient-to-tr from-emerald-50 to-green-100 rounded-[1.5rem] flex items-center justify-center text-4xl mb-3 shadow-inner group-hover:scale-110 transition-transform border border-white">🏅</div>
              <span className="text-[11px] font-black text-gray-600">พิชิตภารกิจ 7 วัน</span>
            </div>
            <div className="flex flex-col items-center group relative opacity-40 grayscale">
              <div className="w-20 h-20 bg-gray-100 rounded-[1.5rem] flex items-center justify-center text-4xl mb-3 shadow-inner border border-white">🏃</div>
              <Lock size={16} className="absolute top-2 right-2 text-gray-400" />
              <span className="text-[11px] font-black text-gray-400">เดินทะลุแสนก้าว</span>
            </div>
          </div>
        </div>

        {/* Action Buttons */}
        <button onClick={onOpenRewards} className="w-full bg-gradient-to-r from-[#FF7A00] to-[#FF004D] text-white rounded-[2rem] py-5 text-xl font-black shadow-[0_10px_25px_rgba(255,122,0,0.3)] flex items-center justify-center gap-3 mt-4 hover:scale-[1.02] active:scale-95 transition-all">
          <span className="text-2xl drop-shadow-md">🎁</span> ร้านค้าแลกของรางวัล
        </button>

        <button onClick={onLogout} className="w-full bg-white text-red-500 font-black border-[3px] border-red-50 rounded-[2rem] hover:bg-red-50 transition-colors py-4 flex justify-center items-center gap-2 shadow-sm mt-4">
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
        {[1, 2, 3, 4, 5].map(star => <Star key={star} size={14} className={star <= score ? 'fill-yellow-400 text-yellow-400 drop-shadow-sm' : 'fill-gray-200 text-gray-200'} />)}
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
  return <div className="flex h-screen w-full items-center justify-center bg-gray-900"><Activity className="text-[#FF004D] animate-pulse drop-shadow-[0_0_15px_rgba(255,0,77,0.8)]" size={80} strokeWidth={2} /></div>;
}


```
