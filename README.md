// App.jsx
import React, { useEffect, useState, useRef } from 'react';
import { jsPDF } from 'jspdf';

/*
  Prototipo Helpie - App.jsx
  - UI basada en Tailwind (se asume configuración Tailwind ya instalada)
  - Idiomas: es (español), qu (quechua), en (inglés) -> traducciones minimalistas
  - LocalStorage usado para caché offline y progreso
  - Archivo soportados: imagenes y PDF (análisis simulado)
  - Certificado: generado con jsPDF (descarga cliente)
  - NOTAS: Integrar IA/Backend en los puntos marcados con TODO
*/

/* ---------------------------
   Traducciones básicas
   --------------------------- */
const I18N = {
  es: {
    appName: 'Helpie',
    slogan: 'Actúa, ayuda, salva',
    tipsTitle: 'Tips del día',
    dashboard: 'Inicio',
    chat: 'Chat',
    upload: 'Subir archivo',
    hospitals: 'Hospitales',
    profile: 'Perfil',
    simulator: 'Simulador',
    analyze: 'Analizar',
    generateCert: 'Generar certificado',
    emergency: 'EMERGENCIA',
    call: 'Llamar',
    planGratis: 'Gratis',
    planPremium: 'Premium',
    subscribe: 'Suscribirse - $10/mes',
    unsubscribe: 'Cambiar a Gratis',
    language: 'Idioma',
    welcome: 'Bienvenido/a',
    noFiles: 'No hay archivos subidos',
    analysisResult: 'Resultado del análisis',
    rcp_step1: 'Comprueba la seguridad de la escena.',
    rcp_step2: 'Comprueba si la persona responde y respira.',
    rcp_step3: 'Si no respira, inicia RCP (30 compresiones / 2 ventilaciones).'
  },
  qu: {
    appName: 'Helpie',
    slogan: 'Actuay, yanapay, kichay',
    tipsTitle: 'Kawsay rimaykuna',
    dashboard: 'Yachay',
    chat: 'Rimay',
    upload: 'Archivotan Upiy',
    hospitals: 'Sutinanpachaw',
    profile: 'Perfil',
    simulator: 'Simulador',
    analyze: 'Willay',
    generateCert: 'Certificadomanta',
    emergency: 'QARAQI',
    call: 'Taqay',
    planGratis: 'Kuskan',
    planPremium: 'Premium',
    subscribe: 'Ruwanapaq - $10/mes',
    unsubscribe: 'Kuskanpaq',
    language: 'Runa simi',
    welcome: 'Kawsayki',
    noFiles: 'Mana archivotakunataqa kanchu',
    analysisResult: 'Willaypa chaylla',
    rcp_step1: 'Riqsichiy llant'iy markana.',
    rcp_step2: 'Riqsichiy ima rimanmi karqan wakinmi aswanmi.',
    rcp_step3: 'Mana riqsichiychu, RCP hamuy (30 pacha / 2 sutinakuna).'
  },
  en: {
    appName: 'Helpie',
    slogan: 'Act, help, save',
    tipsTitle: 'Tip of the day',
    dashboard: 'Dashboard',
    chat: 'Chat',
    upload: 'Upload file',
    hospitals: 'Hospitals',
    profile: 'Profile',
    simulator: 'Simulator',
    analyze: 'Analyze',
    generateCert: 'Generate certificate',
    emergency: 'EMERGENCY',
    call: 'Call',
    planGratis: 'Free',
    planPremium: 'Premium',
    subscribe: 'Subscribe - $10/month',
    unsubscribe: 'Downgrade to Free',
    language: 'Language',
    welcome: 'Welcome',
    noFiles: 'No files uploaded',
    analysisResult: 'Analysis result',
    rcp_step1: 'Check scene safety.',
    rcp_step2: 'Check responsiveness and breathing.',
    rcp_step3: 'If not breathing, start CPR (30 compressions / 2 breaths).'
  }
};

/* ---------------------------
   Datos iniciales (ejemplos Sucre)
   --------------------------- */
