import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot, query, serverTimestamp } from 'firebase/firestore';
import { Timer, Users, Trophy, LayoutDashboard, Send, CheckCircle, Lock, ShieldCheck } from 'lucide-react';

// Firebase Configuration
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'civil-tech-2024';

const CONFERENCE_DATE = new Date('2024-12-30T09:00:00').getTime();
const ADMIN_PASSCODE = "1234"; // يمكنك تغيير رمز الدخول من هنا

export default function App() {
  const [user, setUser] = useState(null);
  const [view, setView] = useState('home'); 
  const [isAdminAuthenticated, setIsAdminAuthenticated] = useState(false);
  const [passcodeInput, setPasscodeInput] = useState('');
  const [timeLeft, setTimeLeft] = useState({});
  const [formData, setFormData] = useState({ name: '', email: '', competition: 'الابتكار الهندسي', phone: '' });
  const [submissions, setSubmissions] = useState([]);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [successMsg, setSuccessMsg] = useState(false);
  const [adminError, setAdminError] = useState('');

  // Auth logic
  useEffect(() => {
    const initAuth = async () => {
      try {
        await signInAnonymously(auth);
      } catch (error) {
        console.error("Auth error:", error);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  // Fetch Submissions for Admin (Only if authenticated)
  useEffect(() => {
    if (!user || !isAdminAuthenticated || view !== 'admin') return;

    const q = collection(db, 'artifacts', appId, 'public', 'data', 'registrations');
    const unsubscribe = onSnapshot(q, (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setSubmissions(data.sort((a, b) => (b.timestamp?.seconds || 0) - (a.timestamp?.seconds || 0)));
    }, (error) => {
      console.error("Firestore error:", error);
    });

    return () => unsubscribe();
  }, [user, view, isAdminAuthenticated]);

  // Countdown Logic
  useEffect(() => {
    const timer = setInterval(() => {
      const now = new Date().getTime();
      const distance = CONFERENCE_DATE - now;

      setTimeLeft({
        days: Math.floor(distance / (1000 * 60 * 60 * 24)),
        hours: Math.floor((distance % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60)),
        minutes: Math.floor((distance % (1000 * 60 * 60)) / (1000 * 60)),
        seconds: Math.floor((distance % (1000 * 60)) / 1000)
      });
    }, 1000);
    return () => clearInterval(timer);
  }, []);

  const handleRegister = async (e) => {
    e.preventDefault();
    if (!user) return;
    setIsSubmitting(true);
    try {
      await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'registrations'), {
        ...formData,
        userId: user.uid,
        timestamp: serverTimestamp()
      });
      setSuccessMsg(true);
      setFormData({ name: '', email: '', competition: 'الابتكار الهندسي', phone: '' });
      setTimeout(() => setSuccessMsg(false), 5000);
    } catch (err) {
      console.error(err);
    } finally {
      setIsSubmitting(false);
    }
  };

  const handleAdminLogin = (e) => {
    e.preventDefault();
    if (passcodeInput === ADMIN_PASSCODE) {
      setIsAdminAuthenticated(true);
      setAdminError('');
    } else {
      setAdminError('رمز الدخول غير صحيح!');
    }
  };

  const Nav = () => (
    <nav className="bg-slate-900 text-white p-4 sticky top-0 z-50 shadow-xl">
      <div className="max-w-6xl mx-auto flex justify-between items-center">
        <h1 className="text-2xl font-bold tracking-tighter flex items-center gap-2 cursor-pointer" onClick={() => setView('home')}>
          <div className="bg-blue-600 p-1 rounded">CT</div> Civil Tech
        </h1>
        <div className="flex gap-6 items-center">
          <button onClick={() => setView('home')} className={`hover:text-blue-400 transition ${view === 'home' ? 'text-blue-400' : ''}`}>الرئيسية</button>
          <button onClick={() => setView('register')} className="bg-blue-600 px-4 py-2 rounded-lg hover:bg-blue-700 transition">سجل الآن</button>
          <button onClick={() => setView('admin')} className={`flex items-center gap-1 text-sm ${view === 'admin' ? 'text-blue-400' : 'text-slate-400'}`}>
            <LayoutDashboard size={16} /> لوحة التحكم
          </button>
        </div>
      </div>
    </nav>
  );

  return (
    <div className="min-h-screen bg-slate-50 font-sans text-right" dir="rtl">
      <Nav />

      {view === 'home' && (
        <main>
          <section className="bg-gradient-to-b from-slate-900 to-slate-800 text-white py-24 px-4 text-center">
            <h2 className="text-5xl md:text-7xl font-black mb-6">مؤتمر Civil Tech 2024</h2>
            <p className="text-xl text-slate-300 mb-12 max-w-2xl mx-auto leading-relaxed">
              المستقبل يبدأ هنا. انضم إلى نخبة المهندسين والمبتكرين في أضخم حدث تكنولوجي في الهندسة المدنية.
            </p>
            
            <div className="grid grid-cols-2 md:grid-cols-4 gap-4 max-w-3xl mx-auto mb-16">
              {[
                { label: 'يوم', value: timeLeft.days },
                { label: 'ساعة', value: timeLeft.hours },
                { label: 'دقيقة', value: timeLeft.minutes },
                { label: 'ثانية', value: timeLeft.seconds }
              ].map((item, i) => (
                <div key={i} className="bg-white/10 backdrop-blur-md p-6 rounded-2xl border border-white/20 shadow-2xl">
                  <div className="text-4xl md:text-6xl font-mono font-bold text-blue-400">
                    {item.value || '0'}
                  </div>
                  <div className="text-sm uppercase tracking-widest text-slate-400 mt-2">{item.label}</div>
                </div>
              ))}
            </div>

            <button 
              onClick={() => setView('register')}
              className="bg-blue-600 text-white px-10 py-4 rounded-full text-xl font-bold hover:bg-blue-500 transform hover:scale-105 transition shadow-lg shadow-blue-500/20"
            >
              سجل في المسابقات
            </button>
          </section>

          <section className="max-w-6xl mx-auto py-20 px-4 grid md:grid-cols-3 gap-8">
            <div className="bg-white p-8 rounded-2xl shadow-sm border border-slate-100 text-center">
              <div className="bg-blue-100 w-16 h-16 rounded-full flex items-center justify-center mx-auto mb-6">
                <Trophy className="text-blue-600" size={32} />
              </div>
              <h3 className="text-xl font-bold mb-3">مسابقات تقنية</h3>
              <p className="text-slate-600">نافس في تحديات الهندسة المدنية واربح جوائز قيمة.</p>
            </div>
            <div className="bg-white p-8 rounded-2xl shadow-sm border border-slate-100 text-center">
              <div className="bg-green-100 w-16 h-16 rounded-full flex items-center justify-center mx-auto mb-6">
                <Users className="text-green-600" size={32} />
              </div>
              <h3 className="text-xl font-bold mb-3">شبكة علاقات</h3>
              <p className="text-slate-600">تواصل مع كبار الخبراء والشركات في هذا القطاع.</p>
            </div>
            <div className="bg-white p-8 rounded-2xl shadow-sm border border-slate-100 text-center">
              <div className="bg-purple-100 w-16 h-16 rounded-full flex items-center justify-center mx-auto mb-6">
                <Timer className="text-purple-600" size={32} />
              </div>
              <h3 className="text-xl font-bold mb-3">ورش عمل</h3>
              <p className="text-slate-600">تعلم أحدث التقنيات المستخدمة في التصميم والإنشاء.</p>
            </div>
          </section>
        </main>
      )}

      {view === 'register' && (
        <section className="max-w-xl mx-auto py-16 px-4">
          <div className="bg-white p-8 rounded-3xl shadow-xl border border-slate-200">
            <div className="text-center mb-10">
              <h2 className="text-3xl font-bold text-slate-900 mb-2">تسجيل مسابقة</h2>
              <p className="text-slate-500">يرجى تعبئة البيانات بدقة للمشاركة</p>
            </div>

            {successMsg ? (
              <div className="bg-green-50 border border-green-200 text-green-700 p-6 rounded-2xl text-center flex flex-col items-center gap-4">
                <CheckCircle size={48} />
                <h4 className="font-bold text-lg">تم التسجيل بنجاح!</h4>
                <button onClick={() => setView('home')} className="mt-4 text-green-800 underline">العودة للرئيسية</button>
              </div>
            ) : (
              <form onSubmit={handleRegister} className="space-y-6">
                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">الاسم الكامل</label>
                  <input required type="text" className="w-full px-4 py-3 rounded-xl border border-slate-300" value={formData.name} onChange={e => setFormData({...formData, name: e.target.value})} />
                </div>
                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">البريد الإلكتروني</label>
                  <input required type="email" className="w-full px-4 py-3 rounded-xl border border-slate-300" value={formData.email} onChange={e => setFormData({...formData, email: e.target.value})} />
                </div>
                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">رقم الهاتف</label>
                  <input required type="tel" className="w-full px-4 py-3 rounded-xl border border-slate-300" value={formData.phone} onChange={e => setFormData({...formData, phone: e.target.value})} />
                </div>
                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">اختر المسابقة</label>
                  <select className="w-full px-4 py-3 rounded-xl border border-slate-300" value={formData.competition} onChange={e => setFormData({...formData, competition: e.target.value})}>
                    <option>الابتكار الهندسي</option>
                    <option>أفضل تصميم إنشائي</option>
                    <option>تكنولوجيا الاستدامة</option>
                  </select>
                </div>
                <button disabled={isSubmitting} className="w-full bg-blue-600 text-white py-4 rounded-xl font-bold hover:bg-blue-700 transition">
                  {isSubmitting ? 'جاري الإرسال...' : 'إرسال الطلب'}
                </button>
              </form>
            )}
          </div>
        </section>
      )}

      {view === 'admin' && (
        <section className="max-w-6xl mx-auto py-12 px-4">
          {!isAdminAuthenticated ? (
            <div className="max-w-md mx-auto bg-white p-8 rounded-3xl shadow-xl border border-slate-200">
              <div className="text-center mb-8">
                <div className="bg-slate-100 w-16 h-16 rounded-full flex items-center justify-center mx-auto mb-4">
                  <Lock className="text-slate-600" size={24} />
                </div>
                <h2 className="text-2xl font-bold text-slate-900">دخول المسؤول</h2>
                <p className="text-slate-500 text-sm">هذه المنطقة محمية، يرجى إدخال الرمز السري</p>
              </div>
              <form onSubmit={handleAdminLogin} className="space-y-4">
                <input 
                  type="password" 
                  placeholder="أدخل الرمز السري" 
                  className="w-full px-4 py-3 rounded-xl border border-slate-300 text-center text-2xl tracking-widest outline-none focus:ring-2 focus:ring-blue-500"
                  value={passcodeInput}
                  onChange={e => setPasscodeInput(e.target.value)}
                />
                {adminError && <p className="text-red-500 text-sm text-center font-bold">{adminError}</p>}
                <button className="w-full bg-slate-900 text-white py-3 rounded-xl font-bold hover:bg-slate-800 transition">دخول</button>
              </form>
            </div>
          ) : (
            <>
              <div className="flex justify-between items-center mb-8">
                <div className="flex items-center gap-3">
                  <div className="bg-green-100 p-2 rounded-lg text-green-600">
                    <ShieldCheck size={24} />
                  </div>
                  <h2 className="text-3xl font-bold text-slate-900">لوحة تحكم المسؤول</h2>
                </div>
                <div className="flex gap-4">
                  <div className="bg-blue-100 text-blue-700 px-4 py-2 rounded-lg font-bold">إجمالي المسجلين: {submissions.length}</div>
                  <button onClick={() => setIsAdminAuthenticated(false)} className="text-slate-400 hover:text-red-500 text-sm">خروج من الوضع الآمن</button>
                </div>
              </div>

              <div className="bg-white rounded-3xl shadow-xl overflow-hidden border border-slate-200">
                <div className="overflow-x-auto">
                  <table className="w-full text-right">
                    <thead>
                      <tr className="bg-slate-50 border-b border-slate-200">
                        <th className="px-6 py-4 font-bold text-slate-700">الاسم</th>
                        <th className="px-6 py-4 font-bold text-slate-700">البريد</th>
                        <th className="px-6 py-4 font-bold text-slate-700">الهاتف</th>
                        <th className="px-6 py-4 font-bold text-slate-700">المسابقة</th>
                        <th className="px-6 py-4 font-bold text-slate-700">التاريخ</th>
                      </tr>
                    </thead>
                    <tbody className="divide-y divide-slate-100">
                      {submissions.map((sub) => (
                        <tr key={sub.id} className="hover:bg-slate-50 transition">
                          <td className="px-6 py-4 font-medium">{sub.name}</td>
                          <td className="px-6 py-4 text-slate-600 font-mono text-sm">{sub.email}</td>
                          <td className="px-6 py-4 text-slate-600">{sub.phone}</td>
                          <td className="px-6 py-4">
                            <span className="bg-blue-50 text-blue-700 px-3 py-1 rounded-full text-xs font-bold">{sub.competition}</span>
                          </td>
                          <td className="px-6 py-4 text-slate-400 text-sm">
                            {sub.timestamp ? new Date(sub.timestamp.seconds * 1000).toLocaleDateString('ar-EG') : '...'}
                          </td>
                        </tr>
                      ))}
                    </tbody>
                  </table>
                </div>
              </div>
            </>
          )}
        </section>
      )}

      <footer className="py-12 border-t border-slate-200 text-center text-slate-500">
        <p>© 2024 مؤتمر Civil Tech. جميع الحقوق محفوظة.</p>
      </footer>
    </div>
  );
}
