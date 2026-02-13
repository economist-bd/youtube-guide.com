import React, { useState, useEffect, useRef } from 'react';
import { 
  BookOpen, 
  CheckCircle, 
  Clock, 
  Download, 
  MessageCircle, 
  ShieldCheck, 
  Star, 
  Menu, 
  X, 
  ChevronDown, 
  ChevronUp,
  CreditCard,
  Smartphone,
  Lock,
  Gift,
  ArrowRight
} from 'lucide-react';
import { initializeApp } from "firebase/app";
import { 
  getFirestore, 
  collection, 
  addDoc, 
  serverTimestamp 
} from "firebase/firestore";
import { 
  getAuth, 
  signInAnonymously,
  signInWithCustomToken,
  onAuthStateChanged 
} from "firebase/auth";

// --- Firebase Configuration ---
// Using environment variables provided by the platform
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const auth = getAuth(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

// --- Main App Component ---
export default function App() {
  const [user, setUser] = useState(null);
  const [view, setView] = useState('landing'); // landing, checkout, success
  const [showMobileMenu, setShowMobileMenu] = useState(false);
  const [timeLeft, setTimeLeft] = useState({ minutes: 14, seconds: 59 });
  
  // Checkout States
  const [orderStep, setOrderStep] = useState(1); // 1: Info, 2: Payment
  const [formData, setFormData] = useState({
    name: '',
    phone: '',
    email: '',
    paymentMethod: 'bkash', // bkash, nagad, card
    transactionId: ''
  });
  const [isProcessing, setIsProcessing] = useState(false);
  const [orderError, setOrderError] = useState('');

  // --- Auth & Init ---
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (error) {
        console.error("Auth initialization failed:", error);
      }
    };
    initAuth();
    
    const unsubscribe = onAuthStateChanged(auth, (u) => {
      setUser(u);
    });
    return () => unsubscribe();
  }, []);

  // --- Countdown Timer Logic ---
  useEffect(() => {
    const timer = setInterval(() => {
      setTimeLeft(prev => {
        if (prev.seconds > 0) {
          return { ...prev, seconds: prev.seconds - 1 };
        } else if (prev.minutes > 0) {
          return { minutes: prev.minutes - 1, seconds: 59 };
        } else {
          // Reset for scarcity loop
          return { minutes: 14, seconds: 59 };
        }
      });
    }, 1000);
    return () => clearInterval(timer);
  }, []);

  // --- Handlers ---
  const handleInputChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const handleCheckoutSubmit = async (e) => {
    e.preventDefault();
    if (!user) {
      setOrderError('Authentication invalid. Please refresh.');
      return;
    }
    
    setIsProcessing(true);
    setOrderError('');

    // Simulate Payment Processing
    setTimeout(async () => {
      try {
        // Save Order to Firestore
        // Path: /artifacts/{appId}/public/data/orders
        await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'orders'), {
          ...formData,
          amount: 490,
          status: 'paid', // In real app, this would be 'pending' until webhook confirms
          timestamp: serverTimestamp(),
          product: "Zero to USA Dollar Ebook",
          userId: user.uid
        });

        setIsProcessing(false);
        setView('success');
        window.scrollTo(0, 0);
        
        // Facebook Pixel Track Purchase (Simulation)
        console.log("Pixel Event: Purchase", { value: 490, currency: 'BDT' });

      } catch (err) {
        console.error(err);
        setOrderError('দুঃখিত, অর্ডার প্রসেসিং এ সমস্যা হয়েছে। আবার চেষ্টা করুন।');
        setIsProcessing(false);
      }
    }, 2000);
  };

  const scrollToSection = (id) => {
    const element = document.getElementById(id);
    if (element) element.scrollIntoView({ behavior: 'smooth' });
    setShowMobileMenu(false);
  };

  // --- Sub-Components ---

  const StickyCTA = () => (
    <div className="fixed bottom-0 left-0 w-full bg-white border-t border-gray-200 p-4 shadow-2xl z-50 md:hidden flex justify-between items-center animate-slide-up">
      <div className="text-sm">
        <p className="line-through text-gray-400 text-xs">মূল্য ৳১৫০০</p>
        <p className="font-bold text-red-600 text-xl">মাত্র ৳৪৯০</p>
      </div>
      <button 
        onClick={() => setView('checkout')}
        className="bg-red-600 text-white px-6 py-2 rounded-full font-bold shadow-lg hover:bg-red-700 transition"
      >
        এখনই কিনুন
      </button>
    </div>
  );

  const CheckoutModal = () => {
    if (view !== 'checkout') return null;

    return (
      <div className="fixed inset-0 bg-black/60 z-50 flex items-center justify-center p-4 backdrop-blur-sm overflow-y-auto">
        <div className="bg-white rounded-2xl w-full max-w-md shadow-2xl overflow-hidden my-auto">
          <div className="bg-gray-50 p-4 border-b flex justify-between items-center">
            <h3 className="font-bold text-lg text-gray-800">অর্ডার কনফার্ম করুন</h3>
            <button onClick={() => setView('landing')} className="text-gray-500 hover:text-red-500">
              <X size={24} />
            </button>
          </div>

          <div className="p-6">
            {/* Progress Bar */}
            <div className="flex mb-6">
              <div className={`flex-1 h-2 rounded-l ${orderStep >= 1 ? 'bg-green-500' : 'bg-gray-200'}`}></div>
              <div className={`flex-1 h-2 rounded-r ${orderStep >= 2 ? 'bg-green-500' : 'bg-gray-200'}`}></div>
            </div>

            <form onSubmit={handleCheckoutSubmit}>
              {orderStep === 1 && (
                <div className="space-y-4 animate-fade-in">
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">আপনার নাম</label>
                    <input 
                      required
                      type="text" 
                      name="name"
                      value={formData.name}
                      onChange={handleInputChange}
                      placeholder="এখানে নাম লিখুন"
                      className="w-full border border-gray-300 rounded-lg p-3 focus:ring-2 focus:ring-green-500 outline-none"
                    />
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">মোবাইল নাম্বার</label>
                    <input 
                      required
                      type="tel" 
                      name="phone"
                      value={formData.phone}
                      onChange={handleInputChange}
                      placeholder="017xxxxxxxx"
                      className="w-full border border-gray-300 rounded-lg p-3 focus:ring-2 focus:ring-green-500 outline-none"
                    />
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">ইমেইল এড্রেস (ডেলিভারির জন্য)</label>
                    <input 
                      required
                      type="email" 
                      name="email"
                      value={formData.email}
                      onChange={handleInputChange}
                      placeholder="example@gmail.com"
                      className="w-full border border-gray-300 rounded-lg p-3 focus:ring-2 focus:ring-green-500 outline-none"
                    />
                  </div>
                  <button 
                    type="button"
                    onClick={() => {
                      if(formData.name && formData.phone && formData.email) setOrderStep(2);
                    }}
                    className="w-full bg-green-600 text-white font-bold py-3 rounded-lg hover:bg-green-700 mt-4 flex items-center justify-center gap-2"
                  >
                    পরের ধাপ <ArrowRight size={18} />
                  </button>
                </div>
              )}

              {orderStep === 2 && (
                <div className="space-y-4 animate-fade-in">
                  <div className="bg-blue-50 p-3 rounded border border-blue-200 text-sm text-blue-800 mb-4">
                    সর্বমোট পেমেন্ট: <strong>৳৪৯০.০০</strong>
                  </div>

                  <p className="text-sm font-medium text-gray-700">পেমেন্ট মেথড সিলেক্ট করুন:</p>
                  <div className="grid grid-cols-3 gap-2">
                    {['bkash', 'nagad', 'card'].map((method) => (
                      <div 
                        key={method}
                        onClick={() => setFormData({...formData, paymentMethod: method})}
                        className={`cursor-pointer border-2 rounded-lg p-2 text-center flex flex-col items-center justify-center h-20 transition ${formData.paymentMethod === method ? 'border-pink-500 bg-pink-50' : 'border-gray-200 hover:border-gray-300'}`}
                      >
                         {/* Icons Simulation */}
                         {method === 'bkash' && <span className="font-bold text-pink-600">bKash</span>}
                         {method === 'nagad' && <span className="font-bold text-orange-600">Nagad</span>}
                         {method === 'card' && <CreditCard className="text-blue-600" />}
                      </div>
                    ))}
                  </div>

                  <div className="bg-gray-100 p-4 rounded text-sm text-gray-600 mt-4">
                    {formData.paymentMethod === 'bkash' && (
                      <p>আপনার bKash অ্যাপ থেকে <strong>017XXXXXXXX</strong> নম্বরে ৳৪৯০ সেন্ড মানি করুন। এরপর নিচের বক্সে TrxID দিন।</p>
                    )}
                    {formData.paymentMethod === 'nagad' && (
                      <p>আপনার Nagad অ্যাপ থেকে <strong>017XXXXXXXX</strong> নম্বরে ৳৪৯০ সেন্ড মানি করুন। এরপর নিচের বক্সে TrxID দিন।</p>
                    )}
                    {formData.paymentMethod === 'card' && (
                      <p>নিচে আপনার কার্ডের তথ্য দিন (Secure Payment Gateway via SSLCommerz/Stripe)।</p>
                    )}
                  </div>

                  {formData.paymentMethod !== 'card' && (
                    <div>
                      <label className="block text-sm font-medium text-gray-700 mb-1">Transaction ID (TrxID)</label>
                      <input 
                        required
                        type="text" 
                        name="transactionId"
                        value={formData.transactionId}
                        onChange={handleInputChange}
                        placeholder="e.g. 8JKS99D2"
                        className="w-full border border-gray-300 rounded-lg p-3 focus:ring-2 focus:ring-green-500 outline-none uppercase"
                      />
                    </div>
                  )}

                  <div className="flex gap-3 mt-6">
                    <button 
                      type="button"
                      onClick={() => setOrderStep(1)}
                      className="w-1/3 border border-gray-300 text-gray-600 font-bold py-3 rounded-lg hover:bg-gray-50"
                    >
                      পেছনে
                    </button>
                    <button 
                      type="submit"
                      disabled={isProcessing}
                      className="w-2/3 bg-green-600 text-white font-bold py-3 rounded-lg hover:bg-green-700 flex items-center justify-center gap-2"
                    >
                      {isProcessing ? 'প্রসেসিং হচ্ছে...' : 'পেমেন্ট নিশ্চিত করুন'}
                    </button>
                  </div>
                  {orderError && <p className="text-red-500 text-sm mt-2 text-center">{orderError}</p>}
                </div>
              )}
            </form>
          </div>
          
          <div className="bg-gray-50 px-6 py-3 text-center border-t">
             <div className="flex items-center justify-center gap-2 text-xs text-gray-500">
               <Lock size={12} /> 100% Secure & Encrypted Payment
             </div>
          </div>
        </div>
      </div>
    );
  };

  const SuccessView = () => (
    <div className="min-h-screen bg-gray-50 flex flex-col items-center justify-center p-4">
      <div className="bg-white max-w-lg w-full rounded-2xl shadow-xl p-8 text-center animate-scale-in">
        <div className="w-20 h-20 bg-green-100 rounded-full flex items-center justify-center mx-auto mb-6">
          <CheckCircle className="text-green-600 w-10 h-10" />
        </div>
        <h1 className="text-2xl font-bold text-gray-800 mb-2">অভিনন্দন, {formData.name}!</h1>
        <p className="text-gray-600 mb-6">আপনার পেমেন্ট সফল হয়েছে। নিচের বাটনে ক্লিক করে ইবুকটি ডাউনলোড করুন।</p>
        
        <div className="bg-blue-50 border border-blue-200 rounded-lg p-4 mb-6">
          <p className="text-sm text-blue-800">আপনার ইমেইল ({formData.email})-এ বইয়ের ডাউনলোড লিংক এবং ইনভয়েস পাঠানো হয়েছে।</p>
        </div>

        <a 
          href="#" 
          onClick={(e) => e.preventDefault()} // Placeholder
          className="block w-full bg-green-600 text-white font-bold py-4 rounded-lg hover:bg-green-700 shadow-lg flex items-center justify-center gap-2 mb-4"
        >
          <Download size={20} /> ইবুক ডাউনলোড করুন (PDF)
        </a>
        
        <button 
           onClick={() => window.open('https://wa.me/8801700000000', '_blank')}
           className="text-green-600 font-medium hover:underline flex items-center justify-center gap-2"
        >
          <MessageCircle size={16} /> কোনো সমস্যা? হোয়াটসঅ্যাপে নক দিন
        </button>
      </div>
    </div>
  );

  // --- Landing Page Layout ---
  if (view === 'success') return <SuccessView />;

  return (
    <div className="font-sans text-gray-900 bg-white selection:bg-green-100 selection:text-green-900">
      
      {/* Top Banner (Scarcity) */}
      <div className="bg-red-600 text-white text-center py-2 px-4 text-sm font-medium sticky top-0 z-40">
        স্পেশাল ডিসকাউন্ট শেষ হতে বাকি: <span className="font-bold font-mono ml-1">{timeLeft.minutes}:{timeLeft.seconds < 10 ? `0${timeLeft.seconds}` : timeLeft.seconds}</span> মিনিট
      </div>

      {/* Navigation */}
      <nav className="border-b bg-white relative z-30">
        <div className="max-w-6xl mx-auto px-4 py-4 flex justify-between items-center">
          <div className="flex items-center gap-2">
             <BookOpen className="text-green-600" />
             <span className="font-bold text-xl tracking-tight">PassiveIncomeBD</span>
          </div>
          
          {/* Desktop Menu */}
          <div className="hidden md:flex gap-6 text-sm font-medium text-gray-600">
            <button onClick={() => scrollToSection('benefits')} className="hover:text-green-600">সুবিধাসমূহ</button>
            <button onClick={() => scrollToSection('reviews')} className="hover:text-green-600">রিভিউ</button>
            <button onClick={() => scrollToSection('faq')} className="hover:text-green-600">প্রশ্নোত্তর</button>
          </div>

          <button 
            onClick={() => setView('checkout')}
            className="hidden md:block bg-green-600 text-white px-6 py-2 rounded-full font-bold hover:bg-green-700 transition"
          >
            ইবুকটি কিনুন
          </button>

          {/* Mobile Menu Icon */}
          <button className="md:hidden" onClick={() => setShowMobileMenu(!showMobileMenu)}>
            {showMobileMenu ? <X /> : <Menu />}
          </button>
        </div>

        {/* Mobile Dropdown */}
        {showMobileMenu && (
          <div className="md:hidden bg-white border-b absolute w-full left-0 top-full p-4 flex flex-col gap-4 shadow-lg">
             <button onClick={() => scrollToSection('benefits')} className="text-left py-2 border-b">সুবিধাসমূহ</button>
             <button onClick={() => scrollToSection('reviews')} className="text-left py-2 border-b">রিভিউ</button>
             <button onClick={() => scrollToSection('faq')} className="text-left py-2 border-b">প্রশ্নোত্তর</button>
          </div>
        )}
      </nav>

      {/* Hero Section */}
      <header className="relative overflow-hidden bg-gradient-to-b from-green-50 to-white pt-16 pb-20 px-4">
         <div className="max-w-4xl mx-auto text-center">
            <span className="inline-block bg-green-100 text-green-700 px-4 py-1 rounded-full text-sm font-bold mb-6">
              🔥 ২০০০+ কপি বিক্রি হয়েছে গত সপ্তাহে
            </span>
            <h1 className="text-4xl md:text-6xl font-extrabold leading-tight mb-6 text-gray-900">
              <span className="text-green-600">ইউটিউব</span> থেকে প্রতি মাসে <br className="hidden md:block"/> 
              ১০০০+ ডলার আয়ের সিক্রেট ব্লুপ্রিন্ট
            </h1>
            <p className="text-lg md:text-xl text-gray-600 mb-8 max-w-2xl mx-auto leading-relaxed">
              কোনো ক্যামেরা বা দামি গিয়ার ছাড়াই, শুধুমাত্র মোবাইল ব্যবহার করে বিদেশি চ্যানেল তৈরির A-Z গাইডলাইন। বেকারত্ব দূর করুন, ডলার আয় করুন।
            </p>
            
            <div className="flex flex-col md:flex-row gap-4 justify-center items-center mb-10">
              <button 
                 onClick={() => setView('checkout')}
                 className="w-full md:w-auto bg-red-600 text-white text-lg px-8 py-4 rounded-lg font-bold shadow-lg hover:bg-red-700 hover:scale-105 transition transform flex items-center justify-center gap-2"
              >
                ডাউনলোড করুন - মূল্য ৳৪৯০ <ArrowRight size={20} />
              </button>
              <p className="text-sm text-gray-500">মূল্য ৳১৫০০ | অফারটি সীমিত সময়ের জন্য</p>
            </div>

            {/* Video Placeholder */}
            <div className="relative aspect-video max-w-3xl mx-auto bg-gray-900 rounded-2xl shadow-2xl flex items-center justify-center text-white border-4 border-white">
               <div className="text-center">
                 <div className="w-16 h-16 bg-white/20 rounded-full flex items-center justify-center mx-auto backdrop-blur-sm mb-4">
                   <div className="w-0 h-0 border-t-[10px] border-t-transparent border-l-[20px] border-l-white border-b-[10px] border-b-transparent ml-1"></div>
                 </div>
                 <p className="font-bold">প্রমোশনাল ভিডিও দেখুন</p>
               </div>
            </div>
         </div>
      </header>

      {/* Trust Badges */}
      <section className="bg-gray-50 py-8 border-y">
        <div className="max-w-6xl mx-auto px-4 flex flex-wrap justify-center gap-8 md:gap-16 opacity-70 grayscale hover:grayscale-0 transition duration-500">
           {/* Placeholders for logos like bKash, Google, YouTube, Payoneer */}
           <div className="font-bold text-xl flex items-center gap-2"><ShieldCheck /> ১০০% মানিব্যাক গ্যারান্টি</div>
           <div className="font-bold text-xl flex items-center gap-2"><Smartphone /> মোবাইল ফ্রেন্ডলি</div>
           <div className="font-bold text-xl flex items-center gap-2"><Clock /> ইনস্ট্যান্ট ডাউনলোড</div>
        </div>
      </section>

      {/* Problem/Solution Section */}
      <section className="py-16 px-4 max-w-4xl mx-auto">
        <h2 className="text-3xl font-bold text-center mb-12">কেন আপনি শুরু করতে পারছেন না?</h2>
        <div className="grid md:grid-cols-2 gap-8">
           <div className="bg-red-50 p-6 rounded-xl border border-red-100">
             <h3 className="font-bold text-red-700 text-xl mb-4 flex items-center gap-2"><X className="bg-red-200 rounded-full p-1" /> সমস্যাগুলো</h3>
             <ul className="space-y-3 text-gray-700">
               <li>❌ ক্যামেরার সামনে কথা বলতে লজ্জা পান।</li>
               <li>❌ ভালো ইংরেজি জানেন না।</li>
               <li>❌ ভিডিও এডিটিং এর দামী পিসি নেই।</li>
               <li>❌ সঠিক গাইডলাইনের অভাবে চ্যানেল গ্রো করছে না।</li>
             </ul>
           </div>
           <div className="bg-green-50 p-6 rounded-xl border border-green-100">
             <h3 className="font-bold text-green-700 text-xl mb-4 flex items-center gap-2"><CheckCircle className="bg-green-200 rounded-full p-1" /> আমাদের সমাধান</h3>
             <ul className="space-y-3 text-gray-700">
               <li>✅ ফেসল্যাস (Faceless) চ্যানেল আইডিয়া।</li>
               <li>✅ AI দিয়ে স্ক্রিপ্ট ও ভয়েস জেনারেশন।</li>
               <li>✅ মোবাইল দিয়েই প্রফেশনাল এডিটিং।</li>
               <li>✅ SEO এবং ভাইরাল হওয়ার সিক্রেট মেথড।</li>
             </ul>
           </div>
        </div>
      </section>

      {/* What's Inside (Modules) */}
      <section id="benefits" className="py-16 bg-gray-900 text-white px-4">
        <div className="max-w-5xl mx-auto">
          <h2 className="text-3xl md:text-4xl font-bold text-center mb-12">এই ইবুকে যা যা থাকছে</h2>
          <div className="grid md:grid-cols-3 gap-6">
            {[
              { title: "AI কন্টেন্ট মাস্টারি", desc: "চ্যাটজিপিটি ব্যবহার করে অটোমেটেড স্ক্রিপ্ট এবং ভিডিও আইডিয়া বের করার পদ্ধতি।" },
              { title: "মোবাইল ভিডিও এডিটিং", desc: "ক্যাপকাট এবং ইনশট দিয়ে পিসির মতো প্রফেশনাল এডিটিং টিউটোরিয়াল।" },
              { title: "ডলার ইনকাম মেথড", desc: "মনিটাইজেশন ছাড়াও স্পন্সরশিপ এবং এফিলিয়েট থেকে আয়ের উপায়।" },
              { title: "USA অডিয়েন্স টার্গেট", desc: "বাংলাদেশ থেকে আমেরিকায় ভিডিও র‍্যাংক করানোর গোপন অ্যালগরিদম হ্যাক।" },
              { title: "কপিরাইট ফ্রি রিসোর্স", desc: "কোথা থেকে ফ্রি ভিডিও, মিউজিক এবং ছবি পাবেন তার বিশাল লিস্ট।" },
              { title: "ব্যাংক উইথড্রল", desc: "টাকা সরাসরি বাংলাদেশের ব্যাংকে আনার সম্পূর্ণ গাইডলাইন।" },
            ].map((item, i) => (
              <div key={i} className="bg-gray-800 p-6 rounded-xl hover:bg-gray-700 transition duration-300 border border-gray-700">
                <div className="w-10 h-10 bg-green-600 rounded-lg flex items-center justify-center font-bold text-xl mb-4">{i+1}</div>
                <h3 className="font-bold text-xl mb-2">{item.title}</h3>
                <p className="text-gray-400 text-sm">{item.desc}</p>
              </div>
            ))}
          </div>
        </div>
      </section>

      {/* Bonuses */}
      <section className="py-16 px-4 bg-yellow-50">
        <div className="max-w-4xl mx-auto">
           <h2 className="text-3xl font-bold text-center mb-8">🎁 সাথে থাকছে স্পেশাল বোনাস (ফ্রি)</h2>
           <div className="bg-white p-6 rounded-xl shadow-lg border-2 border-yellow-400 flex flex-col md:flex-row items-center gap-6">
             <div className="bg-yellow-100 p-4 rounded-full">
               <Gift className="text-yellow-600 w-12 h-12" />
             </div>
             <div>
               <h3 className="font-bold text-xl text-gray-800">বোনাস ১: ১০০টি ভাইরাল ভিডিও আইডিয়া লিস্ট</h3>
               <p className="text-gray-600 mt-2">যেই টপিকগুলো এখন ট্রেন্ডিংয়ে আছে, তার রেডিমেড লিস্ট।</p>
               <div className="mt-4 inline-block bg-yellow-200 text-yellow-800 px-3 py-1 text-xs font-bold rounded uppercase">মূল্য ৫০০ টাকা (এখন ফ্রি)</div>
             </div>
           </div>
        </div>
      </section>

      {/* Pricing Section */}
      <section className="py-16 px-4">
        <div className="max-w-3xl mx-auto text-center bg-white rounded-3xl shadow-2xl overflow-hidden border border-gray-100">
           <div className="bg-green-600 text-white py-4">
             <h3 className="text-xl font-bold">LIMITED TIME OFFER</h3>
           </div>
           <div className="p-8 md:p-12">
             <h2 className="text-4xl font-extrabold text-gray-900 mb-4">আজকের অফার প্রাইস</h2>
             <div className="flex items-center justify-center gap-4 mb-6">
               <span className="text-2xl text-gray-400 line-through">৳১৫০০</span>
               <span className="text-5xl font-bold text-red-600">৳৪৯০</span>
             </div>
             <p className="text-gray-600 mb-8">জীবনের মোড় ঘুরিয়ে দেওয়ার জন্য ৪৯০ টাকা ইনভেস্টমেন্ট কিছুই না। এখনই সিদ্ধান্ত নিন।</p>
             
             <ul className="text-left max-w-sm mx-auto space-y-3 mb-8">
               <li className="flex items-center gap-2"><CheckCircle size={18} className="text-green-500"/> লাইফটাইম অ্যাক্সেস</li>
               <li className="flex items-center gap-2"><CheckCircle size={18} className="text-green-500"/> ফ্রি আপডেট</li>
               <li className="flex items-center gap-2"><CheckCircle size={18} className="text-green-500"/> প্রাইভেট সাপোর্ট গ্রুপ</li>
             </ul>

             <button 
                onClick={() => setView('checkout')}
                className="w-full bg-green-600 text-white text-xl px-8 py-4 rounded-lg font-bold shadow-lg hover:bg-green-700 transition transform hover:-translate-y-1"
             >
               ডাউনলোড করুন (৳৪৯০)
             </button>
             <p className="mt-4 text-xs text-gray-400">Secure Payment via bKash/Nagad</p>
           </div>
        </div>
      </section>

      {/* FAQ Section */}
      <section id="faq" className="py-16 px-4 bg-gray-50">
        <div className="max-w-3xl mx-auto">
          <h2 className="text-3xl font-bold text-center mb-10">সচরাচর জিজ্ঞাসা (FAQ)</h2>
          <div className="space-y-4">
            {[
              { q: "আমি কি মোবাইল দিয়ে কাজ করতে পারব?", a: "হ্যাঁ, ১০০%। এই গাইডলাইনটিই মোবাইল ইউজারদের কথা মাথায় রেখে তৈরি করা হয়েছে।" },
              { q: "পেমেন্ট করার পর বই পাবো কীভাবে?", a: "পেমেন্ট কমপ্লিট হওয়ার সাথে সাথেই আপনাকে ডাউনলোড পেজে নিয়ে যাওয়া হবে এবং ইমেইলে লিংক পাঠানো হবে।" },
              { q: "আমি কি পরে সাপোর্ট পাবো?", a: "হ্যাঁ, আমাদের একটি সিক্রেট টেলিগ্রাম গ্রুপ আছে যেখানে আপনি সাপোর্ট পাবেন।" },
            ].map((faq, index) => (
              <details key={index} className="bg-white p-4 rounded-lg shadow-sm group">
                <summary className="font-bold text-lg cursor-pointer list-none flex justify-between items-center text-gray-800">
                  {faq.q}
                  <ChevronDown className="group-open:rotate-180 transition-transform"/>
                </summary>
                <p className="mt-3 text-gray-600 leading-relaxed pl-4 border-l-2 border-green-500">{faq.a}</p>
              </details>
            ))}
          </div>
        </div>
      </section>

      {/* Footer */}
      <footer className="bg-gray-900 text-gray-400 py-12 px-4 text-center">
        <div className="max-w-4xl mx-auto mb-8 flex justify-center gap-6">
           <a href="#" className="hover:text-white">Terms & Conditions</a>
           <a href="#" className="hover:text-white">Privacy Policy</a>
           <a href="#" className="hover:text-white">Refund Policy</a>
        </div>
        <p>© 2026 PassiveIncomeBD by Manjurul Haque. All rights reserved.</p>
        <div className="mt-4 text-xs text-gray-600">
           Disclaimer: This site is not a part of the YouTube website or Google Inc.
        </div>
      </footer>

      {/* Floating Elements */}
      <a 
        href="https://wa.me/8801700000000" 
        target="_blank" 
        rel="noreferrer"
        className="fixed bottom-20 md:bottom-8 right-4 md:right-8 bg-green-500 text-white p-4 rounded-full shadow-2xl hover:bg-green-600 z-40 transition-transform hover:scale-110"
      >
        <MessageCircle size={28} />
      </a>
      
      <StickyCTA />
      <CheckoutModal />

    </div>
  );
}