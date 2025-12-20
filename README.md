import React, { useState, useEffect, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, getDoc, updateDoc, onSnapshot } from 'firebase/firestore';
import { 
  Heart, Activity, Apple, ShoppingCart, TrendingUp, 
  CheckCircle2, Coffee, Sun, Bell, ArrowLeft, 
  Flame, Award, Info, ChevronRight, Utensils, 
  Droplets, Brain
} from 'lucide-react';

// --- CONEXIÓN AUTOMÁTICA (No necesitas tocar nada aquí) ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'vitalia-v2';

// --- BASE DE DATOS DE SALUD (DÍAS REALES) ---
const PROGRAM = {
  1: {
    title: "Día 1: Despertar Metabólico",
    slogan: "Limpiando tu sangre desde hoy",
    ritual: "Agua tibia con medio limón exprimido",
    ritualWhy: "Estimula la bilis para movilizar grasas hepáticas.",
    breakfast: { name: "Avena integral con Canela", benefit: "Contiene Beta-glucanos que atrapan el LDL en el intestino." },
    lunch: { name: "Ensalada de Lentejas y Pimientos", benefit: "Fibra y Vitamina C para mejorar la elasticidad arterial." },
    dinner: { name: "Sopa depurativa de Apio y Calabacín", benefit: "Facilita el descanso del hígado durante la noche." },
    exercise: { name: "Caminata Moderada", dur: "25 min", goal: "Activar el metabolismo de lípidos." },
    challenge: "Cero Azúcar Hoy",
    challengeWhy: "El exceso de azúcar se convierte en triglicéridos rápidamente."
  },
  2: {
    title: "Día 2: Protección Endotelial",
    slogan: "Fortaleciendo tus arterias",
    ritual: "Té verde (sin endulzar)",
    ritualWhy: "Las catequinas protegen las paredes internas de tus venas.",
    breakfast: { name: "Yogur griego con semillas de Chía", benefit: "Omega-3 vegetal para reducir la inflamación." },
    lunch: { name: "Salmón o Trucha a la plancha", benefit: "Ácidos grasos que barren el exceso de colesterol." },
    dinner: { name: "Tortilla de claras con Espinacas", benefit: "Proteína magra para reparación celular." },
    exercise: { name: "Subir Escaleras", dur: "10 min continuos", goal: "Aumentar capacidad cardiovascular." },
    challenge: "8 Vasos de Agua",
    challengeWhy: "La hidratación es clave para filtrar impurezas en la sangre."
  }
};

