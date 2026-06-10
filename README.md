<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="Coach Hybride">
<meta name="theme-color" content="#17191B">
<title>Coach Hybride</title>
<link href="https://cdnjs.cloudflare.com/ajax/libs/tailwindcss/2.2.19/tailwind.min.css" rel="stylesheet">
<link href="https://fonts.googleapis.com/css2?family=Barlow+Condensed:wght@600;700&family=Barlow:wght@400;500;600&family=IBM+Plex+Mono:wght@500;600&display=swap" rel="stylesheet">
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.5/babel.min.js"></script>
<style>
  html, body { background: #F2F3EF; }
  input, textarea { -webkit-appearance: none; appearance: none; }
  input:focus, textarea:focus { border-color: #23262A !important; outline: none; }
  * { -webkit-tap-highlight-color: transparent; }
</style>
</head>
<body>
<div id="root"></div>
<script type="text/babel">
const { useState, useEffect, useMemo } = React;

/* ------------------------------------------------------------------ */
/*  Design tokens — "craie & fonte" + code couleur disques de compèt   */
/* ------------------------------------------------------------------ */
const C = {
  chalk: "#F2F3EF",
  paper: "#FBFBF8",
  iron: "#17191B",
  ink: "#23262A",
  mute: "#6E7378",
  line: "#E1E3DC",
  types: {
    course: { label: "Course", color: "#2D5FD0", bg: "#E9EFFB" },
    muscu:  { label: "Muscu",  color: "#C2362B", bg: "#FAE9E7" },
    hyrox:  { label: "Hyrox",  color: "#A87E00", bg: "#FBF1D6" },
    recup:  { label: "Récup",  color: "#2F7D52", bg: "#E6F2EA" },
  },
};
const F = {
  disp: "'Barlow Condensed', ui-sans-serif, sans-serif",
  body: "'Barlow', ui-sans-serif, sans-serif",
  mono: "'IBM Plex Mono', ui-monospace, monospace",
};

const HYROX_STATIONS = [
  "SkiErg", "Sled Push", "Sled Pull", "Burpee Broad Jump",
  "Row", "Farmers Carry", "Lunges", "Wall Balls",
];
const RUN_KINDS = ["Footing", "Seuil", "Intervalles", "Sortie longue", "Compromised"];
const HALF_KM = 21.0975;

const DEFAULT_PROFILE = {
  hyroxLabel: "Hyrox Doubles",
  hyroxDate: "2026-11-14",
  semiDate: "2026-11-01",
  semiTargetMin: 110,
  semiPRMin: 118,
  benchCurrent: 100,
  benchTarget: 110,
  weightCurrent: 75,
  weightTarget: 70,
  weakPoints: "Wall balls (station faible)",
  constraints:
    "Périostite tibia gauche : surveiller le volume d'impact, progression douce. Salle = haut du corps + circuits Hyrox (pas de squat/SDT lourds). Créneau midi possible (45–60 min).",
};

const EMPTY_DRAFT = {
  type: "muscu", date: "", title: "", duration: "", rpe: 7,
  distance: "", timeMin: "", runKind: "", benchKg: "", benchReps: "", stations: [], notes: "",
};

/* ------------------------------------------------------------------ */
/*  Helpers                                                            */
/* ------------------------------------------------------------------ */
const uid = () => Date.now().toString(36) + Math.random().toString(36).slice(2, 7);
const todayISO = () => new Date().toISOString().slice(0, 10);
const fmtFR = (iso) => {
  try {
    return new Date(iso + "T12:00:00").toLocaleDateString("fr-FR", {
      weekday: "short", day: "numeric", month: "short",
    });
  } catch { return iso; }
};
const daysUntil = (iso) => Math.ceil((new Date(iso + "T00:00:00") - new Date()) / 86400000);
const epley = (kg, reps) => Math.round(kg * (1 + reps / 30));
const mondayOf = (d) => {
  const dt = new Date(d); const day = (dt.getDay() + 6) % 7;
  dt.setDate(dt.getDate() - day); dt.setHours(0, 0, 0, 0); return dt;
};
const paceOf = (minPerKm) => {
  if (!minPerKm || !isFinite(minPerKm)) return "–";
  let m = Math.floor(minPerKm), s = Math.round((minPerKm - m) * 60);
  if (s === 60) { m += 1; s = 0; }
  return m + "'" + String(s).padStart(2, "0") + "\"";
};
const paceStr = (min, km) => (!min || !km ? null : paceOf(min / km) + "/km");
const minToHM = (min) => {
  if (!min) return "–";
  let h = Math.floor(min / 60), m = Math.round(min % 60);
  if (m === 60) { h += 1; m = 0; }
  return h > 0 ? h + "h" + String(m).padStart(2, "0") : m + " min";
};
const runZones = (targetMin) => {
  const p = targetMin / HALF_KM;
  return { cible: p, seuil: p - 10 / 60, vma: p - 35 / 60, footMin: p + 60 / 60, footMax: p + 80 / 60 };
};

const SKEY = "coach-hybride:sessions";
const PKEY = "coach-hybride:profile";
function storeGet(key) {
  try { const v = localStorage.getItem(key); return v ? JSON.parse(v) : null; }
  catch { return null; }
}
function storeSet(key, value) {
  try { localStorage.setItem(key, JSON.stringify(value)); } catch (e) {}
}

/* ------------------------------------------------------------------ */
/*  Icônes (SVG inline)                                                */
/* ------------------------------------------------------------------ */
const Svg = ({ children, size = 18, style }) => (
  <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke="currentColor"
    strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" style={style}>{children}</svg>
);
const IcoActivity = (p) => <Svg {...p}><polyline points="22 12 18 12 15 21 9 3 6 12 2 12" /></Svg>;
const IcoPlus = (p) => <Svg {...p}><path d="M12 5v14M5 12h14" /></Svg>;
const IcoSpark = (p) => <Svg {...p}><path d="M12 3l1.9 5.1L19 10l-5.1 1.9L12 17l-1.9-5.1L5 10l5.1-1.9L12 3z" /></Svg>;
const IcoHistory = (p) => <Svg {...p}><circle cx="12" cy="12" r="9" /><polyline points="12 7 12 12 15 14" /></Svg>;
const IcoGear = (p) => <Svg {...p}><circle cx="12" cy="12" r="3" /><path d="M12 2v3M12 19v3M2 12h3M19 12h3M4.9 4.9l2.1 2.1M17 17l2.1 2.1M19.1 4.9L17 7M7 17l-2.1 2.1" /></Svg>;
const IcoX = (p) => <Svg {...p}><path d="M6 6l12 12M18 6L6 18" /></Svg>;
const IcoTrash = (p) => <Svg {...p}><polyline points="3 6 21 6" /><path d="M8 6V4h8v2M6 6l1 14h10l1-14" /></Svg>;
const IcoCheck = (p) => <Svg {...p}><polyline points="20 6 9 17 4 12" /></Svg>;
const IcoChevron = (p) => <Svg {...p}><polyline points="9 18 15 12 9 6" /></Svg>;
const IcoTimer = (p) => <Svg {...p}><circle cx="12" cy="13" r="8" /><path d="M12 9v4M10 2h4" /></Svg>;
const IcoTarget = (p) => <Svg {...p}><circle cx="12" cy="12" r="9" /><circle cx="12" cy="12" r="5" /><circle cx="12" cy="12" r="1.5" /></Svg>;
const IcoDumbbell = (p) => <Svg {...p}><path d="M2 12h3M19 12h3M6 7v10M18 7v10M9 9v6M15 9v6M9 12h6" /></Svg>;

/* ------------------------------------------------------------------ */
/*  Mini-graphiques SVG maison                                         */
/* ------------------------------------------------------------------ */
const MiniBars = ({ data, color, fmt }) => {
  const max = Math.max(1, ...data.map((d) => d.v));
  return (
    <div className="mt-2">
      <div className="flex items-end" style={{ height: 110, gap: 6 }}>
        {data.map((d, i) => (
          <div key={i} className="flex-1 flex flex-col justify-end" style={{ height: "100%" }}>
            <div className="text-center" style={{ fontFamily: F.mono, fontSize: 10, color: C.mute, marginBottom: 2 }}>
              {d.v ? fmt(d.v) : ""}
            </div>
            <div style={{
              height: Math.max((d.v / max) * 84, d.v ? 3 : 1),
              background: d.v ? color : C.line,
              borderRadius: "3px 3px 0 0",
            }} />
          </div>
        ))}
      </div>
      <div className="flex" style={{ gap: 6 }}>
        {data.map((d, i) => (
          <div key={i} className="flex-1 text-center" style={{ fontFamily: F.mono, fontSize: 10, color: C.mute, marginTop: 4 }}>
            {d.label}
          </div>
        ))}
      </div>
    </div>
  );
};

const MiniLine = ({ pts, color, refY, refColor, fmtY, reversed }) => {
  const W = 320, H = 140, PL = 40, PR = 10, PT = 10, PB = 22;
  const ys = pts.map((p) => p.y).concat(refY != null ? [refY] : []);
  let min = Math.min(...ys), max = Math.max(...ys);
  if (max - min < 0.001) { max += 1; min -= 1; }
  const pad = (max - min) * 0.18; min -= pad; max += pad;
  const X = (i) => PL + (i * (W - PL - PR)) / Math.max(pts.length - 1, 1);
  const Y = (v) => {
    const t = (v - min) / (max - min);
    return PT + (reversed ? t : 1 - t) * (H - PT - PB);
  };
  const ticks = [min + (max - min) * 0.15, (min + max) / 2, max - (max - min) * 0.15];
  const line = pts.map((p, i) => X(i) + "," + Y(p.y)).join(" ");
  return (
    <svg viewBox={"0 0 " + W + " " + H} className="w-full" style={{ marginTop: 8 }}>
      {ticks.map((t, i) => (
        <g key={i}>
          <line x1={PL} y1={Y(t)} x2={W - PR} y2={Y(t)} stroke={C.line} strokeWidth="1" />
          <text x={PL - 5} y={Y(t) + 3} textAnchor="end" fontSize="9" fill={C.mute} fontFamily={F.mono}>{fmtY(t)}</text>
        </g>
      ))}
      {refY != null && (
        <line x1={PL} y1={Y(refY)} x2={W - PR} y2={Y(refY)} stroke={refColor} strokeWidth="1.5" strokeDasharray="4 4" />
      )}
      <polyline points={line} fill="none" stroke={color} strokeWidth="2" />
      {pts.map((p, i) => <circle key={i} cx={X(i)} cy={Y(p.y)} r="3" fill={color} />)}
      {pts.length > 0 && (
        <g>
          <text x={X(0)} y={H - 6} textAnchor="start" fontSize="9" fill={C.mute} fontFamily={F.mono}>{pts[0].label}</text>
          {pts.length > 1 && (
            <text x={X(pts.length - 1)} y={H - 6} textAnchor="end" fontSize="9" fill={C.mute} fontFamily={F.mono}>
              {pts[pts.length - 1].label}
            </text>
          )}
        </g>
      )}
    </svg>
  );
};

/* ------------------------------------------------------------------ */
/*  App                                                                */
/* ------------------------------------------------------------------ */
function CoachHybride() {
  const [tab, setTab] = useState("home");
  const [profile, setProfile] = useState(() => ({ ...DEFAULT_PROFILE, ...(storeGet(PKEY) || {}) }));
  const [sessions, setSessions] = useState(() => storeGet(SKEY) || []);
  const [settingsOpen, setSettingsOpen] = useState(false);
  const [draft, setDraft] = useState({ ...EMPTY_DRAFT, date: todayISO() });
  const [toast, setToast] = useState(null);
  const [pendingDelete, setPendingDelete] = useState(null);
  const [pDraft, setPDraft] = useState(DEFAULT_PROFILE);
  const [backup, setBackup] = useState("");
  const [proposal, setProposal] = useState(null);

  const storageOk = useMemo(() => {
    try { localStorage.setItem("__t", "1"); localStorage.removeItem("__t"); return true; }
    catch { return false; }
  }, []);

  const notify = (msg) => {
    setToast(msg);
    setTimeout(() => setToast(null), 2400);
  };

  /* ---- mutations ---- */
  const addSession = () => {
    if (!draft.date) { notify("Date manquante"); return; }
    const t = C.types[draft.type];
    const sess = {
      id: uid(),
      type: draft.type,
      date: draft.date,
      title: draft.title.trim() || t.label,
      duration: parseFloat(draft.duration) || null,
      rpe: draft.rpe,
      distance: parseFloat(draft.distance) || null,
      timeMin: parseFloat(draft.timeMin) || null,
      runKind: draft.runKind || null,
      benchKg: parseFloat(draft.benchKg) || null,
      benchReps: parseInt(draft.benchReps) || null,
      stations: draft.stations,
      notes: draft.notes.trim(),
    };
    const next = [sess, ...sessions].sort((a, b) => (a.date < b.date ? 1 : -1));
    setSessions(next);
    storeSet(SKEY, next);
    setDraft({ ...EMPTY_DRAFT, date: todayISO() });
    notify("Séance enregistrée");
    setTab("history");
  };

  const deleteSession = (id) => {
    const next = sessions.filter((s) => s.id !== id);
    setSessions(next);
    storeSet(SKEY, next);
    setPendingDelete(null);
    notify("Séance supprimée");
  };

  const saveProfile = (p) => {
    setProfile(p);
    storeSet(PKEY, p);
    setSettingsOpen(false);
    notify("Profil mis à jour");
  };

  const markDone = (id) => {
    const next = sessions
      .map((s) => (s.id === id ? { ...s, planned: false, date: todayISO() } : s))
      .sort((a, b) => (a.date < b.date ? 1 : -1));
    setSessions(next);
    storeSet(SKEY, next);
    notify("Séance marquée réalisée");
  };

  /* ---- dérivés ---- */
  const zones = useMemo(() => runZones(profile.semiTargetMin || 110), [profile.semiTargetMin]);
  const prPace = (profile.semiPRMin || 118) / HALF_KM;

  const benchSeries = useMemo(() => {
    return sessions
      .filter((s) => !s.planned && s.benchKg && s.benchReps)
      .map((s) => ({ date: s.date, e1rm: epley(s.benchKg, s.benchReps) }))
      .sort((a, b) => (a.date > b.date ? 1 : -1));
  }, [sessions]);

  const bestE1RM = useMemo(() => {
    const logged = benchSeries.length ? Math.max(...benchSeries.map((d) => d.e1rm)) : 0;
    return Math.max(logged, profile.benchCurrent || 0);
  }, [benchSeries, profile.benchCurrent]);

  const paceSeries = useMemo(() => {
    return sessions
      .filter((s) => !s.planned && s.type === "course" && s.distance && s.timeMin)
      .map((s) => ({ date: s.date, pace: s.timeMin / s.distance }))
      .sort((a, b) => (a.date > b.date ? 1 : -1));
  }, [sessions]);

  const weekBuckets = (fill) => {
    const buckets = [];
    const start = mondayOf(new Date());
    for (let i = 5; i >= 0; i--) {
      const d = new Date(start); d.setDate(d.getDate() - i * 7);
      buckets.push({ key: d.toISOString().slice(0, 10), label: i === 0 ? "S" : "S-" + i, v: 0 });
    }
    sessions.forEach((s) => {
      if (s.planned) return;
      const wk = mondayOf(new Date(s.date + "T12:00:00")).toISOString().slice(0, 10);
      const b = buckets.find((x) => x.key === wk);
      if (b) b.v += fill(s);
    });
    return buckets;
  };
  const weeklyKm = useMemo(() => weekBuckets((s) => (s.type === "course" ? s.distance || 0 : 0)), [sessions]);
  const weeklyMin = useMemo(() => weekBuckets((s) => s.duration || 0), [sessions]);

  const thisWeek = useMemo(() => {
    const wk = mondayOf(new Date()).toISOString().slice(0, 10);
    const list = sessions.filter(
      (s) => !s.planned && mondayOf(new Date(s.date + "T12:00:00")).toISOString().slice(0, 10) === wk
    );
    const runs = list.filter((s) => s.type === "course");
    return {
      count: list.length,
      min: list.reduce((a, s) => a + (s.duration || 0), 0),
      km: runs.reduce((a, s) => a + (s.distance || 0), 0),
      runs: runs.length,
    };
  }, [sessions]);

  /* ---- Coach : moteur local ---- */
  const localProposal = () => {
    const now = new Date(todayISO() + "T12:00:00");
    const done = sessions.filter((s) => !s.planned);
    const within = (days, fn) =>
      done.filter((s) => (now - new Date(s.date + "T12:00:00")) / 86400000 <= days && fn(s));
    const runs7 = within(7, (s) => s.type === "course");
    const quality7 = runs7.filter((s) => ["Seuil", "Intervalles"].includes(s.runKind));
    const long7 = runs7.filter((s) => s.runKind === "Sortie longue");
    const hyrox7 = within(7, (s) => s.type === "hyrox");
    const last = done[0];
    const lastTwoHard = done.length >= 2 && done.slice(0, 2).every((s) => (s.rpe || 0) >= 8);
    const dc = Math.round((bestE1RM * 0.8) / 2.5) * 2.5;

    if (lastTwoHard) return {
      type: "recup", titre: "Récupération active", duree_min: 40,
      echauffement: ["10 min de marche rapide ou vélo très facile"],
      blocs: [{ nom: "Aérobie Z1 + mobilité", contenu: [
        "20 min footing très facile (" + paceOf(zones.footMax) + "/km) ou vélo",
        "10 min mobilité hanches/épaules + auto-massage mollet gauche"] }],
      conseils: ["RPE max 4 — tes deux dernières séances étaient dures", "Surveille le tibia gauche"],
      justification: "Deux séances consécutives à RPE >= 8 : on absorbe la charge avant la prochaine qualité.",
    };

    if ((!last || last.type !== "course") && runs7.length < 3) {
      if (quality7.length === 0) return {
        type: "course", titre: "Seuil 3 × 8 min", duree_min: 55,
        echauffement: ["15 min footing " + paceOf(zones.footMin) + "–" + paceOf(zones.footMax) + "/km", "3 lignes droites progressives"],
        blocs: [
          { nom: "Corps de séance", contenu: ["3 × 8 min à " + paceOf(zones.seuil) + "/km, récup 2 min trot", "Cadence haute, appuis légers (tibia)"] },
          { nom: "Retour au calme", contenu: ["10 min footing très facile"] }],
        conseils: ["LA séance qui rapproche du " + minToHM(profile.semiTargetMin) + " — non négociable", "Douleur tibiale > 3/10 : coupe le 3e bloc"],
        justification: "Aucune séance qualité course sur 7 jours : le seuil est le moteur direct de l'objectif semi.",
      };
      if (long7.length === 0) return {
        type: "course", titre: "Sortie longue progressive", duree_min: 75,
        echauffement: ["10 premières minutes très faciles"],
        blocs: [{ nom: "Corps de séance", contenu: [
          "60–70 min à " + paceOf(zones.footMin) + "/km",
          "10 dernières minutes à allure semi " + paceOf(zones.cible) + "/km"] }],
        conseils: ["+10 % de durée max vs ta dernière sortie longue", "Hydratation si > 1 h"],
        justification: "Pas de sortie longue cette semaine : elle construit le fond du semi et des 8 km du Hyrox.",
      };
      return {
        type: "course", titre: "Footing aérobie", duree_min: 45,
        echauffement: ["Départ très progressif 5 min"],
        blocs: [{ nom: "Corps de séance", contenu: [
          "40 min entre " + paceOf(zones.footMin) + " et " + paceOf(zones.footMax) + "/km",
          "Finir par 4 lignes droites relâchées"] }],
        conseils: ["Le volume facile doit rester ~80 % du kilométrage", "Surface souple si possible (tibia)"],
        justification: "La qualité de la semaine est faite : volume facile sans stresser le tibia.",
      };
    }

    if (hyrox7.length === 0) return {
      type: "hyrox", titre: "Circuit stations + wall balls", duree_min: 50,
      echauffement: ["5 min rameur progressif", "2 tours légers : 10 wall balls légers + 10 fentes"],
      blocs: [{ nom: "Circuit · 3 tours chronométrés", contenu: [
        "500 m SkiErg ou rameur",
        "20 m farmers carry lourd",
        "15 fentes avec charge",
        "20 wall balls — technique propre (point faible)",
        "400 m run à allure semi " + paceOf(zones.cible) + "/km"] }],
      conseils: ["Vise la régularité des tours, pas le sprint du 1er", "Wall balls : casse en séries propres plutôt qu'à l'échec"],
      justification: "Pas de travail stations sur 7 jours et wall balls = point faible : circuit avec compromised running.",
    };

    return {
      type: "muscu", titre: "Push haut du corps + DC lourd", duree_min: 60,
      echauffement: ["5 min rameur", "2 séries de DC légères (50 %)"],
      blocs: [
        { nom: "Force", contenu: ["DC 5 × 5 @ " + dc + " kg (~80 % e1RM)", "Développé militaire 4 × 8", "Dips lestés 3 × 8"] },
        { nom: "Accessoires", contenu: ["Tractions 4 × max", "Élévations latérales 3 × 12", "Gainage 3 × 45 s"] }],
      conseils: ["Repos 2–3 min sur le DC", "Logge ta meilleure série pour alimenter la courbe e1RM"],
      justification: "Course et stations couvertes récemment : on avance vers les " + profile.benchTarget + " kg au DC.",
    };
  };

  const generateSession = () => {
    setProposal(localProposal());
  };

  const proposalLines = (prop) => [
    ...(prop.echauffement || []).map((l) => "Échauffement · " + l),
    ...((prop.blocs || []).flatMap((b) => [
      (b.nom || "Bloc").toUpperCase(),
      ...(b.contenu || []).map((l) => "• " + l),
    ])),
  ];

  const prefillFromProposal = () => {
    if (!proposal) return;
    setDraft({
      ...EMPTY_DRAFT,
      date: todayISO(),
      type: C.types[proposal.type] ? proposal.type : "muscu",
      title: proposal.titre || "",
      duration: proposal.duree_min ? String(proposal.duree_min) : "",
      notes: proposalLines(proposal).join("\n"),
    });
    setTab("log");
    notify("Journal pré-rempli");
  };

  const planFromProposal = () => {
    if (!proposal) return;
    const sess = {
      id: uid(),
      type: C.types[proposal.type] ? proposal.type : "muscu",
      date: todayISO(),
      title: proposal.titre || "Séance planifiée",
      duration: proposal.duree_min || null,
      rpe: null,
      distance: null, timeMin: null, runKind: null,
      benchKg: null, benchReps: null, stations: [],
      notes: proposalLines(proposal).join("\n"),
      planned: true,
    };
    const next = [sess, ...sessions].sort((a, b) => (a.date < b.date ? 1 : -1));
    setSessions(next);
    storeSet(SKEY, next);
    notify("Séance planifiée — retrouve-la dans l'historique");
  };

  /* ------------------------------------------------------------------ */
  /*  UI fragments                                                       */
  /* ------------------------------------------------------------------ */
  const Eyebrow = ({ children }) => (
    <div className="text-xs uppercase" style={{ fontFamily: F.disp, letterSpacing: "0.14em", color: C.mute, fontWeight: 600 }}>
      {children}
    </div>
  );

  const SectionTitle = ({ color, children }) => (
    <div className="flex items-center pt-2" style={{ gap: 8 }}>
      <span style={{ width: 10, height: 10, borderRadius: 99, background: color, display: "inline-block" }} />
      <div className="text-sm uppercase" style={{ fontFamily: F.disp, letterSpacing: "0.14em", color: C.ink, fontWeight: 700 }}>
        {children}
      </div>
      <div className="flex-1" style={{ height: 1, background: C.line }} />
    </div>
  );

  const TypeBadge = ({ type }) => {
    const t = C.types[type] || C.types.muscu;
    return (
      <span className="inline-flex items-center px-2 rounded-full text-xs"
        style={{ gap: 6, paddingTop: 2, paddingBottom: 2, background: t.bg, color: t.color, fontFamily: F.disp, fontWeight: 700, letterSpacing: "0.06em" }}>
        <span style={{ width: 8, height: 8, borderRadius: 99, background: t.color, display: "inline-block" }} />
        {t.label.toUpperCase()}
      </span>
    );
  };

  const Card = ({ children }) => (
    <div className="rounded-xl p-4" style={{ background: C.paper, border: "1px solid " + C.line }}>
      {children}
    </div>
  );

  /* ---- Accueil ---- */
  const Home = () => {
    const dSemi = Math.max(daysUntil(profile.semiDate), 0);
    const dHyrox = Math.max(daysUntil(profile.hyroxDate), 0);
    const pct = Math.min(100, Math.round((bestE1RM / profile.benchTarget) * 100));
    const gain = Math.round((prPace - zones.cible) * 60);
    return (
      <div className="space-y-3">
        {!storageOk && (
          <div className="rounded-xl p-3 text-sm"
            style={{ background: C.types.hyrox.bg, color: C.types.hyrox.color, fontFamily: F.body, fontWeight: 600 }}>
            Stockage navigateur indisponible (navigation privée ?) : tes données ne seront pas conservées. Utilise ⚙️ → Exporter.
          </div>
        )}

        <div className="rounded-xl p-5" style={{ background: C.iron, color: C.chalk }}>
          <div className="grid grid-cols-2 gap-4">
            <div>
              <Eyebrow>Semi · {fmtFR(profile.semiDate)}</Eyebrow>
              <div style={{ fontFamily: F.mono, fontSize: 36, fontWeight: 600, lineHeight: 1.1 }}>J-{dSemi}</div>
              <div className="text-xs" style={{ marginTop: 4, fontFamily: F.mono, color: "#9BA0A6" }}>
                obj {minToHM(profile.semiTargetMin)} · PR {minToHM(profile.semiPRMin)}
              </div>
            </div>
            <div style={{ paddingLeft: 16, borderLeft: "1px solid #34373B" }}>
              <Eyebrow>{profile.hyroxLabel} · {fmtFR(profile.hyroxDate)}</Eyebrow>
              <div style={{ fontFamily: F.mono, fontSize: 36, fontWeight: 600, lineHeight: 1.1 }}>J-{dHyrox}</div>
              <div className="text-xs" style={{ marginTop: 4, fontFamily: F.mono, color: "#9BA0A6" }}>
                8 × 1 km entre stations
              </div>
            </div>
          </div>
        </div>

        <SectionTitle color={C.types.course.color}>Course</SectionTitle>

        <div className="grid grid-cols-2 gap-3">
          <Card>
            <Eyebrow>Cette semaine</Eyebrow>
            <div style={{ marginTop: 4, fontFamily: F.mono, fontSize: 26, fontWeight: 600, color: C.ink }}>
              {Math.round(thisWeek.km * 10) / 10}<span className="text-sm" style={{ color: C.mute }}> km</span>
            </div>
            <div className="text-xs" style={{ marginTop: 8, color: C.mute }}>
              {thisWeek.runs} sortie{thisWeek.runs > 1 ? "s" : ""} loggée{thisWeek.runs > 1 ? "s" : ""}
            </div>
          </Card>
          <Card>
            <Eyebrow>Allure cible semi</Eyebrow>
            <div style={{ marginTop: 4, fontFamily: F.mono, fontSize: 26, fontWeight: 600, color: C.types.course.color }}>
              {paceOf(zones.cible)}<span className="text-sm" style={{ color: C.mute }}>/km</span>
            </div>
            <div className="text-xs" style={{ marginTop: 8, color: C.mute }}>
              PR {paceOf(prPace)}/km → {gain}"/km à gagner
            </div>
          </Card>
        </div>

        <Card>
          <Eyebrow>Allures d'entraînement (objectif {minToHM(profile.semiTargetMin)})</Eyebrow>
          <div className="space-y-1" style={{ marginTop: 8 }}>
            {[
              ["Footing facile", paceOf(zones.footMin) + "–" + paceOf(zones.footMax)],
              ["Seuil / tempo", "~" + paceOf(zones.seuil)],
              ["Allure semi", paceOf(zones.cible)],
              ["Intervalles / VMA", "~" + paceOf(zones.vma)],
            ].map(([k, v]) => (
              <div key={k} className="flex items-center justify-between text-sm">
                <span style={{ color: C.ink, fontFamily: F.body }}>{k}</span>
                <span style={{ fontFamily: F.mono, fontWeight: 600, color: C.types.course.color }}>{v}/km</span>
              </div>
            ))}
          </div>
        </Card>

        <Card>
          <Eyebrow>Kilométrage · 6 dernières semaines</Eyebrow>
          <MiniBars data={weeklyKm} color={C.types.course.color} fmt={(v) => Math.round(v * 10) / 10} />
        </Card>

        <Card>
          <Eyebrow>Allure des sorties vs cible {paceOf(zones.cible)}/km</Eyebrow>
          {paceSeries.length >= 2 ? (
            <MiniLine
              pts={paceSeries.map((p) => ({ label: p.date.slice(8, 10) + "/" + p.date.slice(5, 7), y: p.pace }))}
              color={C.types.course.color} refY={zones.cible} refColor={C.types.hyrox.color}
              fmtY={(v) => paceOf(v)} reversed={true} />
          ) : (
            <p className="text-sm" style={{ marginTop: 8, color: C.mute }}>
              Logge au moins 2 sorties avec distance + temps : la courbe d'allure s'affichera ici (plus haut = plus rapide).
            </p>
          )}
        </Card>

        <SectionTitle color={C.types.muscu.color}>Force</SectionTitle>

        <Card>
          <div className="flex items-center justify-between">
            <Eyebrow>Développé couché</Eyebrow>
            <IcoDumbbell size={14} style={{ color: C.mute }} />
          </div>
          <div style={{ marginTop: 4, fontFamily: F.mono, fontSize: 26, fontWeight: 600, color: C.ink }}>
            {bestE1RM}<span className="text-sm" style={{ color: C.mute }}> /{profile.benchTarget} kg</span>
          </div>
          <div className="rounded-full" style={{ marginTop: 8, height: 6, background: C.line }}>
            <div className="rounded-full" style={{ height: 6, width: pct + "%", background: C.types.muscu.color }} />
          </div>
          <div className="text-xs" style={{ marginTop: 4, color: C.mute }}>{pct}% de la cible (e1RM)</div>
          {benchSeries.length >= 2 ? (
            <MiniLine
              pts={benchSeries.map((p) => ({ label: p.date.slice(8, 10) + "/" + p.date.slice(5, 7), y: p.e1rm }))}
              color={C.types.muscu.color} refY={profile.benchTarget} refColor={C.types.hyrox.color}
              fmtY={(v) => Math.round(v)} reversed={false} />
          ) : (
            <p className="text-sm" style={{ marginTop: 12, color: C.mute }}>
              Logge au moins 2 séances muscu avec ta meilleure série de DC pour tracer la courbe vers les {profile.benchTarget} kg.
            </p>
          )}
        </Card>

        <SectionTitle color={C.iron}>Charge globale</SectionTitle>

        <Card>
          <Eyebrow>
            {thisWeek.count} séance{thisWeek.count > 1 ? "s" : ""} cette semaine
            {thisWeek.min ? " · " + Math.round(thisWeek.min) + " min" : ""}
          </Eyebrow>
          <MiniBars data={weeklyMin} color={C.ink} fmt={(v) => Math.round(v)} />
        </Card>

        {sessions.length === 0 && (
          <Card>
            <p className="text-sm" style={{ color: C.ink }}>
              Aucune séance pour l'instant. Enregistre ta première séance dans le journal, ou demande une proposition au Coach.
            </p>
            <button onClick={() => setTab("coach")}
              className="inline-flex items-center text-sm px-3 py-2 rounded-lg"
              style={{ gap: 6, marginTop: 12, background: C.iron, color: C.chalk, fontFamily: F.disp, fontWeight: 700, letterSpacing: "0.04em" }}>
              <IcoSpark size={14} /> OUVRIR LE COACH
            </button>
          </Card>
        )}
      </div>
    );
  };

  /* ---- Journal ---- */
  const Log = () => {
    const set = (k, v) => setDraft((d) => ({ ...d, [k]: v }));
    const toggleStation = (s) =>
      set("stations", draft.stations.includes(s)
        ? draft.stations.filter((x) => x !== s)
        : [...draft.stations, s]);
    const pace = paceStr(parseFloat(draft.timeMin), parseFloat(draft.distance));

    const input = {
      className: "w-full px-3 rounded-lg text-base",
      style: { paddingTop: 10, paddingBottom: 10, background: C.paper, border: "1px solid " + C.line, color: C.ink, fontFamily: F.body },
    };

    return (
      <div className="space-y-4">
        <div>
          <Eyebrow>Type de séance</Eyebrow>
          <div className="grid grid-cols-4 gap-2" style={{ marginTop: 6 }}>
            {Object.entries(C.types).map(([k, t]) => {
              const on = draft.type === k;
              return (
                <button key={k} onClick={() => set("type", k)}
                  className="py-2 rounded-lg text-sm"
                  style={{
                    fontFamily: F.disp, fontWeight: 700, letterSpacing: "0.05em",
                    background: on ? t.color : C.paper,
                    color: on ? "#FFFFFF" : C.mute,
                    border: "1px solid " + (on ? t.color : C.line),
                  }}>
                  {t.label.toUpperCase()}
                </button>
              );
            })}
          </div>
        </div>

        <div className="grid grid-cols-2 gap-3">
          <div>
            <Eyebrow>Date</Eyebrow>
            <input type="date" value={draft.date} onChange={(e) => set("date", e.target.value)} {...input} />
          </div>
          <div>
            <Eyebrow>Durée (min)</Eyebrow>
            <input inputMode="numeric" placeholder="60" value={draft.duration}
              onChange={(e) => set("duration", e.target.value)} {...input} />
          </div>
        </div>

        <div>
          <Eyebrow>Titre</Eyebrow>
          <input placeholder={"Ex. " + (draft.type === "course" ? "Tempo 8 km" : draft.type === "hyrox" ? "Circuit stations 1–4" : "Push haut du corps")}
            value={draft.title} onChange={(e) => set("title", e.target.value)} {...input} />
        </div>

        {draft.type === "course" && (
          <React.Fragment>
            <div>
              <Eyebrow>Type de sortie</Eyebrow>
              <div className="flex flex-wrap" style={{ gap: 8, marginTop: 6 }}>
                {RUN_KINDS.map((k) => {
                  const on = draft.runKind === k;
                  return (
                    <button key={k} onClick={() => set("runKind", on ? "" : k)}
                      className="rounded-full text-xs"
                      style={{
                        padding: "6px 10px", fontFamily: F.body, fontWeight: 600,
                        background: on ? C.types.course.bg : C.paper,
                        color: on ? C.types.course.color : C.mute,
                        border: "1px solid " + (on ? C.types.course.color : C.line),
                      }}>
                      {k}
                    </button>
                  );
                })}
              </div>
            </div>
            <div className="grid grid-cols-2 gap-3">
              <div>
                <Eyebrow>Distance (km)</Eyebrow>
                <input inputMode="decimal" placeholder="10" value={draft.distance}
                  onChange={(e) => set("distance", e.target.value)} {...input} />
              </div>
              <div>
                <Eyebrow>Temps (min)</Eyebrow>
                <input inputMode="decimal" placeholder="52" value={draft.timeMin}
                  onChange={(e) => set("timeMin", e.target.value)} {...input} />
                {pace && (
                  <div className="text-xs" style={{ marginTop: 4, fontFamily: F.mono, color: C.types.course.color }}>
                    Allure {pace}
                  </div>
                )}
              </div>
            </div>
          </React.Fragment>
        )}

        {draft.type === "muscu" && (
          <div className="grid grid-cols-2 gap-3">
            <div>
              <Eyebrow>DC · meilleure série (kg)</Eyebrow>
              <input inputMode="decimal" placeholder="90" value={draft.benchKg}
                onChange={(e) => set("benchKg", e.target.value)} {...input} />
            </div>
            <div>
              <Eyebrow>Reps</Eyebrow>
              <input inputMode="numeric" placeholder="5" value={draft.benchReps}
                onChange={(e) => set("benchReps", e.target.value)} {...input} />
              {draft.benchKg && draft.benchReps && (
                <div className="text-xs" style={{ marginTop: 4, fontFamily: F.mono, color: C.types.muscu.color }}>
                  e1RM ≈ {epley(parseFloat(draft.benchKg), parseInt(draft.benchReps))} kg
                </div>
              )}
            </div>
          </div>
        )}

        {draft.type === "hyrox" && (
          <div>
            <Eyebrow>Stations travaillées</Eyebrow>
            <div className="flex flex-wrap" style={{ gap: 8, marginTop: 6 }}>
              {HYROX_STATIONS.map((s) => {
                const on = draft.stations.includes(s);
                return (
                  <button key={s} onClick={() => toggleStation(s)}
                    className="rounded-full text-xs"
                    style={{
                      padding: "6px 10px", fontFamily: F.body, fontWeight: 600,
                      background: on ? C.types.hyrox.bg : C.paper,
                      color: on ? C.types.hyrox.color : C.mute,
                      border: "1px solid " + (on ? C.types.hyrox.color : C.line),
                    }}>
                    {s}
                  </button>
                );
              })}
            </div>
          </div>
        )}

        <div>
          <Eyebrow>RPE · {draft.rpe}/10</Eyebrow>
          <div className="flex" style={{ gap: 4, marginTop: 6 }}>
            {Array.from({ length: 10 }, (_, i) => i + 1).map((n) => (
              <button key={n} onClick={() => set("rpe", n)}
                className="flex-1 rounded-md text-xs"
                style={{
                  height: 36, fontFamily: F.mono, fontWeight: 600,
                  background: n <= draft.rpe ? C.iron : C.paper,
                  color: n <= draft.rpe ? C.chalk : C.mute,
                  border: "1px solid " + (n <= draft.rpe ? C.iron : C.line),
                }}>
                {n}
              </button>
            ))}
          </div>
        </div>

        <div>
          <Eyebrow>Notes / contenu</Eyebrow>
          <textarea rows={5} placeholder="Exos, séries, temps par station, sensations…"
            value={draft.notes} onChange={(e) => set("notes", e.target.value)} {...input} />
        </div>

        <button onClick={addSession}
          className="w-full rounded-xl flex items-center justify-center"
          style={{ gap: 8, padding: "14px 0", background: C.iron, color: C.chalk, fontFamily: F.disp, fontWeight: 700, fontSize: 16, letterSpacing: "0.06em" }}>
          <IcoCheck size={18} /> ENREGISTRER LA SÉANCE
        </button>
      </div>
    );
  };

  /* ---- Coach ---- */
  const Coach = () => (
    <div className="space-y-3">
      <div className="rounded-xl p-4" style={{ background: C.iron, color: C.chalk }}>
        <Eyebrow>Le brief du coach</Eyebrow>
        <p className="text-sm" style={{ marginTop: 4, fontFamily: F.body, color: "#C8CCD0" }}>
          Moteur embarqué : analyse tes séances réalisées + double échéance
          (semi {minToHM(profile.semiTargetMin)} à J-{Math.max(daysUntil(profile.semiDate), 0)},
          Hyrox à J-{Math.max(daysUntil(profile.hyroxDate), 0)}), allures dérivées de l'objectif,
          contraintes (périostite, wall balls, créneau midi). Pour les séances générées par l'IA + Strava,
          utilise la version Claude de l'app sur claude.ai (web).
        </p>
      </div>

      <button onClick={generateSession}
        className="w-full rounded-xl flex items-center justify-center"
        style={{ gap: 8, padding: "14px 0", background: C.iron, color: C.chalk, fontFamily: F.disp, fontWeight: 700, fontSize: 16, letterSpacing: "0.06em" }}>
        <IcoSpark size={18} /> GÉNÉRER MA PROCHAINE SÉANCE
      </button>

      {proposal && (
        <Card>
          <div className="flex items-center justify-between">
            <TypeBadge type={C.types[proposal.type] ? proposal.type : "muscu"} />
            <div className="flex items-center text-sm" style={{ gap: 4, fontFamily: F.mono, color: C.mute }}>
              <IcoTimer size={14} /> {proposal.duree_min || "–"} min
            </div>
          </div>
          <h2 style={{ marginTop: 8, fontFamily: F.disp, fontWeight: 700, fontSize: 24, color: C.ink, lineHeight: 1.1 }}>
            {proposal.titre}
          </h2>

          {(proposal.echauffement || []).length > 0 && (
            <div style={{ marginTop: 12 }}>
              <Eyebrow>Échauffement</Eyebrow>
              <ul className="space-y-1" style={{ marginTop: 4 }}>
                {proposal.echauffement.map((l, i) => (
                  <li key={i} className="text-sm flex" style={{ gap: 8, color: C.ink, fontFamily: F.body }}>
                    <span style={{ color: C.mute }}>—</span> {l}
                  </li>
                ))}
              </ul>
            </div>
          )}

          {(proposal.blocs || []).map((b, i) => (
            <div key={i} style={{ marginTop: 12, paddingTop: 12, borderTop: "1px solid " + C.line }}>
              <Eyebrow>{b.nom || "Bloc " + (i + 1)}</Eyebrow>
              <ul className="space-y-1" style={{ marginTop: 4 }}>
                {(b.contenu || []).map((l, j) => (
                  <li key={j} className="text-sm flex" style={{ gap: 8, color: C.ink, fontFamily: F.body }}>
                    <IcoChevron size={14} style={{ color: C.mute, flexShrink: 0, marginTop: 2 }} /> {l}
                  </li>
                ))}
              </ul>
            </div>
          ))}

          {(proposal.conseils || []).length > 0 && (
            <div style={{ marginTop: 12, paddingTop: 12, borderTop: "1px solid " + C.line }}>
              <Eyebrow>Conseils</Eyebrow>
              <ul className="space-y-1" style={{ marginTop: 4 }}>
                {proposal.conseils.map((l, i) => (
                  <li key={i} className="text-sm flex" style={{ gap: 8, color: C.ink, fontFamily: F.body }}>
                    <IcoTarget size={13} style={{ color: C.types.hyrox.color, flexShrink: 0, marginTop: 2 }} /> {l}
                  </li>
                ))}
              </ul>
            </div>
          )}

          {proposal.justification && (
            <p className="text-xs italic" style={{ marginTop: 12, color: C.mute, fontFamily: F.body }}>
              {proposal.justification}
            </p>
          )}

          <div className="space-y-2" style={{ marginTop: 16 }}>
            <button onClick={planFromProposal}
              className="w-full rounded-lg text-sm"
              style={{ padding: "10px 0", background: C.iron, color: C.chalk, fontFamily: F.disp, fontWeight: 700, letterSpacing: "0.04em" }}>
              PLANIFIER CETTE SÉANCE
            </button>
            <div className="grid grid-cols-2 gap-2">
              <button onClick={prefillFromProposal}
                className="rounded-lg text-sm"
                style={{ padding: "10px 0", background: C.paper, color: C.ink, border: "1px solid " + C.line, fontFamily: F.disp, fontWeight: 700, letterSpacing: "0.04em" }}>
                PRÉ-REMPLIR LE JOURNAL
              </button>
              <button onClick={generateSession}
                className="rounded-lg text-sm"
                style={{ padding: "10px 0", background: C.paper, color: C.ink, border: "1px solid " + C.line, fontFamily: F.disp, fontWeight: 700, letterSpacing: "0.04em" }}>
                RÉGÉNÉRER
              </button>
            </div>
          </div>
        </Card>
      )}
    </div>
  );

  /* ---- Historique ---- */
  const HistoryView = () => (
    <div className="space-y-2">
      {sessions.length === 0 && (
        <Card>
          <p className="text-sm" style={{ color: C.mute }}>
            Historique vide. Tes séances enregistrées s'afficheront ici, avec leur impact sur les courbes de l'accueil.
          </p>
        </Card>
      )}
      {sessions.map((s) => {
        const t = C.types[s.type] || C.types.muscu;
        const meta = [
          s.duration && s.duration + " min",
          s.rpe && "RPE " + s.rpe,
          s.runKind,
          s.distance && s.distance + " km",
          paceStr(s.timeMin, s.distance),
          s.benchKg && "DC " + s.benchKg + "×" + s.benchReps + " (e1RM " + epley(s.benchKg, s.benchReps) + ")",
        ].filter(Boolean).join(" · ");
        return (
          <div key={s.id} className="rounded-xl"
            style={{ padding: 14, background: C.paper, border: "1px solid " + C.line, borderLeft: "3px solid " + t.color }}>
            <div className="flex items-start justify-between" style={{ gap: 8 }}>
              <div className="min-w-0">
                <div className="flex items-center flex-wrap" style={{ gap: 8 }}>
                  <span className="text-xs" style={{ fontFamily: F.mono, color: C.mute }}>{fmtFR(s.date)}</span>
                  <TypeBadge type={s.type} />
                  {s.planned && (
                    <span className="text-xs rounded" style={{ padding: "2px 6px", background: C.types.hyrox.bg, color: C.types.hyrox.color, fontFamily: F.disp, fontWeight: 700, letterSpacing: "0.06em" }}>
                      À FAIRE
                    </span>
                  )}
                </div>
                <div className="truncate" style={{ marginTop: 4, fontFamily: F.disp, fontWeight: 700, fontSize: 17, color: C.ink }}>
                  {s.title}
                </div>
                {meta && (
                  <div className="text-xs" style={{ marginTop: 2, fontFamily: F.mono, color: C.mute }}>{meta}</div>
                )}
                {s.stations && s.stations.length > 0 && (
                  <div className="text-xs" style={{ marginTop: 4, color: t.color, fontFamily: F.body }}>
                    {s.stations.join(" · ")}
                  </div>
                )}
                {s.notes && (
                  <p className="text-xs whitespace-pre-line" style={{ marginTop: 6, color: C.mute, fontFamily: F.body }}>
                    {s.notes}
                  </p>
                )}
              </div>
              <div className="flex items-center" style={{ gap: 4, flexShrink: 0 }}>
                {s.planned && (
                  <button onClick={() => markDone(s.id)}
                    className="rounded-lg text-xs flex items-center"
                    style={{ gap: 4, padding: "6px 8px", background: C.types.recup.bg, color: C.types.recup.color, fontFamily: F.body, fontWeight: 600 }}>
                    <IcoCheck size={14} /> Fait
                  </button>
                )}
                <button
                  onClick={() => (pendingDelete === s.id ? deleteSession(s.id) : setPendingDelete(s.id))}
                  className="rounded-lg text-xs flex items-center"
                  style={{
                    gap: 4, padding: "6px 8px",
                    color: pendingDelete === s.id ? "#FFFFFF" : C.mute,
                    background: pendingDelete === s.id ? C.types.muscu.color : "transparent",
                    fontFamily: F.body, fontWeight: 600,
                  }}>
                  <IcoTrash size={14} /> {pendingDelete === s.id ? "Confirmer" : ""}
                </button>
              </div>
            </div>
          </div>
        );
      })}
    </div>
  );

  /* ---- Réglages ---- */
  const SettingsModal = () => {
    const p = pDraft;
    const set = (k, v) => setPDraft((x) => ({ ...x, [k]: v }));
    const doExport = async () => {
      const json = JSON.stringify({ profile: p, sessions });
      setBackup(json);
      try {
        await navigator.clipboard.writeText(json);
        notify("Sauvegarde copiée dans le presse-papiers");
      } catch {
        notify("Copie auto impossible — sélectionne le texte affiché");
      }
    };
    const doImport = () => {
      try {
        const obj = JSON.parse(backup);
        if (!obj || !Array.isArray(obj.sessions)) throw new Error("format");
        const prof = { ...DEFAULT_PROFILE, ...(obj.profile || {}) };
        setProfile(prof); storeSet(PKEY, prof);
        setSessions(obj.sessions); storeSet(SKEY, obj.sessions);
        setPDraft(prof);
        notify("Import réussi : " + obj.sessions.length + " séances");
      } catch {
        notify("Import impossible : texte JSON invalide");
      }
    };
    const input = {
      className: "w-full px-3 rounded-lg text-base",
      style: { paddingTop: 10, paddingBottom: 10, background: C.chalk, border: "1px solid " + C.line, color: C.ink, fontFamily: F.body },
    };
    return (
      <div className="fixed inset-0 z-50 flex items-end sm:items-center justify-center"
        style={{ background: "rgba(23,25,27,0.55)" }} onClick={() => setSettingsOpen(false)}>
        <div className="w-full sm:max-w-md overflow-y-auto rounded-t-2xl sm:rounded-2xl p-5"
          style={{ maxHeight: "88vh", background: C.paper }} onClick={(e) => e.stopPropagation()}>
          <div className="flex items-center justify-between">
            <h2 style={{ fontFamily: F.disp, fontWeight: 700, fontSize: 22, color: C.ink }}>Profil & objectifs</h2>
            <button onClick={() => setSettingsOpen(false)}><IcoX size={20} style={{ color: C.mute }} /></button>
          </div>
          <div className="space-y-3" style={{ marginTop: 16 }}>
            <div className="grid grid-cols-2 gap-3">
              <div>
                <Eyebrow>Semi · date</Eyebrow>
                <input type="date" value={p.semiDate} onChange={(e) => set("semiDate", e.target.value)} {...input} />
              </div>
              <div>
                <Eyebrow>Objectif semi (min)</Eyebrow>
                <input inputMode="numeric" value={p.semiTargetMin}
                  onChange={(e) => set("semiTargetMin", parseFloat(e.target.value) || 0)} {...input} />
              </div>
            </div>
            <div className="grid grid-cols-2 gap-3">
              <div>
                <Eyebrow>Record semi (min)</Eyebrow>
                <input inputMode="numeric" value={p.semiPRMin}
                  onChange={(e) => set("semiPRMin", parseFloat(e.target.value) || 0)} {...input} />
              </div>
              <div>
                <Eyebrow>Hyrox · date</Eyebrow>
                <input type="date" value={p.hyroxDate} onChange={(e) => set("hyroxDate", e.target.value)} {...input} />
              </div>
            </div>
            <div>
              <Eyebrow>Compétition Hyrox</Eyebrow>
              <input value={p.hyroxLabel} onChange={(e) => set("hyroxLabel", e.target.value)} {...input} />
            </div>
            <div className="grid grid-cols-2 gap-3">
              <div>
                <Eyebrow>DC actuel (kg)</Eyebrow>
                <input inputMode="decimal" value={p.benchCurrent}
                  onChange={(e) => set("benchCurrent", parseFloat(e.target.value) || 0)} {...input} />
              </div>
              <div>
                <Eyebrow>DC cible (kg)</Eyebrow>
                <input inputMode="decimal" value={p.benchTarget}
                  onChange={(e) => set("benchTarget", parseFloat(e.target.value) || 0)} {...input} />
              </div>
            </div>
            <div className="grid grid-cols-2 gap-3">
              <div>
                <Eyebrow>Poids actuel (kg)</Eyebrow>
                <input inputMode="decimal" value={p.weightCurrent}
                  onChange={(e) => set("weightCurrent", parseFloat(e.target.value) || 0)} {...input} />
              </div>
              <div>
                <Eyebrow>Poids cible (kg)</Eyebrow>
                <input inputMode="decimal" value={p.weightTarget}
                  onChange={(e) => set("weightTarget", parseFloat(e.target.value) || 0)} {...input} />
              </div>
            </div>
            <div>
              <Eyebrow>Points faibles</Eyebrow>
              <input value={p.weakPoints} onChange={(e) => set("weakPoints", e.target.value)} {...input} />
            </div>
            <div>
              <Eyebrow>Contraintes (lues par le coach)</Eyebrow>
              <textarea rows={4} value={p.constraints} onChange={(e) => set("constraints", e.target.value)} {...input} />
            </div>
            <button onClick={() => saveProfile(p)}
              className="w-full rounded-xl"
              style={{ padding: "12px 0", background: C.iron, color: C.chalk, fontFamily: F.disp, fontWeight: 700, letterSpacing: "0.06em" }}>
              ENREGISTRER LE PROFIL
            </button>

            <div style={{ paddingTop: 12, borderTop: "1px solid " + C.line }}>
              <Eyebrow>Sauvegarde / restauration des données</Eyebrow>
              <p className="text-xs" style={{ marginTop: 4, color: C.mute, fontFamily: F.body }}>
                Exporter copie tout (profil + séances) en texte — garde-le dans Notes. Pour restaurer ou migrer : colle une sauvegarde ci-dessous puis Importer. Compatible avec la version Claude de l'app.
              </p>
              <textarea rows={3} value={backup} onChange={(e) => setBackup(e.target.value)}
                placeholder="La sauvegarde apparaîtra ici…"
                className="w-full px-3 rounded-lg text-xs"
                style={{ paddingTop: 8, paddingBottom: 8, marginTop: 8, background: C.chalk, border: "1px solid " + C.line, color: C.ink, fontFamily: F.mono }} />
              <div className="grid grid-cols-2 gap-2" style={{ marginTop: 8 }}>
                <button onClick={doExport}
                  className="rounded-lg text-sm"
                  style={{ padding: "10px 0", background: C.paper, color: C.ink, border: "1px solid " + C.line, fontFamily: F.disp, fontWeight: 700, letterSpacing: "0.04em" }}>
                  EXPORTER
                </button>
                <button onClick={doImport}
                  className="rounded-lg text-sm"
                  style={{ padding: "10px 0", background: C.paper, color: C.ink, border: "1px solid " + C.line, fontFamily: F.disp, fontWeight: 700, letterSpacing: "0.04em" }}>
                  IMPORTER
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>
    );
  };

  /* ---- Navigation ---- */
  const NAV = [
    { id: "home", label: "Accueil", Ico: IcoActivity },
    { id: "log", label: "Journal", Ico: IcoPlus },
    { id: "coach", label: "Coach", Ico: IcoSpark },
    { id: "history", label: "Historique", Ico: IcoHistory },
  ];

  return (
    <div className="min-h-screen" style={{ background: C.chalk, fontFamily: F.body }}>
      {/* Header */}
      <div className="sticky top-0 z-40 px-4 flex items-center justify-between"
        style={{ paddingTop: 16, paddingBottom: 12, background: C.chalk, borderBottom: "1px solid " + C.line }}>
        <div>
          <div style={{ fontFamily: F.disp, fontWeight: 700, fontSize: 22, color: C.ink, letterSpacing: "0.02em", lineHeight: 1 }}>
            COACH HYBRIDE
          </div>
          <div className="text-xs" style={{ marginTop: 2, fontFamily: F.mono, color: C.mute }}>
            SEMI J-{Math.max(daysUntil(profile.semiDate), 0)} · HYROX J-{Math.max(daysUntil(profile.hyroxDate), 0)}
          </div>
        </div>
        <button onClick={() => { setPDraft(profile); setBackup(""); setSettingsOpen(true); }} className="p-2 rounded-lg"
          style={{ border: "1px solid " + C.line, background: C.paper }}>
          <IcoGear size={18} style={{ color: C.ink }} />
        </button>
      </div>

      {/* Contenu */}
      <div className="px-4 py-4 max-w-md mx-auto" style={{ paddingBottom: 112 }}>
        {tab === "home" && Home()}
        {tab === "log" && Log()}
        {tab === "coach" && Coach()}
        {tab === "history" && HistoryView()}
      </div>

      {/* Toast */}
      {toast && (
        <div className="fixed z-50 rounded-full text-sm"
          style={{ left: "50%", transform: "translateX(-50%)", bottom: 96, padding: "8px 16px", background: C.iron, color: C.chalk, fontFamily: F.body, fontWeight: 500 }}>
          {toast}
        </div>
      )}

      {/* Barre d'onglets */}
      <div className="fixed bottom-0 left-0 right-0 z-40"
        style={{ background: C.iron, paddingBottom: "env(safe-area-inset-bottom)" }}>
        <div className="max-w-md mx-auto grid grid-cols-4">
          {NAV.map(({ id, label, Ico }) => {
            const on = tab === id;
            return (
              <button key={id} onClick={() => { setTab(id); setPendingDelete(null); }}
                className="flex flex-col items-center"
                style={{ gap: 4, padding: "12px 0" }}>
                <Ico size={20} style={{ color: on ? C.chalk : "#6E7378" }} />
                <span className="text-xs" style={{
                  fontFamily: F.disp, fontWeight: 700, letterSpacing: "0.08em",
                  color: on ? C.chalk : "#6E7378",
                }}>
                  {label.toUpperCase()}
                </span>
              </button>
            );
          })}
        </div>
      </div>

      {settingsOpen && SettingsModal()}
    </div>
  );
}

ReactDOM.createRoot(document.getElementById("root")).render(<CoachHybride />);
</script>
</body>
</html>