const HOSPITALES_SUCRE = [
  {
    id: 'h1',
    name: 'Hospital Santa Bárbara',
    phones: ['+591-4-6111111'],
    address: 'Av. Potosí 123, Sucre',
    emergency24: true
  },
  {
    id: 'h2',
    name: 'Hospital San Juan de Dios',
    phones: ['+591-4-6222222'],
    address: 'C. Sucre 456, Sucre',
    emergency24: true
  },
  {
    id: 'h3',
    name: 'Caja Nacional de Salud - Sucre',
    phones: ['+591-4-6333333'],
    address: 'Plaza 1 Mayo, Sucre',
    emergency24: false
  }
];

/* ---------------------------
   Tips por defecto (se guardan en caché)
   --------------------------- */
const DEFAULT_TIPS = [
  'Comprueba la seguridad de la escena antes de acercarte.',
  'Si la persona no responde, verifica respiración y llama a emergencias.',
  'Para quemaduras, refrigera con agua fría por 10 minutos.'
];

/* ---------------------------
   Utilidades: storage, idioma
   --------------------------- */
const STORAGE_KEY = 'helpie_state_v1';

function loadState() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return null;
    return JSON.parse(raw);
  } catch {
    return null;
  }
}
function saveState(state) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
}

/* ---------------------------
   Componentes UI
   --------------------------- */

function Card({ children, className = '' }) {
  return (
    <div className={`bg-white rounded-2xl shadow-sm p-4 ${className}`}>
      {children}
    </div>
  );
}

/* ---------------------------
   Dashboard
   --------------------------- */
function Dashboard({ t, user, onNavigate, tips }) {
  return (
    <div>
      <div className="flex items-center justify-between mb-4">
        <div>
          <h2 className="text-2xl font-bold">{t.welcome}, {user.name}</h2>
          <p className="text-sm text-gray-600">{t.slogan}</p>
        </div>
        <div className="text-right">
          <div className="text-sm">Plan: <span className="font-semibold">{user.plan === 'free' ? t.planGratis : t.planPremium}</span></div>
        </div>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-4">
        <Card>
          <h3 className="font-semibold">{t.tipsTitle}</h3>
          <p className="mt-2">{tips.current}</p>
          <div className="mt-3 flex gap-2">
            <button onClick={() => onNavigate('chat')} className="px-3 py-1 rounded bg-blue-600 text-white text-sm">Ir al chat</button>
            <button onClick={() => onNavigate('simulator')} className="px-3 py-1 rounded border text-sm">Abrir simulador</button>
          </div>
        </Card>

        <Card>
          <h3 className="font-semibold">Progreso</h3>
          <div className="mt-3">
            {/* Progreso simulado */}
            <ProgressBar label="RCP" value={user.progress?.rcp || 10} />
            <ProgressBar label="Hemorragias" value={user.progress?.hemorragias || 0} />
            <ProgressBar label="Primeros Auxilios Psicológicos" value={user.progress?.psico || 0} />
          </div>
        </Card>

        <Card>
          <h3 className="font-semibold">Accesos</h3>
          <ul className="mt-3 space-y-2">
            <li><button className="text-left w-full" onClick={() => onNavigate('upload')}>{t.upload}</button></li>
            <li><button className="text-left w-full" onClick={() => onNavigate('hospitals')}>{t.hospitals}</button></li>
            <li><button className="text-left w-full" onClick={() => onNavigate('profile')}>{t.profile}</button></li>
          </ul>
        </Card>
      </div>

      <Card>
        <h3 className="font-semibold">Guía rápida</h3>
        <ol className="mt-2 list-decimal list-inside text-sm text-gray-700">
          <li>{t.rcp_step1}</li>
          <li>{t.rcp_step2}</li>
          <li>{t.rcp_step3}</li>
        </ol>
      </Card>
    </div>
  );
}

/* ---------------------------
   ProgressBar
   --------------------------- */
function ProgressBar({ label, value = 0 }) {
  return (
    <div className="mb-3">
      <div className="flex justify-between text-sm mb-1">
        <span>{label}</span>
        <span>{value}%</span>
      </div>
      <div className="bg-gray-200 rounded-full h-2">
        <div style={{ width: `${value}%` }} className="bg-blue-500 h-2 rounded-full"></div>
      </div>
    </div>
  );
}

/* ---------------------------
   Chat (stub conversacional)
   --------------------------- */