export default function VitaliaApp() {
  const [user, setUser] = useState(null);
  const [profile, setProfile] = useState(null);
  const [dailyLog, setDailyLog] = useState(null);
  const [view, setView] = useState('loading');
  const [currentNotif, setCurrentNotif] = useState(null);

  // 1. INICIO DE SESIÓN AUTOMÁTICO
  useEffect(() => {
    const login = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    login();
    const unsub = onAuthStateChanged(auth, setUser);
    return () => unsub();
  }, []);

  // 2. SINCRONIZACIÓN CON LA NUBE (Persistencia)
  useEffect(() => {
    if (!user) return;

    const profileRef = doc(db, 'artifacts', appId, 'users', user.uid, 'profile');
    const todayId = new Date().toISOString().split('T')[0];
    const logRef = doc(db, 'artifacts', appId, 'users', user.uid, 'dailyLogs', todayId);

    // Escuchar perfil
    const unsubProfile = onSnapshot(profileRef, (snap) => {
      if (snap.exists()) {
        setProfile(snap.data());
      } else {
        setView('onboarding');
      }
    });

    // Escuchar progreso del día
    const unsubLog = onSnapshot(logRef, (snap) => {
      if (snap.exists()) {
        setDailyLog(snap.data());
        if (view === 'loading') setView('dashboard');
      } else {
        const initialLog = { ritual: false, breakfast: false, lunch: false, dinner: false, exercise: false, challenge: false };
        setDoc(logRef, initialLog);
        setDailyLog(initialLog);
        if (view === 'loading') setView('dashboard');
      }
    });

    // Notificaciones según la hora
    const updateNotif = () => {
      const h = new Date().getHours();
      if (h < 10) setCurrentNotif("🌅 Hora del Ritual y Desayuno. ¡Empecemos con energía!");
      else if (h < 15) setCurrentNotif("🥗 ¿Ya almorzaste? Tu corazón cuenta contigo.");
      else if (h < 19) setCurrentNotif("🏃 Es el momento ideal para tu ejercicio diario.");
      else setCurrentNotif("🌙 Cena ligero para que tu hígado descanse.");
    };
    updateNotif();
    const interval = setInterval(updateNotif, 60000);

    return () => { unsubProfile(); unsubLog(); clearInterval(interval); };
  }, [user]);

  // CÁLCULO DEL DÍA ACTUAL
  const dayNumber = useMemo(() => {
    if (!profile?.startDate) return 1;
    const start = new Date(profile.startDate);
    const now = new Date();
    const diff = Math.floor((now - start) / (1000 * 60 * 60 * 24)) + 1;
    return diff > 0 ? diff : 1;
  }, [profile]);

  const todayData = PROGRAM[dayNumber] || PROGRAM[1];

  const toggleTask = async (task) => {
    const todayId = new Date().toISOString().split('T')[0];
    const logRef = doc(db, 'artifacts', appId, 'users', user.uid, 'dailyLogs', todayId);
    const newValue = !dailyLog[task];
    setDailyLog({ ...dailyLog, [task]: newValue });
    await updateDoc(logRef, { [task]: newValue });
  };

  const handleRegister = async (e) => {
    e.preventDefault();
    const name = e.target.name.value;
    const initialProfile = {
      name,
      startDate: new Date().toISOString(),
      streak: 1,
      totalPoints: 0
    };
    await setDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'profile'), initialProfile);
    setView('dashboard');
  };

  if (view === 'loading') return (
    <div className="h-screen flex items-center justify-center bg-emerald-50 text-emerald-600">
      <Activity className="animate-spin" size={48} />
    </div>
  );

  if (view === 'onboarding') return (
    <div className="max-w-md mx-auto min-h-screen bg-slate-50 p-8 flex flex-col justify-center">
      <div className="text-center mb-10">
        <div className="w-24 h-24 bg-emerald-100 rounded-full flex items-center justify-center mx-auto mb-6 shadow-inner">
          <Heart className="text-emerald-600 fill-emerald-600" size={48} />
        </div>
        <h1 className="text-4xl font-black text-slate-800 tracking-tight">VITALIA</h1>
        <p className="text-slate-500 mt-3 font-medium text-lg italic">"Tu corazón merece una oportunidad natural"</p>
      </div>
      <form onSubmit={handleRegister} className="space-y-6">
        <div className="bg-white p-6 rounded-3xl shadow-xl border border-emerald-100">
          <label className="block text-sm font-bold text-slate-700 mb-2">¿Cuál es tu nombre?</label>
          <input name="name" required className="w-full p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl focus:border-emerald-500 outline-none transition-all" placeholder="Juan Pérez" />
        </div>
        <button type="submit" className="w-full py-5 bg-emerald-600 text-white rounded-3xl font-black text-lg shadow-2xl hover:bg-emerald-700 active:scale-95 transition-all">
          EMPEZAR DÍA 1
        </button>
      </form>
    </div>
  );

  const progressPercent = Math.round((Object.values(dailyLog || {}).filter(v => v === true).length / 6) * 100);

  return (
    <div className="max-w-md mx-auto min-h-screen bg-slate-50 pb-28 relative font-sans">
      {/* Notificación Superior */}
      {currentNotif && (
        <div className="bg-amber-100 border-b border-amber-200 p-3 flex items-center gap-3 animate-in slide-in-from-top-full duration-500">
          <Bell className="text-amber-600 shrink-0" size={18} />
          <p className="text-xs font-bold text-amber-800">{currentNotif}</p>
        </div>
      )}

      {/* Header Premium */}
      <header className="bg-gradient-to-br from-emerald-600 to-emerald-800 p-6 text-white rounded-b-[3rem] shadow-2xl">
        <div className="flex justify-between items-center mb-8">
          <div>
            <p className="text-emerald-200 text-xs font-black uppercase tracking-widest">Día {dayNumber}</p>
            <h1 className="text-2xl font-black">{todayData.title}</h1>
          </div>
          <div className="flex flex-col items-end">
            <div className="flex items-center gap-1 bg-white/20 px-3 py-1.5 rounded-full backdrop-blur-md border border-white/20">
              <Flame size={16} className="text-orange-400 fill-orange-400" />
              <span className="font-black text-sm">{profile?.streak || 1} d</span>
            </div>
          </div>
        </div>

        <div className="bg-white/10 backdrop-blur-lg rounded-3xl p-5 border border-white/20">
          <div className="flex justify-between items-end mb-3">
            <span className="text-sm font-bold">Progreso Diario</span>
            <span className="text-2xl font-black">{progressPercent}%</span>
          </div>
          <div className="w-full bg-black/20 h-3 rounded-full overflow-hidden p-0.5">
            <div 
              className="bg-white h-full rounded-full transition-all duration-1000 shadow-[0_0_10px_white]" 
              style={{ width: `${progressPercent}%` }}
            ></div>
          </div>
          <p className="text-[11px] mt-3 font-medium italic opacity-90">"{todayData.slogan}"</p>
        </div>
      </header>

      {/* Contenido Principal */}
      <main className="px-5 mt-8 space-y-4">
        
        {/* Desafío Central */}
        <div 
          onClick={() => toggleTask('challenge')}
          className={`p-5 rounded-3xl border-2 transition-all cursor-pointer ${dailyLog?.challenge ? 'bg-emerald-600 border-emerald-600 text-white' : 'bg-orange-50 border-orange-200 text-orange-950'}`}
        >
          <div className="flex justify-between items-center">
            <div className="flex gap-4">
              <div className={`p-3 rounded-2xl ${dailyLog?.challenge ? 'bg-white/20' : 'bg-orange-100'}`}>
                <Award className={dailyLog?.challenge ? 'text-white' : 'text-orange-600'} />
              </div>
              <div>
                <p className="text-[10px] font-black uppercase opacity-70 tracking-widest">Reto del Día</p>
                <h3 className="font-black text-lg">{todayData.challenge}</h3>
                <p className="text-xs mt-1 opacity-80">{todayData.challengeWhy}</p>
              </div>
            </div>
            {dailyLog?.challenge && <CheckCircle2 size={28} className="text-white fill-emerald-500" />}
          </div>
        </div>

        <h2 className="text-xl font-black text-slate-800 mb-2 px-1 flex items-center gap-2">
          <Activity className="text-emerald-600" size={24} /> ACCIONES DE HOY
        </h2>

        {/* Tarjetas de Acción */}
        <div className="space-y-3">
          <ActionItem 
            icon={<Sun className="text-amber-500" />} 
            title="Ritual Matutino" 
            label={todayData.ritual}
            info={todayData.ritualWhy}
            done={dailyLog?.ritual}
            onCheck={() => toggleTask('ritual')}
          />
          <ActionItem 
            icon={<Coffee className="text-blue-500" />} 
            title="Desayuno Saludable" 
            label={todayData.breakfast.name}
            info={todayData.breakfast.benefit}
            done={dailyLog?.breakfast}
            onCheck={() => toggleTask('breakfast')}
          />
          <ActionItem 
            icon={<Utensils className="text-emerald-500" />} 
            title="Almuerzo del Corazón" 
            label={todayData.lunch.name}
            info={todayData.lunch.benefit}
            done={dailyLog?.lunch}
            onCheck={() => toggleTask('lunch')}
          />
          <ActionItem 
            icon={<Activity className="text-indigo-500" />} 
            title="Tu Ejercicio Diario" 
            label={todayData.exercise.name}
            info={`${todayData.exercise.dur} • ${todayData.exercise.goal}`}
            done={dailyLog?.exercise}
            onCheck={() => toggleTask('exercise')}
            special
          />
          <ActionItem 
            icon={<Utensils className="text-purple-500" />} 
            title="Cena Ligera" 
            label={todayData.dinner.name}
            info={todayData.dinner.benefit}
            done={dailyLog?.dinner}
            onCheck={() => toggleTask('dinner')}
          />
        </div>

        {/* Educación Rápida */}
        <div className="bg-white border-2 border-slate-100 rounded-[2rem] p-6 shadow-sm mt-6">
          <div className="flex items-center gap-3 text-emerald-600 mb-3">
            <Brain size={24} />
            <span className="font-black text-sm uppercase tracking-tighter">Sabiduría Vital</span>
          </div>
          <h4 className="font-black text-slate-800 text-lg mb-2">¿Por qué el Día 1?</h4>
          <p className="text-sm text-slate-600 leading-relaxed font-medium">
            Hoy estamos reseteando tu metabolismo. Al quitar el azúcar y añadir fibra soluble (avena), obligamos a tu cuerpo a usar el colesterol almacenado para producir energía. **¡Tus arterias están empezando a respirar!**
          </p>
        </div>
      </main>

      {/* Navegación Inferior */}
      <nav className="fixed bottom-6 left-6 right-6 bg-white/80 backdrop-blur-xl border border-white/50 h-20 rounded-[2.5rem] flex justify-around items-center px-4 shadow-2xl z-50">
        <NavIcon active icon={<Heart size={26} />} label="Hoy" />
        <NavIcon icon={<Apple size={26} />} label="Dieta" onClick={() => alert("Menú detallado próximamente")} />
        <NavIcon icon={<ShoppingCart size={26} />} label="Tienda" onClick={() => alert("Lista de compras próximamente")} />
        <NavIcon icon={<TrendingUp size={26} />} label="Metas" onClick={() => alert("Gráficos de salud próximamente")} />
      </nav>
    </div>
  );
}

