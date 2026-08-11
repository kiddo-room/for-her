import React, { useState, useEffect, useRef, useCallback } from "react";
import { Heart, Calendar, Clock, MapPin, Sparkles, Lock, Eye, Trash2, Download, X, Volume2, VolumeX, ChevronLeft } from "lucide-react";

/* ---------------------------------------------------------
   storage helpers — shared:true so both her submission and
   your admin view read/write the same records
--------------------------------------------------------- */
const PREFIX = "date-request:";
const ADMIN_PASSCODE = "ourdate"; // change this before sharing the link

async function saveSubmission(sub) {
  try {
    await window.storage.set(PREFIX + sub.id, JSON.stringify(sub), true);
    return true;
  } catch (e) {
    console.error("save failed", e);
    return false;
  }
}
async function loadSubmissions() {
  try {
    const list = await window.storage.list(PREFIX, true);
    if (!list || !list.keys) return [];
    const out = [];
    for (const k of list.keys) {
      try {
        const r = await window.storage.get(k, true);
        if (r) out.push(JSON.parse(r.value));
      } catch (e) {}
    }
    return out.sort((a, b) => b.timestamp - a.timestamp);
  } catch (e) {
    return [];
  }
}
async function deleteSubmissionKey(id) {
  try {
    await window.storage.delete(PREFIX + id, true);
    return true;
  } catch (e) {
    return false;
  }
}

const PLACES = [
  { key: "dinner", label: "Dinner", icon: "🍽️" },
  { key: "cafe", label: "Café", icon: "☕" },
  { key: "movie", label: "Movie", icon: "🎬" },
  { key: "walk", label: "Park / Walk", icon: "🌳" },
  { key: "shopping", label: "Shopping", icon: "🛍️" },
  { key: "adventure", label: "Adventure", icon: "🎡" },
  { key: "surprise", label: "Your choice", icon: "💕" },
];
const VIBES = [
  { key: "romantic", label: "Romantic", icon: "❤️" },
  { key: "relaxing", label: "Relaxing", icon: "🌸" },
  { key: "fun", label: "Fun", icon: "😂" },
  { key: "adventure", label: "Adventure", icon: "🗺️" },
  { key: "foodie", label: "Foodie", icon: "🍜" },
  { key: "movie", label: "Movie night", icon: "🎬" },
  { key: "surprise", label: "Surprise me", icon: "🎁" },
];
const THINK_MESSAGES = [
  "Are you sure? 🥺",
  "I'll even let you choose everything ❤️",
  "Okay okay… but think carefully 😭❤️",
  "I'll wait as long as you need. 🌙",
  "…still here. Still hoping. 🥹",
];

function useTypewriter(text, speed = 32, start = true) {
  const [out, setOut] = useState("");
  useEffect(() => {
    if (!start) return;
    setOut("");
    let i = 0;
    const id = setInterval(() => {
      i++;
      setOut(text.slice(0, i));
      if (i >= text.length) clearInterval(id);
    }, speed);
    return () => clearInterval(id);
  }, [text, speed, start]);
  return out;
}

function uid() {
  return Date.now().toString(36) + Math.random().toString(36).slice(2, 8);
}

function FloatingHearts({ count = 10 }) {
  const hearts = useRef(
    Array.from({ length: count }).map((_, i) => ({
      id: i,
      left: Math.random() * 100,
      delay: Math.random() * 10,
      duration: 14 + Math.random() * 10,
      size: 10 + Math.random() * 14,
      glyph: Math.random() > 0.5 ? "❤" : "♡",
    }))
  ).current;
  return (
    <div className="dwy-hearts-layer" aria-hidden="true">
      {hearts.map((h) => (
        <span
          key={h.id}
          className="dwy-float-heart"
          style={{
            left: h.left + "%",
            animationDelay: h.delay + "s",
            animationDuration: h.duration + "s",
            fontSize: h.size + "px",
          }}
        >
          {h.glyph}
        </span>
      ))}
    </div>
  );
}

