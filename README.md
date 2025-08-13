# HAHGROUP

import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, onAuthStateChanged, signInAnonymously } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot } from 'firebase/firestore';

// Global variables from the Canvas environment
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Hardcoded admin password for demonstration purposes.
// In a real application, you should use a secure authentication method.
const ADMIN_PASSWORD = 'admin123';

// Main App Component
function App() {
  // State for Firebase services
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);

  // State for content and admin mode
  const [siteContent, setSiteContent] = useState(null);
  const [isAdmin, setIsAdmin] = useState(false);
  const [showLogin, setShowLogin] = useState(false);
  const [adminPassword, setAdminPassword] = useState('');
  const [message, setMessage] = useState('');
  const [showModal, setShowModal] = useState(false);
  
  // State for editable content
  const [editableContent, setEditableContent] = useState({});

  // --- Firebase Initialization and Authentication ---
  useEffect(() => {
    // Check if firebaseConfig is available
    if (Object.keys(firebaseConfig).length > 0) {
      const app = initializeApp(firebaseConfig);
      const firestore = getFirestore(app);
      const authService = getAuth(app);
      setDb(firestore);
      setAuth(authService);
      
      // Listen for authentication state changes
      const unsubscribe = onAuthStateChanged(authService, async (user) => {
        if (user) {
          // Note: No need to setUserId here as it's not used in this specific app logic
        } else {
          // If no user, sign in anonymously or with custom token
          try {
            if (initialAuthToken) {
              await signInWithCustomToken(authService, initialAuthToken);
            } else {
              await signInAnonymously(authService);
            }
          } catch (error) {
            console.error("Firebase auth error:", error);
          }
        }
        setIsAuthReady(true);
      });

      return () => unsubscribe();
    }
  }, []);

  // --- Firestore Data Fetching ---
  useEffect(() => {
    // Only proceed if Firebase is ready
    if (isAuthReady && db) {
      // Define the path to the public content document
      const docPath = `/artifacts/${appId}/public/data/siteContent/main`;
      const docRef = doc(db, docPath);

      // Listen for real-time updates to the document
      const unsubscribe = onSnapshot(docRef, (docSnap) => {
        if (docSnap.exists()) {
          const content = docSnap.data();
          setSiteContent(content);
          setEditableContent(content); // Initialize editable content
        } else {
          // If no content exists, create a default one
          const defaultContent = {
            heroTitle: 'نحقق رؤيتك في البناء والتشييد',
            heroSubtitle: 'خبرة واسعة وجودة عالية في جميع أنواع المشاريع الإنشائية. دعنا نساعدك في بناء المستقبل.',
            aboutUsTitle: 'من نحن',
            aboutUsText: 'نحن شركة مقاولات رائدة ملتزمة بتقديم حلول بناء مبتكرة وفعالة. منذ تأسيسنا، بنينا سمعة قوية على أساس الثقة، والجودة، والالتزام بالمواعيد. فريقنا من المهندسين والفنيين متخصص في تحويل الأفكار إلى واقع ملموس.',
            servicesTitle: 'خدماتنا',
            services: [
              { title: 'بناء وتشطيبات', description: 'نقدم خدمات بناء كاملة، من الأساسات وحتى التشطيبات النهائية بأعلى مستوى من الجودة.' },
              { title: 'ترميم وصيانة', description: 'متخصصون في تجديد وترميم المباني القديمة وإعادتها لحالتها الأفضل.' },
              { title: 'مقاولات عامة', description: 'نتولى جميع أنواع المشاريع العامة والخاصة بمهنية عالية.' },
            ],
            portfolioTitle: 'أعمالنا',
            portfolio: [
              { title: 'مشروع سكني', imageUrl: 'https://placehold.co/600x400/007bff/ffffff?text=مشروع+1' },
              { title: 'مبنى تجاري', imageUrl: 'https://placehold.co/600x400/007bff/ffffff?text=مشروع+2' },
              { title: 'ترميم فيلا', imageUrl: 'https://placehold.co/600x400/007bff/ffffff?text=مشروع+3' },
              { title: 'مبنى حكومي', imageUrl: 'https://placehold.co/600x400/007bff/ffffff?text=مشروع+4' },
            ],
            contactTitle: 'تواصل معنا',
            contactText: 'يسعدنا أن نسمع منك! املأ النموذج أدناه وسنتواصل معك قريباً.',
            footerText: '© 2024 شركتك للمقاولات. جميع الحقوق محفوظة.',
            companyName: 'شركتك للمقاولات',
          };
          setDoc(docRef, defaultContent, { merge: true });
        }
      }, (error) => {
        console.error("Firestore onSnapshot error:", error);
      });

      return () => unsubscribe();
    }
  }, [isAuthReady, db]);

  // --- Admin Logic ---
  const handleAdminLogin = () => {
    if (adminPassword === ADMIN_PASSWORD) {
      setIsAdmin(true);
      setShowLogin(false);
      setMessage('تم تسجيل الدخول بنجاح!');
      setShowModal(true);
    } else {
      setMessage('كلمة المرور غير صحيحة.');
      setShowModal(true);
    }
    setAdminPassword('');
  };

  const handleLogout = () => {
    setIsAdmin(false);
    setMessage('تم تسجيل الخروج بنجاح.');
    setShowModal(true);
  };
  
  const handleSave = async () => {
    if (db && isAdmin) {
      try {
        const docPath = `/artifacts/${appId}/public/data/siteContent/main`;
        const docRef = doc(db, docPath);
        await setDoc(docRef, editableContent);
        setMessage('تم حفظ التغييرات بنجاح!');
        setShowModal(true);
        // Sync the edited content with the main state
        setSiteContent(editableContent);
      } catch (error) {
        console.error("Error saving document: ", error);
        setMessage('حدث خطأ أثناء الحفظ. يرجى المحاولة مرة أخرى.');
        setShowModal(true);
      }
    }
  };

  const handleInputChange = (e, section, index, field) => {
    const { value } = e.target;
    if (section === 'services' || section === 'portfolio') {
      const newArray = [...editableContent[section]];
      newArray[index] = { ...newArray[index], [field]: value };
      setEditableContent({ ...editableContent, [section]: newArray });
    } else {
      setEditableContent({ ...editableContent, [section]: value });
    }
  };

  if (!isAuthReady || !siteContent) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <div className="text-xl font-bold">يتم تحميل المحتوى...</div>
      </div>
    );
  }

  // --- UI Components ---
  // Replaced react-icons with inline SVGs to fix the compilation error
  const SaveIcon = () => (
    <svg className="w-4 h-4 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
      <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M8 7H5a2 2 0 00-2 2v10a2 2 0 002 2h14a2 2 0 002-2V9a2 2 0 00-2-2h-3m-1 4L8 15h4v6h2v-6h4L13 11l-5 4z"></path>
    </svg>
  );

  const SignOutIcon = () => (
    <svg className="w-4 h-4 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
      <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M17 16l4-4m0 0l-4-4m4 4H7m6 4v1a3 3 0 01-3 3H5a3 3 0 01-3-3V7a3 3 0 013-3h5a3 3 0 013 3v1"></path>
    </svg>
  );

  const SignInIcon = () => (
    <svg className="w-4 h-4 ml-2" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
      <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M11 16l-4-4m0 0l4-4m-4 4h14m-1 4v1a3 3 0 01-3 3H5a3 3 0 01-3-3V7a3 3 0 013-3h5a3 3 0 013 3v1"></path>
    </svg>
  );
  
  const CloseIcon = () => (
    <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
      <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M6 18L18 6M6 6l12 12"></path>
    </svg>
  );

  const ImagePlusIcon = () => (
    <svg className="w-4 h-4 inline ml-1" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
      <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z"></path>
    </svg>
  );

  const AdminControls = () => (
    <div className="fixed top-4 left-4 bg-white p-4 rounded-lg shadow-lg flex items-center space-x-4 z-50">
      <h3 className="font-bold text-lg">وضع المسؤول</h3>
      <button onClick={handleSave} className="bg-green-500 text-white p-2 rounded-md hover:bg-green-600 transition flex items-center">
        <SaveIcon /> حفظ
      </button>
      <button onClick={handleLogout} className="bg-red-500 text-white p-2 rounded-md hover:bg-red-600 transition flex items-center">
        <SignOutIcon /> تسجيل خروج
      </button>
    </div>
  );

  const EditField = ({ field, placeholder }) => (
    <input
      type="text"
      value={editableContent[field] || ''}
      onChange={(e) => handleInputChange(e, field)}
      placeholder={placeholder}
      className="text-center w-full p-2 rounded-md border-2 border-dashed border-blue-400 focus:outline-none focus:border-blue-600 transition duration-300"
    />
  );
  
  const EditTextarea = ({ field, placeholder }) => (
    <textarea
      value={editableContent[field] || ''}
      onChange={(e) => handleInputChange(e, field)}
      placeholder={placeholder}
      className="text-center w-full p-2 rounded-md border-2 border-dashed border-blue-400 focus:outline-none focus:border-blue-600 transition duration-300 h-24"
    />
  );

  const Modal = () => (
    <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex justify-center items-center z-50">
      <div className="bg-white p-6 rounded-lg shadow-xl max-w-sm w-full text-center">
        <h3 className="text-xl font-bold mb-4">رسالة</h3>
        <p>{message}</p>
        <button onClick={() => setShowModal(false)} className="mt-4 bg-blue-600 text-white p-2 rounded-md">
          إغلاق
        </button>
      </div>
    </div>
  );

  // --- Main Render ---
  return (
    <div className="font-sans bg-gray-50 text-gray-800" dir="rtl">
      {showModal && <Modal />}
      {isAdmin && <AdminControls />}

      {/* Login Button/Form */}
      {!isAdmin && (
        <div className="fixed bottom-4 left-4 z-50">
          {!showLogin ? (
            <button
              onClick={() => setShowLogin(true)}
              className="bg-blue-600 text-white p-3 rounded-full shadow-lg hover:bg-blue-700 transition flex items-center"
            >
              <SignInIcon /> دخول المسؤول
            </button>
          ) : (
            <div className="bg-white p-4 rounded-lg shadow-lg flex items-center space-x-2">
              <input
                type="password"
                value={adminPassword}
                onChange={(e) => setAdminPassword(e.target.value)}
                placeholder="كلمة المرور"
                className="p-2 rounded-md border"
              />
              <button
                onClick={handleAdminLogin}
                className="bg-blue-600 text-white p-2 rounded-md hover:bg-blue-700"
              >
                دخول
              </button>
              <button
                onClick={() => setShowLogin(false)}
                className="bg-gray-300 text-gray-700 p-2 rounded-md hover:bg-gray-400"
              >
                <CloseIcon />
              </button>
            </div>
          )}
        </div>
      )}

      {/* الشريط العلوي */}
      <nav className="bg-white shadow-md p-4 sticky top-0 z-40">
        <div className="container mx-auto flex justify-between items-center">
          <a href="#" className="text-2xl font-bold text-blue-600">
            {isAdmin ? <EditField field="companyName" placeholder="اسم الشركة" /> : siteContent.companyName}
          </a>
          <div className="space-x-8 hidden md:flex">
            <a href="#about" className="text-gray-600 hover:text-blue-600 transition">من نحن</a>
            <a href="#services" className="text-gray-600 hover:text-blue-600 transition">خدماتنا</a>
            <a href="#portfolio" className="text-gray-600 hover:text-blue-600 transition">أعمالنا</a>
            <a href="#contact" className="text-gray-600 hover:text-blue-600 transition">اتصل بنا</a>
          </div>
        </div>
      </nav>

      {/* القسم الرئيسي (Hero) */}
      <header className="bg-blue-600 text-white py-20 md:py-32">
        <div className="container mx-auto text-center px-4">
          <h1 className="text-4xl md:text-6xl font-bold mb-4">
            {isAdmin ? <EditField field="heroTitle" placeholder="العنوان الرئيسي" /> : siteContent.heroTitle}
          </h1>
          <p className="text-lg md:text-xl mb-8 max-w-2xl mx-auto">
            {isAdmin ? <EditTextarea field="heroSubtitle" placeholder="النص التعريفي" /> : siteContent.heroSubtitle}
          </p>
          <a href="#contact" className="bg-white text-blue-600 font-bold py-3 px-8 rounded-full shadow-lg hover:bg-gray-100 transition duration-300">
            تواصل معنا الآن
          </a>
        </div>
      </header>

      {/* قسم من نحن */}
      <section id="about" className="py-16 md:py-24">
        <div className="container mx-auto px-4 text-center">
          <h2 className="text-3xl md:text-4xl font-bold mb-4 text-blue-600">
            {isAdmin ? <EditField field="aboutUsTitle" placeholder="عنوان من نحن" /> : siteContent.aboutUsTitle}
          </h2>
          <p className="text-gray-600 text-lg max-w-3xl mx-auto">
            {isAdmin ? <EditTextarea field="aboutUsText" placeholder="نص من نحن" /> : siteContent.aboutUsText}
          </p>
        </div>
      </section>

      {/* قسم خدماتنا */}
      <section id="services" className="bg-white py-16 md:py-24">
        <div className="container mx-auto px-4 text-center">
          <h2 className="text-3xl md:text-4xl font-bold mb-12 text-blue-600">
            {isAdmin ? <EditField field="servicesTitle" placeholder="عنوان خدماتنا" /> : siteContent.servicesTitle}
          </h2>
          <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
            {siteContent.services?.map((service, index) => (
              <div key={index} className="bg-gray-100 p-8 rounded-lg shadow-md">
                <h3 className="text-xl font-bold mb-2">
                  {isAdmin ? (
                    <input
                      type="text"
                      value={editableContent.services[index].title || ''}
                      onChange={(e) => handleInputChange(e, 'services', index, 'title')}
                      className="text-center w-full p-1 rounded-md border-2 border-dashed border-blue-400 focus:outline-none"
                    />
                  ) : (
                    service.title
                  )}
                </h3>
                <p className="text-gray-600">
                  {isAdmin ? (
                    <textarea
                      value={editableContent.services[index].description || ''}
                      onChange={(e) => handleInputChange(e, 'services', index, 'description')}
                      className="text-center w-full p-1 rounded-md border-2 border-dashed border-blue-400 focus:outline-none"
                    />
                  ) : (
                    service.description
                  )}
                </p>
              </div>
            ))}
          </div>
        </div>
      </section>

      {/* قسم معرض أعمالنا */}
      <section id="portfolio" className="py-16 md:py-24">
        <div className="container mx-auto px-4 text-center">
          <h2 className="text-3xl md:text-4xl font-bold mb-12 text-blue-600">
            {isAdmin ? <EditField field="portfolioTitle" placeholder="عنوان أعمالنا" /> : siteContent.portfolioTitle}
          </h2>
          <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
            {siteContent.portfolio?.map((item, index) => (
              <div key={index} className="relative overflow-hidden rounded-lg shadow-lg group">
                {isAdmin ? (
                    <div className="relative">
                        <img src={item.imageUrl} alt={item.title} className="w-full h-auto" />
                        <div className="absolute inset-0 bg-black bg-opacity-50 flex items-center justify-center opacity-100 p-4">
                            <div className="text-center w-full">
                                <input
                                    type="text"
                                    value={editableContent.portfolio[index].title || ''}
                                    onChange={(e) => handleInputChange(e, 'portfolio', index, 'title')}
                                    className="bg-white text-gray-800 text-center w-full p-1 rounded-md mb-2 border-2 border-dashed border-blue-400"
                                    placeholder="عنوان المشروع"
                                />
                                <div className="text-white text-sm flex justify-center items-center mt-2">
                                  <ImagePlusIcon />
                                  <input
                                      type="text"
                                      value={editableContent.portfolio[index].imageUrl || ''}
                                      onChange={(e) => handleInputChange(e, 'portfolio', index, 'imageUrl')}
                                      className="bg-white text-gray-800 text-center w-full p-1 rounded-md border-2 border-dashed border-blue-400"
                                      placeholder="رابط الصورة"
                                  />
                                </div>
                            </div>
                        </div>
                    </div>
                ) : (
                    <>
                        <img src={item.imageUrl} alt={item.title} className="w-full h-auto transform group-hover:scale-105 transition-transform duration-300" />
                        <div className="absolute inset-0 bg-black bg-opacity-50 flex items-center justify-center opacity-0 group-hover:opacity-100 transition-opacity duration-300">
                            <p className="text-white text-xl font-bold">{item.title}</p>
                        </div>
                    </>
                )}
              </div>
            ))}
          </div>
        </div>
      </section>

      {/* قسم اتصل بنا */}
      <section id="contact" className="bg-gray-100 py-16 md:py-24">
        <div className="container mx-auto px-4 text-center">
          <h2 className="text-3xl md:text-4xl font-bold mb-8 text-blue-600">
            {isAdmin ? <EditField field="contactTitle" placeholder="عنوان الاتصال" /> : siteContent.contactTitle}
          </h2>
          <div className="max-w-2xl mx-auto bg-white p-8 rounded-lg shadow-md">
            <p className="text-gray-600 mb-6">
              {isAdmin ? <EditTextarea field="contactText" placeholder="نص الاتصال" /> : siteContent.contactText}
            </p>
            <form>
              <div className="mb-4"><input type="text" placeholder="اسمك" className="w-full p-3 rounded-md border-gray-300 focus:ring-blue-500 focus:border-blue-500" /></div>
              <div className="mb-4"><input type="email" placeholder="بريدك الإلكتروني" className="w-full p-3 rounded-md border-gray-300 focus:ring-blue-500 focus:border-blue-500" /></div>
              <div className="mb-4"><textarea placeholder="رسالتك" rows="4" className="w-full p-3 rounded-md border-gray-300 focus:ring-blue-500 focus:border-blue-500"></textarea></div>
              <button type="submit" className="w-full bg-blue-600 text-white font-bold py-3 px-6 rounded-md hover:bg-blue-700 transition duration-300">أرسل الرسالة</button>
            </form>
          </div>
        </div>
      </section>

      {/* تذييل الصفحة (Footer) */}
      <footer className="bg-gray-800 text-white py-8">
        <div className="container mx-auto text-center px-4">
          <p>
            {isAdmin ? <EditField field="footerText" placeholder="نص التذييل" /> : siteContent.footerText}
          </p>
        </div>
      </footer>
    </div>
  );
}

export default App;