// --- SUBCOMPONENTES ---

function ActionItem({ icon, title, label, info, done, onCheck, special = false }) {
  return (
    <div 
      onClick={onCheck}
      className={`p-5 rounded-3xl border-2 flex items-center gap-4 cursor-pointer transition-all duration-300 ${done ? 'bg-slate-50 border-slate-100 opacity-60' : 'bg-white border-white shadow-md hover:border-emerald-200'} ${special && !done ? 'border-indigo-100 bg-indigo-50/30 ring-2 ring-indigo-50 ring-offset-2' : ''}`}
    >
      <div className={`p-4 rounded-2xl transition-all ${done ? 'bg-slate-200' : 'bg-slate-50 border shadow-inner'}`}>
        {done ? <CheckCircle2 className="text-emerald-600" /> : icon}
      </div>
      <div className="flex-1">
        <h3 className={`text-[10px] font-black uppercase tracking-widest ${done ? 'text-slate-400' : 'text-emerald-600'}`}>{title}</h3>
        <p className={`font-bold text-lg leading-tight ${done ? 'text-slate-500 line-through' : 'text-slate-800'}`}>{label}</p>
        <p className="text-[10px] text-slate-400 mt-1 font-medium italic">{info}</p>
      </div>
      {!done && <ChevronRight size={20} className="text-slate-300" />}
    </div>
  );
}

function NavIcon({ icon, label, active = false, onClick }) {
  return (
    <button onClick={onClick} className={`flex flex-col items-center gap-1 p-2 rounded-2xl transition-all ${active ? 'text-emerald-600 scale-110' : 'text-slate-400 hover:text-slate-600'}`}>
      <div className={`${active ? 'bg-emerald-50 p-2 rounded-xl shadow-inner' : ''}`}>
        {icon}
      </div>
      <span className="text-[10px] font-black tracking-tighter">{label}</span>
    </button>
  );
}