function Burst() {
  const pieces = useRef(
    Array.from({ length: 26 }).map((_, i) => ({
      id: i,
      left: Math.random() * 100,
      delay: Math.random() * 0.4,
      duration: 1.6 + Math.random() * 1.2,
      size: 12 + Math.random() * 16,
      glyph: Math.random() > 0.4 ? "❤" : "✦",
      hue: Math.random() > 0.5 ? "var(--rose)" : "var(--gold)",
    }))
  ).current;
  return (
    <div className="dwy-burst-layer" aria-hidden="true">
      {pieces.map((p) => (
        <span
          key={p.id}
          className="dwy-burst-piece"
          style={{
            left: p.left + "%",
            animationDelay: p.delay + "s",
            animationDuration: p.duration + "s",
            fontSize: p.size + "px",
            color: p.hue,
          }}
        >
          {p.glyph}
        </span>
      ))}
    </div>
  );
}

function Thread({ stage }) {
  const stepOf = { landing: 0, question: 0, message: 1, planner: 1, summary: 2, confirmed: 2 };
  const step = stepOf[stage] ?? 0;
  const labels = ["Question ❤️", "Your Choice 💌", "Our Date 🥰"];
  return (
    <div className="dwy-thread" role="progressbar" aria-valuenow={step + 1} aria-valuemin={1} aria-valuemax={3}>
      {labels.map((l, i) => (
        <React.Fragment key={l}>
          <div className={"dwy-thread-node" + (i <= step ? " active" : "")}>
            <span className="dwy-thread-dot" />
            <span className="dwy-thread-label">{l}</span>
          </div>
          {i < labels.length - 1 && <span className={"dwy-thread-line" + (i < step ? " active" : "")} />}
        </React.Fragment>
      ))}
    </div>
  );
}