function Chat({ t, user, onBack }) {
  const [messages, setMessages] = useState([
    { id: 1, from: 'bot', text: `${t.welcome} ${user.name}. Soy Helpie. ¿En qué puedo ayudarte hoy?` }
  ]);
  const [input, setInput] = useState('');
  const messagesRef = useRef(null);

  useEffect(() => {
    messagesRef.current?.scrollTo({ top: messagesRef.current.scrollHeight, behavior: 'smooth' });
  }, [messages]);

  function sendMessage() {
    if (!input.trim()) return;
    const m = { id: Date.now(), from: 'user', text: input };
    setMessages(prev => [...prev, m]);
    setInput('');
    // TODO: integrar llamada a backend/IA: enviar m.text y obtener respuesta
    setTimeout(() => {
      // Respuesta simulada con detección simple de palabras clave
      let response = 'Entiendo. ¿Puedes describir los signos vitales o la situación?';
      if (/inconscien|no respira|no respire/i.test(m.text)) {
        response = `${t.emergency}! ${t.rcp_step2} ${t.rcp_step3}`;
      } else if (/sangr|hemorrag/i.test(m.text)) {
        response = 'Si la hemorragia es abundante, aplica presión directa y eleva la extremidad si es posible. Busca atención médica.';
      }
      setMessages(prev => [...prev, { id: Date.now() + 1, from: 'bot', text: response }]);
    }, 800);
  }

  return (
    <div>
      <div className="flex items-center justify-between mb-4">
        <button onClick={onBack} className="text-sm text-blue-600">← Volver</button>
        <h3 className="font-semibold">{t.chat}</h3>
        <div />
      </div>

      <div ref={messagesRef} className="h-72 overflow-auto p-3 bg-gray-50 rounded mb-3">
        {messages.map(m => (
          <div key={m.id} className={`mb-2 max-w-full ${m.from === 'bot' ? 'text-left' : 'text-right'}`}>
            <div className={`inline-block p-2 rounded ${m.from === 'bot' ? 'bg-white shadow' : 'bg-blue-500 text-white'}`}>
              {m.text}
            </div>
          </div>
        ))}
      </div>

      <div className="flex gap-2">
        <input value={input} onChange={e => setInput(e.target.value)} className="flex-1 p-2 border rounded" placeholder="Escribe aquí..." />
        <button onClick={sendMessage} className="px-4 py-2 rounded bg-blue-600 text-white">Enviar</button>
      </div>
    </div>
  );
}

/* ---------------------------
   FileUpload + análisis (simulado)
   --------------------------- */
function FileUpload({ t, files, setFiles, onBack, onAnalysisComplete }) {
  const [uploading, setUploading] = useState(false);
  const [analysisResult, setAnalysisResult] = useState(null);

  function handleFiles(e) {
    const list = Array.from(e.target.files);
    const mapped = list.map(f => ({ id: Date.now() + Math.random(), name: f.name, size: f.size, file: f }));
    setFiles(prev => [...prev, ...mapped]);
  }

  async function analyzeFile(fileEntry) {
    setUploading(true);
    // TODO: enviar archivo al backend/IA para análisis; aquí se simula
    await new Promise(r => setTimeout(r, 800));
    // Simulación: si el nombre contiene 'herida' -> herida superficial
    let result = { urgency: 'Moderada', advice: 'Lavar con agua, aplicar vendaje. Consultar si sangrado continúa.' };
    if (/grave|profunda|hematoma|fractura/i.test(fileEntry.name)) {
      result = { urgency: 'Alta', advice: 'Buscar atención médica inmediata. Evitar movilizar si sospecha fractura.' };
    }
    setAnalysisResult(result);
    setUploading(false);
    if (onAnalysisComplete) onAnalysisComplete(result);
  }

  return (
    <div>
      <div className="flex items-center justify-between mb-4">
        <button onClick={onBack} className="text-sm text-blue-600">← Volver</button>
        <h3 className="font-semibold">{t.upload}</h3>
        <div />
      </div>

      <Card>
        <input type="file" accept="image/*,.pdf" onChange={handleFiles} multiple />
        <div className="mt-3">
          {files.length === 0 ? <div className="text-sm text-gray-500">{t.noFiles}</div> : (
            <ul className="space-y-2">
              {files.map(f => (
                <li key={f.id} className="flex items-center justify-between">
                  <div>{f.name} <span className="text-xs text-gray-400">({Math.round(f.size / 1024)} KB)</span></div>
                  <div className="flex gap-2">
                    <button onClick={() => analyzeFile(f)} className="px-2 py-1 text-sm rounded bg-blue-600 text-white">{t.analyze}</button>
                    <button onClick={() => setFiles(prev => prev.filter(x => x.id !== f.id))} className="px-2 py-1 text-sm rounded border">Eliminar</button>
                  </div>
                </li>
              ))}
            </ul>
          )}
        </div>

        {uploading && <div className="mt-3 text-sm text-gray-600">Analizando...</div>}

        {analysisResult && (
          <div className="mt-3 p-3 bg-gray-50 rounded">
            <h4 className="font-semibold">{t.analysisResult}</h4>
            <div className="mt-2 text-sm">Urgencia: <strong>{analysisResult.urgency}</strong></div>
            <div className="mt-1 text-sm">Recomendación: {analysisResult.advice}</div>
          </div>
        )}
      </Card>
    </div>
  );
}

