# Hono + React Router + Vite + ShadCN UI on Cloudflare Workers

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/cloudflare/templates/tree/main/react-router-hono-fullstack-template)
![Build modern full-stack apps with Hono, React Router, and ShadCN UI on Cloudflare Workers](https://imagedelivery.net/wSMYJvS3Xw-n339CbDyDIA/24c5a7dd-e1e3-43a9-b912-d78d9a4293bc/public)

<!-- dash-content-start -->

A modern full-stack template powered by [Cloudflare Workers](https://workers.cloudflare.com/), using [Hono](https://hono.dev/) for backend APIs, [React Router](https://reactrouter.com/) for frontend routing, and [shadcn/ui](https://ui.shadcn.com/) for beautiful, accessible components styled with [Tailwind CSS](https://tailwindcss.com/).

Built with the [Cloudflare Vite plugin](https://developers.cloudflare.com/workers/vite-plugin/) for optimized static asset delivery and seamless local development. React is configured in single-page app (SPA) mode via Workers.

A perfect starting point for building interactive, styled, and edge-deployed SPAs with minimal configuration.

## Features

- ⚡ Full-stack app on Cloudflare Workers
- 🔁 Hono for backend API endpoints
- 🧭 React Router for client-side routing
- 🎨 ShadCN UI with Tailwind CSS for components and styling
- 🧱 File-based route separation
- 🚀 Zero-config Vite build for Workers
- 🛠️ Automatically deploys with Wrangler
- 🔎 Built-in Observability to monitor your Worker
<!-- dash-content-end -->

## Tech Stack

- **Frontend**: React + React Router + ShadCN UI
  - SPA architecture powered by React Router
  - Includes accessible, themeable UI from ShadCN
  - Styled with utility-first Tailwind CSS
  - Built and optimized with Vite

- **Backend**: Hono on Cloudflare Workers
  - API routes defined and handled via Hono in `/api/*`
  - Supports REST-like endpoints, CORS, and middleware

- **Deployment**: Cloudflare Workers via Wrangler
  - Vite plugin auto-bundles frontend and backend together
  - Deployed worldwide on Cloudflare’s edge network

## Resources

- 🧩 [Hono on Cloudflare Workers](https://hono.dev/docs/getting-started/cloudflare-workers)
- 📦 [Vite Plugin for Cloudflare](https://developers.cloudflare.com/workers/vite-plugin/)
- 🛠 [Wrangler CLI reference](https://developers.cloudflare.com/workers/wrangler/)
- 🎨 [shadcn/ui](https://ui.shadcn.com)
- 💨 [Tailwind CSS Documentation](https://tailwindcss.com/)
- 🔀 [React Router Docs](https://reactrouter.com/)
import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInWithCustomToken, 
  signInAnonymously, 
  onAuthStateChanged,
  signOut 
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  addDoc, 
  onSnapshot, 
  query, 
  deleteDoc, 
  doc, 
  serverTimestamp 
} from 'firebase/firestore';
import { 
  Upload, 
  LogOut, 
  Image as ImageIcon, 
  Trash2, 
  Lock, 
  User, 
  Loader2,
  Plus
} from 'lucide-react';

// --- Firebase Configuration ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'photo-admin-panel';

export default function App() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [loginData, setLoginData] = useState({ id: '', password: '' });
  const [photos, setPhotos] = useState([]);
  const [uploading, setUploading] = useState(false);
  const fileInputRef = useRef(null);

  // Constants for your specific credentials
  const ADMIN_ID = 'admin';
  const ADMIN_PASS = 'Ashif@1123';

  // 1. Initialize Auth
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          // We start with a signed out state for a login-based admin panel
          setLoading(false);
        }
      } catch (err) {
        console.error("Auth error:", err);
        setLoading(false);
      }
    };
    initAuth();

    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser);
      setLoading(false);
    });
    return () => unsubscribe();
  }, []);

  // 2. Sync Photos from Firestore
  useEffect(() => {
    if (!user) {
      setPhotos([]);
      return;
    }

    const photosRef = collection(db, 'artifacts', appId, 'public', 'data', 'photos');
    const unsubscribe = onSnapshot(photosRef, 
      (snapshot) => {
        const photoData = snapshot.docs.map(doc => ({
          id: doc.id,
          ...doc.data()
        })).sort((a, b) => (b.createdAt?.seconds || 0) - (a.createdAt?.seconds || 0));
        setPhotos(photoData);
      },
      (err) => {
        console.error("Firestore error:", err);
      }
    );

    return () => unsubscribe();
  }, [user]);

  // Handle Login
  const handleLogin = async (e) => {
    e.preventDefault();
    setError(null);
    
    if (loginData.id === ADMIN_ID && loginData.password === ADMIN_PASS) {
      try {
        setLoading(true);
        // Using anonymous sign-in to represent the session once validated
        await signInAnonymously(auth);
      } catch (err) {
        setError("Failed to establish session.");
        setLoading(false);
      }
    } else {
      setError("Invalid ID or Password.");
    }
  };

  // Handle Logout
  const handleLogout = async () => {
    await signOut(auth);
  };

  // Handle Image Upload
  const handleFileUpload = async (e) => {
    const file = e.target.files[0];
    if (!file) return;

    // Check file size (limit to ~1MB for document storage base64 if no storage bucket is used, 
    // but here we'll assume standard base64 for preview/demo or simulated storage)
    if (file.size > 2 * 1024 * 1024) {
      setError("File is too large. Please keep under 2MB.");
      return;
    }

    setUploading(true);
    const reader = new FileReader();
    reader.onloadend = async () => {
      const base64Data = reader.result;
      
      try {
        await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'photos'), {
          url: base64Data,
          name: file.name,
          createdAt: serverTimestamp(),
          owner: user.uid
        });
        setUploading(false);
      } catch (err) {
        setError("Upload failed.");
        setUploading(false);
      }
    };
    reader.readAsDataURL(file);
  };

  const deletePhoto = async (id) => {
    try {
      await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'photos', id));
    } catch (err) {
      setError("Delete failed.");
    }
  };

  if (loading) {
    return (
      <div className="min-h-screen bg-slate-50 flex items-center justify-center">
        <Loader2 className="w-8 h-8 animate-spin text-blue-600" />
      </div>
    );
  }

  // --- LOGIN VIEW ---
  if (!user) {
    return (
      <div className="min-h-screen bg-slate-100 flex items-center justify-center p-4">
        <div className="max-w-md w-full bg-white rounded-2xl shadow-xl p-8">
          <div className="flex flex-col items-center mb-8">
            <div className="bg-blue-600 p-4 rounded-full mb-4">
              <Lock className="text-white w-8 h-8" />
            </div>
            <h1 className="text-2xl font-bold text-slate-800">Admin Photo Portal</h1>
            <p className="text-slate-500">Sign in to manage your storage</p>
          </div>

          <form onSubmit={handleLogin} className="space-y-6">
            <div>
              <label className="block text-sm font-medium text-slate-700 mb-2 flex items-center gap-2">
                <User size={16} /> User ID
              </label>
              <input
                type="text"
                required
                className="w-full px-4 py-3 rounded-lg border border-slate-200 focus:ring-2 focus:ring-blue-500 focus:outline-none transition-all"
                value={loginData.id}
                onChange={(e) => setLoginData({ ...loginData, id: e.target.value })}
                placeholder="Enter admin ID"
              />
            </div>
            <div>
              <label className="block text-sm font-medium text-slate-700 mb-2 flex items-center gap-2">
                <Lock size={16} /> Password
              </label>
              <input
                type="password"
                required
                className="w-full px-4 py-3 rounded-lg border border-slate-200 focus:ring-2 focus:ring-blue-500 focus:outline-none transition-all"
                value={loginData.password}
                onChange={(e) => setLoginData({ ...loginData, password: e.target.value })}
                placeholder="••••••••"
              />
            </div>

            {error && (
              <div className="p-3 bg-red-50 border border-red-100 text-red-600 rounded-lg text-sm">
                {error}
              </div>
            )}

            <button
              type="submit"
              className="w-full bg-blue-600 hover:bg-blue-700 text-white font-semibold py-3 rounded-lg transition-colors shadow-lg shadow-blue-200"
            >
              Access Admin Panel
            </button>
          </form>
        </div>
      </div>
    );
  }

  // --- DASHBOARD VIEW ---
  return (
    <div className="min-h-screen bg-slate-50">
      {/* Header */}
      <header className="bg-white border-b border-slate-200 sticky top-0 z-10">
        <div className="max-w-7xl mx-auto px-4 h-16 flex items-center justify-between">
          <div className="flex items-center gap-2">
            <ImageIcon className="text-blue-600" />
            <h1 className="font-bold text-slate-800 text-lg hidden sm:block">Photo Manager Pro</h1>
            <h1 className="font-bold text-slate-800 text-lg sm:hidden">PMP</h1>
          </div>
          
          <div className="flex items-center gap-4">
            <span className="text-sm font-medium text-slate-600 bg-slate-100 px-3 py-1 rounded-full flex items-center gap-2">
              <div className="w-2 h-2 bg-green-500 rounded-full animate-pulse"></div>
              {ADMIN_ID}
            </span>
            <button 
              onClick={handleLogout}
              className="p-2 text-slate-400 hover:text-red-600 hover:bg-red-50 rounded-full transition-all"
              title="Logout"
            >
              <LogOut size={20} />
            </button>
          </div>
        </div>
      </header>

      <main className="max-w-7xl mx-auto px-4 py-8">
        {/* Upload Action Area */}
        <div className="mb-10 bg-white p-8 rounded-2xl border border-slate-200 shadow-sm text-center">
          <div className="max-w-sm mx-auto">
            <div className="mb-4 flex justify-center">
              <div className="bg-blue-50 p-6 rounded-full border-2 border-dashed border-blue-200">
                <Upload className="w-12 h-12 text-blue-500" />
              </div>
            </div>
            <h2 className="text-xl font-bold text-slate-800 mb-2">Upload New Photos</h2>
            <p className="text-slate-500 mb-6">Max file size 2MB. Supports JPG, PNG, WEBP.</p>
            
            <input 
              type="file" 
              accept="image/*" 
              className="hidden" 
              ref={fileInputRef}
              onChange={handleFileUpload}
            />
            
            <button
              disabled={uploading}
              onClick={() => fileInputRef.current?.click()}
              className="w-full flex items-center justify-center gap-2 bg-blue-600 hover:bg-blue-700 disabled:bg-blue-400 text-white font-bold py-3 px-6 rounded-xl transition-all"
            >
              {uploading ? (
                <>
                  <Loader2 className="animate-spin" size={20} />
                  Uploading...
                </>
              ) : (
                <>
                  <Plus size={20} />
                  Select Image
                </>
              )}
            </button>
          </div>
        </div>

        {/* Gallery Grid */}
        <div className="space-y-6">
          <div className="flex items-center justify-between">
            <h3 className="text-lg font-bold text-slate-800 flex items-center gap-2">
              Your Library <span className="text-sm font-normal text-slate-400">({photos.length})</span>
            </h3>
          </div>

          {photos.length === 0 && !uploading ? (
            <div className="py-20 text-center bg-white rounded-2xl border-2 border-dashed border-slate-200">
              <ImageIcon className="mx-auto w-12 h-12 text-slate-200 mb-4" />
              <p className="text-slate-400">No photos uploaded yet. Start by adding one above.</p>
            </div>
          ) : (
            <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-6">
              {photos.map((photo) => (
                <div key={photo.id} className="group relative bg-white rounded-xl overflow-hidden shadow-sm border border-slate-200 hover:shadow-md transition-shadow">
                  <div className="aspect-square w-full overflow-hidden bg-slate-100">
                    <img 
                      src={photo.url} 
                      alt={photo.name} 
                      className="w-full h-full object-cover transition-transform duration-500 group-hover:scale-105"
                    />
                  </div>
                  <div className="p-3 flex items-center justify-between bg-white border-t border-slate-100">
                    <div className="truncate pr-2">
                      <p className="text-sm font-medium text-slate-700 truncate">{photo.name}</p>
                      <p className="text-[10px] text-slate-400">
                        {photo.createdAt ? new Date(photo.createdAt.seconds * 1000).toLocaleDateString() : 'Processing...'}
                      </p>
                    </div>
                    <button 
                      onClick={() => deletePhoto(photo.id)}
                      className="p-2 text-slate-300 hover:text-red-500 hover:bg-red-50 rounded-lg transition-colors"
                    >
                      <Trash2 size={16} />
                    </button>
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>
      </main>

      {/* Footer info */}
      <footer className="py-10 text-center text-slate-400 text-sm">
        &copy; 2024 Admin Photo Management System
      </footer>
    </div>
  );
}