export default function App() {
  const [stage, setStage] = useState("landing");
  const [thinkIdx, setThinkIdx] = useState(-1);
  const [muted, setMuted] = useState(true);
  const audioRef = useRef(null);

  const [date, setDate] = useState("");
  const [time, setTime] = useState("");
  const [place, setPlace] = useState("");
  const [customPlace, setCustomPlace] = useState("");
  const [vibes, setVibes] = useState([]);
  const [specialRequest, setSpecialRequest] = useState("");
  const [finalMessage, setFinalMessage] = useState("");
  const [submissionId, setSubmissionId] = useState(null);
  const [saving, setSaving] = useState(false);

  const [adminOpen, setAdminOpen] = useState(false);
  const [adminAuthed, setAdminAuthed] = useState(false);
  const [passInput, setPassInput] = useState("");
  const [passError, setPassError] = useState(false);
  const [submissions, setSubmissions] = useState([]);
  const [loadingSubs, setLoadingSubs] = useState(false);
  const [viewing, setViewing] = useState(null);

  const line1 = useTypewriter(
    "I don't need a perfect place or a perfect plan.",
    28,
    stage === "message"
  );
  const line2 = useTypewriter(
    "I just want to spend some time with you.",
    28,
    line1.length >= "I don't need a perfect place or a perfect plan.".length
  );
  const line3 = useTypewriter(
    "So this time, you get to choose how our little adventure goes. ❤️",
    22,
    line2.length >= "I just want to spend some time with you.".length
  );

  useEffect(() => {
    if (audioRef.current) {
      audioRef.current.muted = muted;
      if (!muted) audioRef.current.play().catch(() => {});
    }
  }, [muted]);

  const toggleVibe = (k) => {
    setVibes((v) => (v.includes(k) ? v.filter((x) => x !== k) : [...v, k]));
  };

  const handleThink = () => {
    setThinkIdx((i) => (i + 1) % THINK_MESSAGES.length);
  };

  const placeLabel =
    place === "surprise" && customPlace.trim() ? customPlace.trim() : PLACES.find((p) => p.key === place)?.label || "";

  const submit = async () => {
    setSaving(true);
    const id = uid();
    const sub = {
      id,
      date,
      time,
      place: PLACES.find((p) => p.key === place)?.label || "",
      customPlace: place === "surprise" ? customPlace.trim() : "",
      vibes: vibes.map((k) => VIBES.find((v) => v.key === k)?.label),
      specialRequest,
      finalMessage,
      timestamp: Date.now(),
      seen: false,
    };
    await saveSubmission(sub);
    setSubmissionId(id);
    setSaving(false);
    setStage("summary");
  };

  const confirm = () => setStage("confirmed");

  /* ---------- admin ---------- */
  const openAdmin = () => setAdminOpen(true);
  const closeAdmin = () => {
    setAdminOpen(false);
    setViewing(null);
  };
  const tryLogin = () => {
    if (passInput === ADMIN_PASSCODE) {
      setAdminAuthed(true);
      setPassError(false);
      refreshSubs();
    } else {
      setPassError(true);
    }
  };
  const refreshSubs = async () => {
    setLoadingSubs(true);
    const subs = await loadSubmissions();
    setSubmissions(subs);
    setLoadingSubs(false);
  };
  const markSeen = async (s) => {
    const updated = { ...s, seen: true };
    await saveSubmission(updated);
    refreshSubs();
  };
  const removeSub = async (id) => {
    await deleteSubmissionKey(id);
    setViewing(null);
    refreshSubs();
  };
  const exportJson = () => {
    const blob = new Blob([JSON.stringify(submissions, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "date-requests.json";
    a.click();
    URL.revokeObjectURL(url);
  };

  const dateNice = date
    ? new Date(date + "T00:00:00").toLocaleDateString(undefined, { weekday: "long", month: "long", day: "numeric" })
    : "";
  const timeNice = time
    ? new Date("2000-01-01T" + time).toLocaleTimeString(undefined, { hour: "numeric", minute: "2-digit" })
    : "";

  return (
    <div className="dwy-root">
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,500;0,600;1,500;1,600&family=Quicksand:wght@400;500;600;700&display=swap');

        .dwy-root {
          --cream: #fdf8f3;
          --blush: #f8dde1;
          --rose: #e08a9c;
          --deep-rose: #c2415b;
          --wine: #7a2e3d;
          --ink: #4a3436;
          --gold: #c9a25c;
          --white: #ffffff;
          min-height: 100vh;
          width: 100%;
          background: linear-gradient(180deg, var(--cream) 0%, #fdf1ee 100%);
          font-family: 'Quicksand', sans-serif;
          color: var(--ink);
          position: relative;
          overflow-x: hidden;
          cursor: default;
        }
        @media (prefers-reduced-motion: reduce) {
          .dwy-root *, .dwy-root *::before, .dwy-root *::after {
            animation-duration: 0.001ms !important;
            animation-iteration-count: 1 !important;
            transition-duration: 0.001ms !important;
          }
        }
        .dwy-display {
          font-family: 'Cormorant Garamond', serif;
        }
        .dwy-hearts-layer, .dwy-burst-layer {
          position: fixed; inset: 0; pointer-events: none; overflow: hidden; z-index: 1;
        }
        .dwy-float-heart {
          position: absolute; bottom: -30px; color: var(--rose); opacity: 0.35;
          animation-name: dwyFloat; animation-timing-function: linear; animation-iteration-count: infinite;
        }
        @keyframes dwyFloat {
          0% { transform: translateY(0) translateX(0) rotate(0deg); opacity: 0; }
          10% { opacity: 0.4; }
          90% { opacity: 0.3; }
          100% { transform: translateY(-110vh) translateX(20px) rotate(25deg); opacity: 0; }
        }
        .dwy-burst-layer { z-index: 50; }
        .dwy-burst-piece {
          position: absolute; top: 40%; animation-name: dwyBurst; animation-timing-function: ease-out; animation-fill-mode: forwards;
        }
        @keyframes dwyBurst {
          0% { transform: translateY(0) scale(0.6) rotate(0deg); opacity: 1; }
          100% { transform: translateY(-70vh) scale(1.1) rotate(180deg); opacity: 0; }
        }

        .dwy-stage {
          position: relative; z-index: 2;
          min-height: 100vh;
          display: flex; flex-direction: column; align-items: center; justify-content: center;
          padding: 32px 20px 64px;
          max-width: 560px; margin: 0 auto;
          text-align: center;
        }
        .dwy-eyebrow {
          font-size: 12px; letter-spacing: 0.14em; text-transform: uppercase; color: var(--rose); font-weight: 600;
          margin-bottom: 10px;
        }
        .dwy-h1 {
          font-size: clamp(34px, 9vw, 52px);
          font-weight: 600; color: var(--wine); line-height: 1.15; margin: 0 0 14px;
        }
        .dwy-h2 {
          font-size: clamp(24px, 6vw, 32px);
          font-weight: 600; font-style: italic; color: var(--wine); line-height: 1.3; margin: 0 0 22px;
        }
        .dwy-p { font-size: 16px; color: var(--ink); opacity: 0.85; margin: 0 0 26px; line-height: 1.6; }

        .dwy-btn {
          font-family: 'Quicksand', sans-serif;
          font-weight: 700;
          font-size: 15px;
          border: none;
          border-radius: 999px;
          padding: 15px 30px;
          cursor: pointer;
          transition: transform .15s ease, box-shadow .15s ease, background .15s ease;
        }
        .dwy-btn-primary {
          background: linear-gradient(135deg, var(--deep-rose), var(--wine));
          color: var(--white);
          box-shadow: 0 14px 30px -12px rgba(194,65,91,0.55);
        }
        .dwy-btn-primary:hover { transform: translateY(-2px); box-shadow: 0 18px 34px -12px rgba(194,65,91,0.65); }
        .dwy-btn-primary:active { transform: translateY(0) scale(.98); }
        .dwy-btn-ghost {
          background: var(--white);
          color: var(--wine);
          border: 1.5px solid var(--blush);
        }
        .dwy-btn-ghost:hover { border-color: var(--rose); transform: translateY(-1px); }
        .dwy-btn:focus-visible { outline: 3px solid var(--gold); outline-offset: 2px; }
        .dwy-btn:disabled { opacity: 0.5; cursor: not-allowed; transform: none; }

        .dwy-card {
          background: var(--white);
          border-radius: 26px;
          padding: 30px 26px;
          box-shadow: 0 30px 60px -32px rgba(122,46,61,0.35);
          width: 100%;
        }

        .dwy-thread {
          display: flex; align-items: center; justify-content: center;
          gap: 6px; margin-bottom: 34px; flex-wrap: wrap;
        }
        .dwy-thread-node { display: flex; align-items: center; gap: 6px; opacity: 0.4; transition: opacity .3s ease; }
        .dwy-thread-node.active { opacity: 1; }
        .dwy-thread-dot { width: 8px; height: 8px; border-radius: 50%; background: var(--rose); display: inline-block; }
        .dwy-thread-node.active .dwy-thread-dot { background: var(--deep-rose); box-shadow: 0 0 0 4px rgba(194,65,91,0.15); }
        .dwy-thread-label { font-size: 11px; font-weight: 600; color: var(--wine); white-space: nowrap; }
        .dwy-thread-line { width: 26px; height: 1.5px; background: repeating-linear-gradient(90deg, var(--blush) 0 5px, transparent 5px 9px); }
        .dwy-thread-line.active { background: repeating-linear-gradient(90deg, var(--rose) 0 5px, transparent 5px 9px); }

        .dwy-field { margin-bottom: 24px; text-align: left; }
        .dwy-field-label { display: block; font-weight: 700; font-size: 14.5px; color: var(--wine); margin-bottom: 10px; }
        .dwy-input, .dwy-textarea {
          width: 100%; font-family: 'Quicksand', sans-serif; font-size: 15px; color: var(--ink);
          background: var(--cream); border: 1.5px solid var(--blush); border-radius: 14px;
          padding: 13px 16px; outline: none; transition: border-color .15s ease;
          accent-color: var(--deep-rose);
        }
        .dwy-input:focus, .dwy-textarea:focus { border-color: var(--rose); }
        .dwy-textarea { resize: vertical; min-height: 88px; }

        .dwy-chip-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 10px; }
        .dwy-chip {
          display: flex; align-items: center; gap: 8px;
          border: 1.5px solid var(--blush); background: var(--cream);
          border-radius: 16px; padding: 12px 14px; cursor: pointer;
          font-size: 14px; font-weight: 600; color: var(--ink);
          transition: border-color .15s ease, background .15s ease, transform .1s ease;
        }
        .dwy-chip:hover { border-color: var(--rose); }
        .dwy-chip.active { background: var(--blush); border-color: var(--deep-rose); color: var(--wine); }
        .dwy-chip:active { transform: scale(0.97); }

        .dwy-summary-row { display: flex; justify-content: space-between; gap: 14px; padding: 12px 0; border-bottom: 1px dashed var(--blush); text-align: left; }
        .dwy-summary-row:last-child { border-bottom: none; }
        .dwy-summary-k { font-size: 12px; font-weight: 700; color: var(--rose); text-transform: uppercase; letter-spacing: .06em; white-space: nowrap; }
        .dwy-summary-v { font-size: 15px; color: var(--ink); text-align: right; }

        .dwy-sound-btn {
          position: fixed; top: 18px; right: 18px; z-index: 10;
          width: 42px; height: 42px; border-radius: 50%; border: 1.5px solid var(--blush);
          background: rgba(255,255,255,0.85); backdrop-filter: blur(6px);
          display: flex; align-items: center; justify-content: center; cursor: pointer; color: var(--wine);
        }
        .dwy-sound-btn:focus-visible { outline: 3px solid var(--gold); outline-offset: 2px; }

        .dwy-admin-trigger {
          position: fixed; bottom: 12px; right: 14px; z-index: 10;
          width: 26px; height: 26px; border-radius: 50%; border: none; background: transparent;
          color: var(--blush); font-size: 13px; cursor: pointer; opacity: 0.5;
        }
        .dwy-admin-trigger:hover { opacity: 0.9; }

        .dwy-modal-overlay {
          position: fixed; inset: 0; background: rgba(74,52,54,0.45); backdrop-filter: blur(3px);
          z-index: 100; display: flex; align-items: flex-start; justify-content: center;
          padding: 40px 16px; overflow-y: auto;
        }
        .dwy-modal {
          background: var(--white); border-radius: 20px; max-width: 640px; width: 100%;
          padding: 26px; position: relative;
        }
        .dwy-modal-close {
          position: absolute; top: 16px; right: 16px; background: none; border: none; cursor: pointer; color: var(--ink); opacity: 0.5;
        }
        .dwy-modal-close:hover { opacity: 1; }

        .dwy-sub-item {
          display: flex; justify-content: space-between; align-items: center; gap: 10px;
          padding: 14px; border: 1px solid var(--blush); border-radius: 14px; margin-bottom: 10px; text-align: left;
          background: var(--cream);
        }
        .dwy-sub-item.unseen { border-color: var(--deep-rose); background: var(--blush); }
        .dwy-sub-main { cursor: pointer; flex: 1; }
        .dwy-sub-title { font-weight: 700; color: var(--wine); font-size: 14px; }
        .dwy-sub-sub { font-size: 12px; color: var(--ink); opacity: 0.7; margin-top: 2px; }
        .dwy-icon-btn { background: none; border: none; cursor: pointer; color: var(--wine); opacity: 0.6; padding: 6px; }
        .dwy-icon-btn:hover { opacity: 1; }

        .dwy-typing-caret { display: inline-block; width: 2px; height: 1em; background: var(--rose); vertical-align: -2px; animation: dwyBlink 1s steps(1) infinite; }
        @keyframes dwyBlink { 50% { opacity: 0; } }

        @media (min-width: 600px) {
          .dwy-chip-grid { grid-template-columns: repeat(3, 1fr); }
        }
      `}</style>

      <FloatingHearts />
      {stage === "confirmed" && <Burst />}

      <button
        className="dwy-sound-btn"
        onClick={() => setMuted((m) => !m)}
        aria-label={muted ? "Unmute background music" : "Mute background music"}
      >
        {muted ? <VolumeX size={18} /> : <Volume2 size={18} />}
      </button>
      {/* Replace src below with your own music file URL to enable sound */}
      <audio ref={audioRef} loop src="" />

      <button className="dwy-admin-trigger" onClick={openAdmin} aria-label="Admin">♥</button>

      {stage === "landing" && (
        <div className="dwy-stage">
          <div className="dwy-eyebrow">a very small website, for a very specific person</div>
          <h1 className="dwy-h1 dwy-display">Hey You ❤️</h1>
          <p className="dwy-p">I have a very important question to ask you…</p>
          <button className="dwy-btn dwy-btn-primary" onClick={() => setStage("question")}>
            Open My Question 💌
          </button>
        </div>
      )}

      {stage === "question" && (
        <div className="dwy-stage">
          <h2 className="dwy-h2 dwy-display" style={{ transform: `scale(${1 + Math.min(thinkIdx + 1, 4) * 0.02})`, transition: "transform .25s ease" }}>
            Will you go on a date with me? 🥺❤️
          </h2>
          <div style={{ display: "flex", gap: 12, flexWrap: "wrap", justifyContent: "center" }}>
            <button className="dwy-btn dwy-btn-primary" onClick={() => setStage("message")}>
              YES, OF COURSE ❤️
            </button>
            <button className="dwy-btn dwy-btn-ghost" onClick={handleThink}>
              Let me think… 👀
            </button>
          </div>
          {thinkIdx >= 0 && (
            <p className="dwy-p" style={{ marginTop: 22, fontStyle: "italic" }}>
              {THINK_MESSAGES[thinkIdx]}
            </p>
          )}
        </div>
      )}

      {stage === "message" && (
        <div className="dwy-stage">
          <Thread stage={stage} />
          <div style={{ minHeight: 140, textAlign: "left" }}>
            <p className="dwy-p dwy-display" style={{ fontSize: 20, fontStyle: "italic", color: "var(--wine)" }}>
              {line1}
              {line1.length < "I don't need a perfect place or a perfect plan.".length && <span className="dwy-typing-caret" />}
            </p>
            <p className="dwy-p dwy-display" style={{ fontSize: 20, fontStyle: "italic", color: "var(--wine)" }}>
              {line2}
            </p>
            <p className="dwy-p dwy-display" style={{ fontSize: 20, fontStyle: "italic", color: "var(--wine)" }}>
              {line3}
            </p>
          </div>
          <button
            className="dwy-btn dwy-btn-primary"
            onClick={() => setStage("planner")}
            disabled={line3.length < "So this time, you get to choose how our little adventure goes. ❤️".length}
          >
            Choose Our Date →
          </button>
        </div>
      )}

      {stage === "planner" && (
        <div className="dwy-stage">
          <Thread stage={stage} />
          <div className="dwy-card">
            <h2 className="dwy-h2 dwy-display" style={{ fontSize: 26 }}>Our Date, Your Choice ❤️</h2>

            <div className="dwy-field">
              <label className="dwy-field-label"><Calendar size={15} style={{ verticalAlign: -2, marginRight: 6 }} />When should I steal you away? 🥰</label>
              <input type="date" className="dwy-input" value={date} onChange={(e) => setDate(e.target.value)} />
            </div>

            <div className="dwy-field">
              <label className="dwy-field-label"><Clock size={15} style={{ verticalAlign: -2, marginRight: 6 }} />What time should we meet? ⏰❤️</label>
              <input type="time" className="dwy-input" value={time} onChange={(e) => setTime(e.target.value)} />
            </div>

            <div className="dwy-field">
              <label className="dwy-field-label"><MapPin size={15} style={{ verticalAlign: -2, marginRight: 6 }} />Where to?</label>
              <div className="dwy-chip-grid">
                {PLACES.map((p) => (
                  <button
                    type="button"
                    key={p.key}
                    className={"dwy-chip" + (place === p.key ? " active" : "")}
                    onClick={() => setPlace(p.key)}
                  >
                    <span>{p.icon}</span> {p.label}
                  </button>
                ))}
              </div>
              {place === "surprise" && (
                <input
                  type="text"
                  className="dwy-input"
                  style={{ marginTop: 10 }}
                  placeholder="Or tell me where you want to go…"
                  value={customPlace}
                  onChange={(e) => setCustomPlace(e.target.value)}
                />
              )}
            </div>

            <div className="dwy-field">
              <label className="dwy-field-label"><Sparkles size={15} style={{ verticalAlign: -2, marginRight: 6 }} />Date vibe (pick as many as you want)</label>
              <div className="dwy-chip-grid">
                {VIBES.map((v) => (
                  <button
                    type="button"
                    key={v.key}
                    className={"dwy-chip" + (vibes.includes(v.key) ? " active" : "")}
                    onClick={() => toggleVibe(v.key)}
                  >
                    <span>{v.icon}</span> {v.label}
                  </button>
                ))}
              </div>
            </div>

            <div className="dwy-field">
              <label className="dwy-field-label">Anything you want me to know? 💌</label>
              <textarea
                className="dwy-textarea"
                placeholder="Tell me anything… what you want to eat, where you want to go, what you want to do, or just leave me a cute message."
                value={specialRequest}
                onChange={(e) => setSpecialRequest(e.target.value)}
              />
            </div>

            <div className="dwy-field">
              <label className="dwy-field-label">Anything you want to tell me before our date? ❤️</label>
              <textarea
                className="dwy-textarea"
                value={finalMessage}
                onChange={(e) => setFinalMessage(e.target.value)}
              />
            </div>

            <button
              className="dwy-btn dwy-btn-primary"
              style={{ width: "100%" }}
              disabled={!date || !time || !place || saving}
              onClick={submit}
            >
              {saving ? "Saving…" : "Review Our Date →"}
            </button>
          </div>
        </div>
      )}

      {stage === "summary" && (
        <div className="dwy-stage">
          <Thread stage={stage} />
          <div className="dwy-card">
            <h2 className="dwy-h2 dwy-display" style={{ fontSize: 26 }}>Our Date ❤️</h2>
            <div className="dwy-summary-row"><span className="dwy-summary-k">Date</span><span className="dwy-summary-v">{dateNice}</span></div>
            <div className="dwy-summary-row"><span className="dwy-summary-k">Time</span><span className="dwy-summary-v">{timeNice}</span></div>
            <div className="dwy-summary-row"><span className="dwy-summary-k">Place</span><span className="dwy-summary-v">{placeLabel}</span></div>
            {vibes.length > 0 && (
              <div className="dwy-summary-row"><span className="dwy-summary-k">Vibe</span><span className="dwy-summary-v">{vibes.map((k) => VIBES.find((v) => v.key === k)?.label).join(", ")}</span></div>
            )}
            {specialRequest && (
              <div className="dwy-summary-row"><span className="dwy-summary-k">You said</span><span className="dwy-summary-v">{specialRequest}</span></div>
            )}
            {finalMessage && (
              <div className="dwy-summary-row"><span className="dwy-summary-k">Message</span><span className="dwy-summary-v">{finalMessage}</span></div>
            )}
            <p className="dwy-p" style={{ marginTop: 22 }}>Is this our plan? 🥰</p>
            <div style={{ display: "flex", gap: 12, justifyContent: "center", flexWrap: "wrap" }}>
              <button className="dwy-btn dwy-btn-primary" onClick={confirm}>YES ❤️</button>
              <button className="dwy-btn dwy-btn-ghost" onClick={() => setStage("planner")}>CHANGE SOMETHING ✏️</button>
            </div>
          </div>
        </div>
      )}

      {stage === "confirmed" && (
        <div className="dwy-stage">
          <h1 className="dwy-h1 dwy-display">DATE ACCEPTED ❤️🥹</h1>
          <p className="dwy-p">Okay… it's officially a date.</p>
          <div className="dwy-card" style={{ marginBottom: 24 }}>
            <div className="dwy-summary-row"><span className="dwy-summary-k">Date</span><span className="dwy-summary-v">{dateNice}</span></div>
            <div className="dwy-summary-row"><span className="dwy-summary-k">Time</span><span className="dwy-summary-v">{timeNice}</span></div>
            <div className="dwy-summary-row"><span className="dwy-summary-k">Place</span><span className="dwy-summary-v">{placeLabel}</span></div>
          </div>
          <p className="dwy-p dwy-display" style={{ fontSize: 22, fontStyle: "italic", color: "var(--wine)" }}>
            I can't wait to see you. ❤️
          </p>
          <p className="dwy-p" style={{ opacity: 0.7, fontSize: 14 }}>
            Date locked in. ❤️ Now there's only one thing left to do… wait for the day. 🥰
          </p>
        </div>
      )}

      {adminOpen && (
        <div className="dwy-modal-overlay" onClick={(e) => e.target === e.currentTarget && closeAdmin()}>
          <div className="dwy-modal">
            <button className="dwy-modal-close" onClick={closeAdmin} aria-label="Close"><X size={20} /></button>

            {!adminAuthed ? (
              <div style={{ textAlign: "center", padding: "20px 0" }}>
                <Lock size={26} color="var(--wine)" style={{ marginBottom: 10 }} />
                <h3 className="dwy-display" style={{ color: "var(--wine)", fontSize: 22, marginBottom: 6 }}>Admin only</h3>
                <p style={{ fontSize: 13, opacity: 0.7, marginBottom: 16 }}>This is a light passcode gate, not real security — don't reuse a sensitive password here.</p>
                <input
                  type="password"
                  className="dwy-input"
                  placeholder="Passcode"
                  value={passInput}
                  onChange={(e) => setPassInput(e.target.value)}
                  onKeyDown={(e) => e.key === "Enter" && tryLogin()}
                  style={{ maxWidth: 220, margin: "0 auto 10px" }}
                />
                {passError && <p style={{ color: "var(--deep-rose)", fontSize: 13, marginBottom: 10 }}>Wrong passcode.</p>}
                <button className="dwy-btn dwy-btn-primary" onClick={tryLogin}>Enter</button>
              </div>
            ) : viewing ? (
              <div style={{ textAlign: "left" }}>
                <button className="dwy-btn dwy-btn-ghost" style={{ marginBottom: 16, padding: "8px 16px", fontSize: 13 }} onClick={() => setViewing(null)}>
                  <ChevronLeft size={14} style={{ verticalAlign: -2 }} /> Back
                </button>
                <h3 className="dwy-display" style={{ color: "var(--wine)", fontSize: 22, marginBottom: 12 }}>Date request</h3>
                <div className="dwy-summary-row"><span className="dwy-summary-k">Date</span><span className="dwy-summary-v">{viewing.date}</span></div>
                <div className="dwy-summary-row"><span className="dwy-summary-k">Time</span><span className="dwy-summary-v">{viewing.time}</span></div>
                <div className="dwy-summary-row"><span className="dwy-summary-k">Place</span><span className="dwy-summary-v">{viewing.customPlace || viewing.place}</span></div>
                <div className="dwy-summary-row"><span className="dwy-summary-k">Vibe</span><span className="dwy-summary-v">{(viewing.vibes || []).join(", ") || "—"}</span></div>
                <div className="dwy-summary-row"><span className="dwy-summary-k">Note</span><span className="dwy-summary-v">{viewing.specialRequest || "—"}</span></div>
                <div className="dwy-summary-row"><span className="dwy-summary-k">Message</span><span className="dwy-summary-v">{viewing.finalMessage || "—"}</span></div>
                <div className="dwy-summary-row"><span className="dwy-summary-k">Submitted</span><span className="dwy-summary-v">{new Date(viewing.timestamp).toLocaleString()}</span></div>
                <div style={{ display: "flex", gap: 10, marginTop: 18 }}>
                  {!viewing.seen && (
                    <button className="dwy-btn dwy-btn-ghost" style={{ fontSize: 13, padding: "10px 16px" }} onClick={() => markSeen(viewing)}>Mark seen</button>
                  )}
                  <button className="dwy-btn dwy-btn-ghost" style={{ fontSize: 13, padding: "10px 16px", color: "var(--deep-rose)" }} onClick={() => removeSub(viewing.id)}>Delete</button>
                </div>
              </div>
            ) : (
              <div>
                <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 16 }}>
                  <h3 className="dwy-display" style={{ color: "var(--wine)", fontSize: 22 }}>Date requests</h3>
                  <button className="dwy-icon-btn" onClick={exportJson} aria-label="Export as JSON" title="Export JSON"><Download size={18} /></button>
                </div>
                {loadingSubs ? (
                  <p style={{ fontSize: 14, opacity: 0.7 }}>Loading…</p>
                ) : submissions.length === 0 ? (
                  <p style={{ fontSize: 14, opacity: 0.7 }}>No date requests yet.</p>
                ) : (
                  submissions.map((s) => (
                    <div key={s.id} className={"dwy-sub-item" + (s.seen ? "" : " unseen")}>
                      <div className="dwy-sub-main" onClick={() => setViewing(s)}>
                        <div className="dwy-sub-title">{s.customPlace || s.place || "No place chosen"} · {s.date}</div>
                        <div className="dwy-sub-sub">{new Date(s.timestamp).toLocaleString()} {!s.seen && "· new"}</div>
                      </div>
                      <button className="dwy-icon-btn" onClick={() => setViewing(s)} aria-label="View"><Eye size={16} /></button>
                      <button className="dwy-icon-btn" onClick={() => removeSub(s.id)} aria-label="Delete"><Trash2 size={16} /></button>
                    </div>
                  ))
                )}
              </div>
            )}
          </div>
        </div>
      )}
    </div>
  );
}