/* ---------------------------
   Hospitals list
   --------------------------- */
function Hospitals({ t, onBack }) {
  const [hospitals] = useState(HOSPITALES_SUCRE);

  return (
    <div>
      <div className="flex items-center justify-between mb-4">
        <button onClick={onBack} className="text-sm text-blue-600">← Volver</button>
        <h3 className="font-semibold">{t.hospitals}</h3>
        <div />
      </div>

      <div className="space-y-3">
        {hospitals.map(h => (
          <Card key={h.id}>
            <div className="flex justify-between">
              <div>
                <div className="font-semibold">{h.name}</div>
                <div className="text-sm text-gray-600">{h.address}</div>
                <div className="text-sm mt-1">Tel: {h.phones.join(', ')}</div>
              </div>
              <div className="flex flex-col items-end gap-2">
                <a href={`tel:${h.phones[0]}`} className="px-3 py-1 rounded bg-blue-600 text-white text-sm">{t.call}</a>
                <a target="_blank" rel="noreferrer" href={`https://www.google.com/maps/search/?api=1&query=${encodeURIComponent(h.address)}`} className="text-sm text-gray-500">Indicaciones</a>
              </div>
            </div>
          </Card>
        ))}
      </div>
    </div>
  );
}

/* ---------------------------
   Perfil y suscripción
   --------------------------- */
function Profile({ t, user, setUser, onBack }) {
  function togglePlan() {
    if (user.plan === 'free') {
      // TODO: integrar pago (Stripe/PayPal) y verificación real
      setUser(prev => ({ ...prev, plan: 'premium' }));
    } else {
      setUser(prev => ({ ...prev, plan: 'free' }));
    }
  }

  function generateCertificate() {
    // Genera un certificado PDF simple con jsPDF
    const doc = new jsPDF();
    doc.setFontSize(18);
    doc.text('Certificado Helpie', 20, 30);
    doc.setFontSize(12);
    doc.text(`Nombre: ${user.name}`, 20, 50);
    doc.text(`Email: ${user.email}`, 20, 60);
    doc.text(`Fecha: ${new Date().toLocaleDateString()}`, 20, 70);
    doc.text('Certifica que el titular completó módulos de primeros auxilios.', 20, 90);
    doc.save(`certificado_helpie_${user.name.replace(/\s+/g, '_')}.pdf`);
  }

  return (
    <div>
      <div className="flex items-center justify-between mb-4">
        <button onClick={onBack} className="text-sm text-blue-600">← Volver</button>
        <h3 className="font-semibold">{t.profile}</h3>
        <div />
      </div>

      <Card>
        <div className="mb-3">
          <label className="block text-sm text-gray-600">Nombre</label>
          <input value={user.name} onChange={e => setUser(u => ({ ...u, name: e.target.value }))} className="w-full p-2 border rounded" />
        </div>
        <div className="mb-3">
          <label className="block text-sm text-gray-600">Email</label>
          <input value={user.email} onChange={e => setUser(u => ({ ...u, email: e.target.value }))} className="w-full p-2 border rounded" />
        </div>

        <div className="flex gap-2">
          <button onClick={togglePlan} className="px-3 py-2 rounded bg-blue-600 text-white">
            {user.plan === 'free' ? t.subscribe : t.unsubscribe}
          </button>
          <button onClick={generateCertificate} className="px-3 py-2 rounded border">
            {t.generateCert}
          </button>
        </div>
      </Card>
    </div>
  );
}

