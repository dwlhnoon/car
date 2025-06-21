import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, getDoc, collection, query, where, getDocs, serverTimestamp } from 'firebase/firestore';

// Global variables for Firebase configuration (provided by the environment)
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

function App() {
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [currentView, setCurrentView] = useState('capture'); // 'capture' or 'search'

  // Initialize Firebase and set up authentication
  useEffect(() => {
    async function initializeFirebase() {
      try {
        const app = initializeApp(firebaseConfig);
        const firestoreDb = getFirestore(app);
        const firebaseAuth = getAuth(app);

        setDb(firestoreDb);
        setAuth(firebaseAuth);

        // Sign in user
        if (initialAuthToken) {
          await signInWithCustomToken(firebaseAuth, initialAuthToken);
        } else {
          await signInAnonymously(firebaseAuth);
        }

        // Listen for auth state changes
        onAuthStateChanged(firebaseAuth, (user) => {
          if (user) {
            setUserId(user.uid);
            console.log("Firebase Auth Ready. User ID:", user.uid);
          } else {
            console.log("No user signed in.");
            setUserId(null);
          }
          setLoading(false);
        });
      } catch (e) {
        console.error("Error initializing Firebase:", e);
        setError("Failed to initialize the application. Please try again later.");
        setLoading(false);
      }
    }

    initializeFirebase();
  }, []);

  if (loading) {
    return (
      <div className="flex justify-center items-center h-screen bg-gray-100">
        <div className="text-xl font-semibold text-gray-700">جاري التحميل...</div>
      </div>
    );
  }

  if (error) {
    return (
      <div className="flex justify-center items-center h-screen bg-red-100 text-red-700 p-4 rounded-lg">
        <div className="text-xl font-semibold">{error}</div>
      </div>
    );
  }

  if (!db || !auth || !userId) {
    return (
      <div className="flex justify-center items-center h-screen bg-yellow-100 text-yellow-700 p-4 rounded-lg">
        <div className="text-xl font-semibold">جاري تهيئة خدمات التطبيق...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-amber-50 to-amber-100 font-inter p-4 sm:p-8 flex flex-col items-center">
      <h1 className="text-4xl font-extrabold text-amber-800 mb-8 mt-4 text-center">
        نون لتأجير السيارات
      </h1>

      <div className="flex space-x-4 mb-8">
        <button
          onClick={() => setCurrentView('capture')}
          className={`px-6 py-3 rounded-full shadow-lg transition-all duration-300 ease-in-out
                      ${currentView === 'capture' ? 'bg-amber-600 text-white transform scale-105' : 'bg-white text-amber-700 hover:bg-amber-100'}`}
        >
          التقاط صور
        </button>
        <button
          onClick={() => setCurrentView('search')}
          className={`px-6 py-3 rounded-full shadow-lg transition-all duration-300 ease-in-out
                      ${currentView === 'search' ? 'bg-amber-600 text-white transform scale-105' : 'bg-white text-amber-700 hover:bg-amber-100'}`}
        >
          البحث عن مركبة
        </button>
      </div>

      <div className="w-full max-w-4xl bg-white rounded-xl shadow-2xl p-6 sm:p-10">
        {currentView === 'capture' && <VehicleCapture db={db} userId={userId} />}
        {currentView === 'search' && <VehicleSearch db={db} userId={userId} />}
      </div>
      <p className="mt-8 text-gray-600 text-sm">معرف المستخدم: {userId}</p>
    </div>
  );
}

// Component for capturing vehicle photos and data
function VehicleCapture({ db, userId }) {
  const videoRef = useRef(null);
  const canvasRef = useRef(null);
  const [stream, setStream] = useState(null);
  const [capturedImages, setCapturedImages] = useState([]);
  const [licensePlate, setLicensePlate] = useState('');
  const [employeeName, setEmployeeName] = useState('');
  const [contractNumber, setContractNumber] = useState('');
  const [message, setMessage] = useState('');
  const [messageType, setMessageType] = useState(''); // 'success' or 'error'
  const [isCameraActive, setIsCameraActive] = useState(false);

  // Start camera on component mount
  useEffect(() => {
    // A small delay to ensure the video ref is attached to the DOM element
    const timer = setTimeout(() => {
      startCamera();
    }, 100); // 100ms delay

    return () => {
      stopCamera();
      clearTimeout(timer); // Clear the timeout if component unmounts
    };
  }, []);

  const startCamera = async () => {
    setMessage('');
    try {
      if (!videoRef.current) {
        console.warn("Video ref is not available yet, attempting to re-initiate start camera.");
        setMessage("عذرًا، الكاميرا غير جاهزة بعد. يرجى المحاولة مرة أخرى أو النقر على 'تشغيل الكاميرا'.", 'error');
        setMessageType('error');
        return;
      }
      const mediaStream = await navigator.mediaDevices.getUserMedia({ video: { width: 1280, height: 720, facingMode: 'environment' } });
      videoRef.current.srcObject = mediaStream;
      setStream(mediaStream);
      setIsCameraActive(true);
    } catch (err) {
      console.error("Error accessing camera:", err);
      setMessage("تعذر الوصول إلى الكاميرا. يرجى التأكد من السماح بالوصول.", 'error');
      setMessageType('error');
    }
  };

  const stopCamera = () => {
    if (stream) {
      stream.getTracks().forEach(track => track.stop());
      setStream(null);
      setIsCameraActive(false);
    }
  };

  const takePhoto = () => {
    if (!videoRef.current || !canvasRef.current) return;

    const video = videoRef.current;
    const canvas = canvasRef.current;
    const context = canvas.getContext('2d');

    // Set canvas dimensions to match video to avoid stretching
    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;

    context.drawImage(video, 0, 0, canvas.width, canvas.height);
    const imageDataUrl = canvas.toDataURL('image/jpeg', 0.7); // Quality 0.7 for smaller file size
    setCapturedImages([...capturedImages, imageDataUrl]);
  };

  const removeImage = (indexToRemove) => {
    setCapturedImages(capturedImages.filter((_, index) => index !== indexToRemove));
  };

  const saveVehicleData = async () => {
    if (!licensePlate || !employeeName || !contractNumber || capturedImages.length === 0) {
      setMessage("الرجاء تعبئة جميع الحقول والتقاط صورة واحدة على الأقل.", 'error');
      setMessageType('error');
      return;
    }

    // Warn about image size if too many images are captured
    const totalImageSize = capturedImages.reduce((sum, img) => sum + img.length, 0); // rough estimate in bytes for base64
    if (totalImageSize > 800 * 1024) { // Check if > 800KB (leaving some buffer for other fields)
      setMessage("تحذير: حجم الصور كبير جدًا وقد يتجاوز حد 1MB في Firestore. يرجى محاولة التقاط صور أقل أو بجودة أقل.", 'error');
      setMessageType('error');
      // Still proceed, but warn the user.
    }

    try {
      const vehicleRef = doc(db, `/artifacts/${appId}/users/${userId}/vehicles`, licensePlate);
      await setDoc(vehicleRef, {
        licensePlate: licensePlate.toUpperCase(), // Store in uppercase for consistent search
        employeeName,
        contractNumber,
        vehicleNumber: licensePlate.toUpperCase(), // Vehicle number can be same as license plate
        images: capturedImages, // Array of base64 image strings
        timestamp: serverTimestamp(),
      });
      setMessage("تم حفظ بيانات المركبة بنجاح!", 'success');
      setMessageType('success');
      // Clear form after successful save
      setLicensePlate('');
      setEmployeeName('');
      setContractNumber('');
      setCapturedImages([]);
    } catch (e) {
      console.error("Error saving document: ", e);
      setMessage("حدث خطأ أثناء حفظ البيانات. يرجى المحاولة مرة أخرى.", 'error');
      setMessageType('error');
    }
  };

  return (
    <div className="space-y-6">
      <h2 className="text-3xl font-bold text-gray-800 text-center mb-6">التقاط صور المركبة</h2>

      {message && (
        <div className={`p-3 rounded-lg text-center ${messageType === 'success' ? 'bg-green-100 text-green-700' : 'bg-red-100 text-red-700'}`}>
          {message}
        </div>
      )}

      <div className="flex flex-col sm:flex-row gap-6">
        <div className="sm:w-1/2">
          <label htmlFor="licensePlate" className="block text-gray-700 font-semibold mb-2">رقم لوحة السيارة:</label>
          <input
            type="text"
            id="licensePlate"
            className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-amber-500 focus:border-transparent transition duration-200"
            value={licensePlate}
            onChange={(e) => setLicensePlate(e.target.value)}
            placeholder="مثال: أ ب ج 1234"
            required
          />
        </div>
        <div className="sm:w-1/2">
          <label htmlFor="employeeName" className="block text-gray-700 font-semibold mb-2">اسم الموظف:</label>
          <input
            type="text"
            id="employeeName"
            className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-amber-500 focus:border-transparent transition duration-200"
            value={employeeName}
            onChange={(e) => setEmployeeName(e.target.value)}
            placeholder="أدخل اسم الموظف"
            required
          />
        </div>
      </div>

      <div className="w-full">
        <label htmlFor="contractNumber" className="block text-gray-700 font-semibold mb-2">رقم العقد:</label>
        <input
          type="text"
          id="contractNumber"
          className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-amber-500 focus:border-transparent transition duration-200"
          value={contractNumber}
          onChange={(e) => setContractNumber(e.target.value)}
          placeholder="أدخل رقم العقد"
          required
        />
      </div>

      <div className="relative border border-gray-300 rounded-lg overflow-hidden bg-gray-200 flex justify-center items-center h-64">
        {isCameraActive ? (
          <video ref={videoRef} autoPlay playsInline className="w-full h-full object-cover"></video>
        ) : (
          <div className="text-gray-500 text-lg">الكاميرا غير نشطة.</div>
        )}
        <canvas ref={canvasRef} className="hidden"></canvas>
      </div>

      <div className="flex justify-center gap-4">
        <button
          onClick={takePhoto}
          disabled={!isCameraActive}
          className="bg-blue-500 hover:bg-blue-600 text-white font-bold py-3 px-6 rounded-full shadow-md transition-transform transform hover:scale-105 disabled:opacity-50 disabled:cursor-not-allowed"
        >
          التقاط صورة
        </button>
        <button
          onClick={startCamera}
          disabled={isCameraActive}
          className="bg-green-500 hover:bg-green-600 text-white font-bold py-3 px-6 rounded-full shadow-md transition-transform transform hover:scale-105 disabled:opacity-50 disabled:cursor-not-allowed"
        >
          تشغيل الكاميرا
        </button>
        <button
          onClick={stopCamera}
          disabled={!isCameraActive}
          className="bg-red-500 hover:bg-red-600 text-white font-bold py-3 px-6 rounded-full shadow-md transition-transform transform hover:scale-105 disabled:opacity-50 disabled:cursor-not-allowed"
        >
          إيقاف الكاميرا
        </button>
      </div>

      {capturedImages.length > 0 && (
        <div className="mt-6">
          <h3 className="text-xl font-bold text-gray-800 mb-4">الصور الملتقطة:</h3>
          <div className="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 gap-4">
            {capturedImages.map((image, index) => (
              <div key={index} className="relative group rounded-lg overflow-hidden shadow-md">
                <img src={image} alt={`Captured Vehicle ${index + 1}`} className="w-full h-32 object-cover" />
                <button
                  onClick={() => removeImage(index)}
                  className="absolute top-1 right-1 bg-red-600 text-white rounded-full p-1 text-xs opacity-0 group-hover:opacity-100 transition-opacity duration-300"
                  title="إزالة الصورة"
                >
                  <svg xmlns="http://www.w3.org/2000/svg" className="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
                  </svg>
                </button>
              </div>
            ))}
          </div>
        </div>
      )}

      <div className="flex justify-center mt-8">
        <button
          onClick={saveVehicleData}
          className="bg-amber-600 hover:bg-amber-700 text-white font-bold py-4 px-8 rounded-full shadow-lg transition-transform transform hover:scale-105 focus:outline-none focus:ring-4 focus:ring-amber-300"
        >
          حفظ بيانات المركبة
        </button>
      </div>
    </div>
  );
}

// Component for searching vehicle data
function VehicleSearch({ db, userId }) {
  const [searchLicensePlate, setSearchLicensePlate] = useState('');
  const [searchResults, setSearchResults] = useState(null);
  const [message, setMessage] = useState('');
  const [messageType, setMessageType] = useState(''); // 'success' or 'error'

  const handleSearch = async () => {
    if (!searchLicensePlate) {
      setMessage("الرجاء إدخال رقم لوحة السيارة للبحث.", 'error');
      setMessageType('error');
      return;
    }

    setMessage('');
    setSearchResults(null); // Clear previous results
    try {
      // Query Firestore for the document
      const docRef = doc(db, `/artifacts/${appId}/users/${userId}/vehicles`, searchLicensePlate.toUpperCase());
      const docSnap = await getDoc(docRef);

      if (docSnap.exists()) {
        setSearchResults(docSnap.data());
        setMessage("تم العثور على بيانات المركبة.", 'success');
        setMessageType('success');
      } else {
        setMessage("لم يتم العثور على مركبة بهذا الرقم.", 'error');
        setMessageType('error');
      }
    } catch (e) {
      console.error("Error searching document: ", e);
      setMessage("حدث خطأ أثناء البحث عن البيانات. يرجى المحاولة مرة أخرى.", 'error');
      setMessageType('error');
    }
  };

  return (
    <div className="space-y-6">
      <h2 className="text-3xl font-bold text-gray-800 text-center mb-6">البحث عن مركبة</h2>

      {message && (
        <div className={`p-3 rounded-lg text-center ${messageType === 'success' ? 'bg-green-100 text-green-700' : 'bg-red-100 text-red-700'}`}>
          {message}
        </div>
      )}

      <div className="flex flex-col sm:flex-row gap-4 items-end">
        <div className="flex-grow">
          <label htmlFor="searchLicensePlate" className="block text-gray-700 font-semibold mb-2">رقم لوحة السيارة:</label>
          <input
            type="text"
            id="searchLicensePlate"
            className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-amber-500 focus:border-transparent transition duration-200"
            value={searchLicensePlate}
            onChange={(e) => setSearchLicensePlate(e.target.value)}
            placeholder="مثال: أ ب ج 1234"
          />
        </div>
        <button
          onClick={handleSearch}
          className="bg-amber-600 hover:bg-amber-700 text-white font-bold py-3 px-6 rounded-full shadow-md transition-transform transform hover:scale-105 focus:outline-none focus:ring-4 focus:ring-amber-300 w-full sm:w-auto"
        >
          بحث
        </button>
      </div>

      {searchResults && (
        <div className="mt-8 p-6 bg-gray-50 rounded-xl shadow-inner">
          <h3 className="text-2xl font-bold text-gray-800 mb-4">نتائج البحث:</h3>
          <div className="space-y-3 text-lg text-gray-700">
            <p><strong>رقم لوحة السيارة:</strong> {searchResults.licensePlate}</p>
            <p><strong>اسم الموظف:</strong> {searchResults.employeeName}</p>
            <p><strong>رقم العقد:</strong> {searchResults.contractNumber}</p>
            <p><strong>رقم المركبة:</strong> {searchResults.vehicleNumber}</p>
            <p className="text-sm text-gray-500">
              <span className="font-semibold">تاريخ الحفظ:</span> {new Date(searchResults.timestamp?.toDate()).toLocaleString('ar-SA')}
            </p>
          </div>

          {searchResults.images && searchResults.images.length > 0 && (
            <div className="mt-6">
              <h4 className="text-xl font-bold text-gray-800 mb-4">الصور:</h4>
              <div className="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 gap-4">
                {searchResults.images.map((image, index) => (
                  <div key={index} className="rounded-lg overflow-hidden shadow-md">
                    <img src={image} alt={`Vehicle Image ${index + 1}`} className="w-full h-32 object-cover" />
                  </div>
                ))}
              </div>
            </div>
          )}
        </div>
      )}
    </div>
  );
}