/* ---------------------------
   Simulador interactivo (decisiones)
   --------------------------- */
function Simulator({ t, onBack, onComplete }) {
  const scenarios = [
    {
      id: 's1',
      title: 'Persona inconsciente, respira?',
      text: 'Encuentras a una persona inconsciente en la vía pública. ¿Respira?',
      choices: [
        { id: 'c1', text: 'Sí, respira', result: 'Mantén la vía aérea abierta y colócala en posición lateral de seguridad. Monitorea.' },
        { id: 'c2', text: 'No, no respira', result: 'Inicia RCP y pide ayuda. Llama a emergencias.' }
      ]
    },
    {
      id: 's2',
      title: 'Hemorragia abundante',
      text: 'Una herida presenta sangrado abundante. ¿Qué haces?',
      choices: [
        { id: 'c1', text: 'Aplico presión directa', result: 'Correcto. Mantén presión y eleva la extremidad si es posible.' },
        { id: 'c2', text: 'Busco vendajes y espero', result: 'Riesgoso. La presión inmediata reduce pérdida de sangre.' }
      ]
    }
  ];

  const [index, setIndex] = useState(0);
  const [feedback, setFeedback] = useState(null);

  function choose(choice) {
    setFeedback(choice.result);
    setTimeout(() => {
      setFeedback(null);
      if (index < scenarios.length - 1) setIndex(i => i + 1);
      else {
        if (onComplete) onComplete({ score: 100 });
      }
    }, 1400);
  }

  const s = scenarios[index];

  return (
    <div>
      <div className="flex items-center justify-between mb-4">
        <button onClick={onBack} className="text-sm text-blue-600">← Volver</button>
        <h3 className="font-semibold">{t.simulator}</h3>
        <div />
      </div>

      <Card>
        <div className="font-semibold mb-2">{s.title}</div>
        <div className="text-sm text-gray-700 mb-3">{s.text}</div>
        <div className="flex flex-col gap-2">
          {s.choices.map(c => (
            <button key={c.id} onClick={() => choose(c)} className="px-3 py-2 rounded border text-left">{c.text}</button>
          ))}
        </div>

        {feedback && <div className="mt-3 p-3 bg-green-50 rounded">{feedback}</div>}
      </Card>
    </div>
  );
}

/* ---------------------------
   Main App
   --------------------------- */
export default function App() {
  // Cargar estado de localStorage o inicializar
  const saved = loadState();
  const initial = saved || {
    user: { name: 'Ervin', email: 'ervin@example.com', plan: 'free', preferredLanguage: 'es', progress: { rcp: 10 }, consents: { analyzeFiles: true } },
    tips: DEFAULT_TIPS,
    tipsIndex: 0,
    files: []
  };

  const [state, setState] = useState(initial);
  const [view, setView] = useState('dashboard'); // dashboard | chat | upload | hospitals | profile | simulator
  const [lang, setLang] = useState(state.user.preferredLanguage || 'es');

  // Guardar estado automáticamente
  useEffect(() => {
    saveState(state);
  }, [state]);

  // Rotación de tips diaria (demo: 6s). En producción cambiar a 24h.
  useEffect(() => {
    const id = setInterval(() => {
      setState(s => ({ ...s, tipsIndex: (s.tipsIndex + 1) % s.tips.length }));
    }, 6000);
    return () => clearInterval(id);
  }, []);

  // Funciones para pasar a componentes
  function setUser(upd) {
    setState(s => ({ ...s, user: { ...s.user, ...upd } }));
  }
  function setFiles(files) {
    setState(s => ({ ...s, files }));
  }

  const t = I18N[lang] || I18N.es;

  return (
    <div className="min-h-screen bg-gray-50 font-sans">
      <header className="bg-gradient-to-r from-blue-400 to-blue-600 text-white p-4 flex justify-between items-center">
        <div>
          <h1 className="text-xl font-bold">{t.appName}</h1>
          <p className="text-sm">{t.slogan}</p>
        </div>
        <div className="flex items-center gap-4">
          <select value={lang} onChange={e => { setLang(e.target.value); setUser({ preferr
