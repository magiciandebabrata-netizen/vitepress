import React, { useEffect, useMemo, useState } from "react";

// ✅ PWA-ready single-file React app for Electrohomeopathy doctors
// Features
// - Pass Key required on first install / new device (stored per-device)
// - Disease library with Symptoms, Lab Diagnosis, Treatment
// - Editable entries; add / edit / delete
// - References: save Google (URL) or "From You" (notes) sources
// - One-tap Google search helper per disease
// - Persistent storage (localStorage)
// - Share/Backup: export JSON, import JSON, Web Share API support
// - Works offline when bundled as a PWA; mobile-friendly UI with Tailwind
// - Disclaimer banner (not a substitute for licensed medical care)

const STORAGE_KEY = "eh_app_data_v1";
const PASS_HASH_KEY = "eh_pass_hash_v1";
const DEVICE_ID_KEY = "eh_device_id_v1";

function uid() {
  return Math.random().toString(36).slice(2) + Date.now().toString(36);
}

async function sha256(text) {
  const enc = new TextEncoder();
  const data = enc.encode(text);
  const hash = await crypto.subtle.digest("SHA-256", data);
  return Array.from(new Uint8Array(hash))
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}

function loadData() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return null;
    return JSON.parse(raw);
  } catch {
    return null;
  }
}

function saveData(data) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
}

function initialSeed() {
  return {
    version: 1,
    clinic: { name: "Electrohomeopathy Practice", owner: "" },
    diseases: [
      {
        id: uid(),
        name: "Anaemia (General)",
        symptoms: ["Fatigue", "Pallor", "Shortness of breath", "Dizziness"],
        labTests: ["CBC (Hb, RBC indices)", "Peripheral smear", "Serum ferritin"],
        diagnosisNotes:
          "Suspect when Hb is low on CBC. Differentiate iron deficiency vs. other causes with indices & ferritin.",
        treatment: "[Example EH plan placeholder] Adjust based on your protocol.",
        references: [
          {
            id: uid(),
            kind: "google",
            label: "WHO anaemia overview",
            url: "https://www.google.com/search?q=WHO+anaemia+overview",
          },
          {
            id: uid(),
            kind: "note",
            label: "Clinic note",
            note:
              "Tailor Electrohomeopathy remedy per patient; add your dilution/remedy notes here.",
          },
        ],
      },
    ],
  };
}

// ---------- UI Helpers ----------
function Pill({ children }) {
  return (
    <span className="inline-block px-2 py-0.5 rounded-full border text-xs mr-1 mb-1">
      {children}
    </span>
  );
}

function ToolbarButton({ onClick, children, title }) {
  return (
    <button
      className="rounded-2xl border px-3 py-2 text-sm hover:shadow active:scale-[0.98]"
      onClick={onClick}
      title={title}
    >
      {children}
    </button>
  );
}

function Field({ label, children }) {
  return (
    <label className="block mb-3">
      <div className="text-sm font-medium mb-1">{label}</div>
      {children}
    </label>
  );
}

function TextInput(props) {
  return (
    <input
      {...props}
      className={`w-full border rounded-2xl px-3 py-2 focus:outline-none focus:ring ${
        props.className || ""
      }`}
    />
  );
}

function TextArea(props) {
  return (
    <textarea
      {...props}
      className={`w-full border rounded-2xl px-3 py-2 min-h-[90px] focus:outline-none focus:ring ${
        props.className || ""
      }`}
    />
  );
}

function SectionCard({ title, children, right }) {
  return (
    <div className="bg-white rounded-2xl shadow p-4 border">
      <div className="flex items-center justify-between mb-3">
        <h2 className="text-lg font-semibold">{title}</h2>
        <div>{right}</div>
      </div>
      {children}
    </div>
  );
}

function Disclaimer() {
  return (
    <div className="mb-4 text-xs text-gray-600 leading-snug">
      <strong>Disclaimer:</strong> This app is an informational tool to organize your
      Electrohomeopathy practice. It is not a substitute for licensed medical care.
      Always use clinical judgment and local regulations.
    </div>
  );
}

// ---------- Pass Key Gate ----------
function PassKeyGate({ onUnlocked }) {
  const [hasHash, setHasHash] = useState(() => !!localStorage.getItem(PASS_HASH_KEY));
  const [mode, setMode] = useState(() => (hasHash ? "unlock" : "create"));
  const [entry, setEntry] = useState("");
  const [error, setError] = useState("");

  useEffect(() => {
    if (!localStorage.getItem(DEVICE_ID_KEY)) {
      localStorage.setItem(DEVICE_ID_KEY, uid());
    }
  }, []);

  const handleCreate = async () => {
    setError("");
    if (entry.length < 4) {
      setError("Pass key must be at least 4 characters.");
      return;
    }
    const hash = await sha256(entry);
    localStorage.setItem(PASS_HASH_KEY, hash);
    setHasHash(true);
    setMode("unlock");
    setEntry("");
  };

  const handleUnlock = async () => {
    setError("");
    const saved = localStorage.getItem(PASS_HASH_KEY);
    const hash = await sha256(entry);
    if (saved && hash === saved) {
      onUnlocked();
    } else {
      setError("Incorrect pass key.");
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50 px-3">
      <div className="max-w-md w-full bg-white border shadow rounded-3xl p-6">
        <h1 className="text-2xl font-bold mb-1">Electrohomeopathy Doctor</h1>
        <p className="text-sm text-gray-600 mb-6">
          {mode === "create"
            ? "Create a pass key for this device/installation. You'll need it on each new device."
            : "Enter your pass key to unlock."}
        </p>
        <Field label={mode === "create" ? "Create Pass Key" : "Pass Key"}>
          <TextInput
            type="password"
            value={entry}
            onChange={(e) => setEntry(e.target.value)}
            placeholder={mode === "create" ? "Set a pass key" : "Enter pass key"}
          />
        </Field>
        {error && <div className="text-red-600 text-sm mb-3">{error}</div>}
        {mode === "create" ? (
          <button
            onClick={handleCreate}
            className="w-full rounded-2xl bg-black text-white py-2 font-medium"
          >
            Save Pass Key
          </button>
        ) : (
          <button
            onClick={handleUnlock}
            className="w-full rounded-2xl bg-black text-white py-2 font-medium"
          >
            Unlock
          </button>
        )}
        {hasHash && (
          <button
            onClick={() => setMode(mode === "unlock" ? "create" : "unlock")}
            className="mt-3 w-full text-sm underline"
          >
            {mode === "unlock" ? "Reset with new pass key" : "Back to unlock"}
          </button>
        )}
      </div>
    </div>
  );
}

// ---------- Disease Editor ----------
function DiseaseEditor({ value, onChange }) {
  const [item, setItem] = useState(() => ({ ...value }));
  useEffect(() => setItem({ ...value }), [value?.id]);

  const update = (patch) => {
    const next = { ...item, ...patch };
    setItem(next);
    onChange(next);
  };

  const [symptomText, setSymptomText] = useState("");
  const [labText, setLabText] = useState("");

  const addList = (key, textSetter, text) => {
    if (!text.trim()) return;
    update({ [key]: [...(item[key] || []), text.trim()] });
    textSetter("");
  };
  const removeList = (key, idx) => {
    const copy = [...(item[key] || [])];
    copy.splice(idx, 1);
    update({ [key]: copy });
  };

  const [refLabel, setRefLabel] = useState("");
  const [refUrl, setRefUrl] = useState("");
  const [refKind, setRefKind] = useState("google");
  const [refNote, setRefNote] = useState("");

  const addRef = () => {
    if (refKind === "google" && !refUrl) return;
    const newRef = {
      id: uid(),
      kind: refKind,
      label: refLabel || (refKind === "google" ? "Google ref" : "Note"),
      url: refKind === "google" ? refUrl : "",
      note: refKind === "note" ? refNote : "",
    };
    update({ references: [...(item.references || []), newRef] });
    setRefLabel("");
    setRefUrl("");
    setRefNote("");
  };

  const removeRef = (rid) => {
    update({ references: (item.references || []).filter((r) => r.id !== rid) });
  };

  const googleQuery = encodeURIComponent(
    `${item.name} symptoms lab diagnosis Electrohomeopathy treatment`
  );
  const googleUrl = `https://www.google.com/search?q=${googleQuery}`;

  return (
    <div className="grid md:grid-cols-2 gap-4">
      {/* Core Details */}
      <SectionCard title="Core Details">
        <Field label="Disease Name">
          <TextInput
            value={item.name}
            onChange={(e) => update({ name: e.target.value })}
          />
        </Field>
        <Field label="Diagnosis Notes">
          <TextArea
            value={item.diagnosisNotes}
            onChange={(e) => update({ diagnosisNotes: e.target.value })}
          />
        </Field>
        <Field label="Electrohomeopathy Treatment">
          <TextArea
            value={item.treatment}
            onChange={(e) => update({ treatment: e.target.value })}
          />
        </Field>
      </SectionCard>

      {/* Symptoms & Labs */}
      <SectionCard
        title="Symptoms & Lab Diagnosis"
        right={
          <a className="text-sm underline" href={googleUrl} target="_blank" rel="noreferrer">
            Search on Google ↗
          </a>
        }
      >
        {/* Symptoms */}
        <div className="mb-3">
          <div className="text-sm font-medium mb-1">Symptoms</div>
          <div className="flex gap-2 mb-2">
            <TextInput
              value={symptomText}
              onChange={(e) => setSymptomText(e.target.value)}
              placeholder="Add a symptom"
            />
            <ToolbarButton onClick={() => addList("symptoms", setSymptomText, symptomText)}>
              Add
            </ToolbarButton>
          </div>
          <div>
            {(item.symptoms || []).map((s, i) => (
              <span key={i} className="inline-flex items-center mr-2 mb-2">
                <Pill>{s}</Pill>
                <button className="ml-1 text-xs text-red-600" onClick={() => removeList("symptoms", i)}>
                  ×
                </button>
              </span>
            ))}
          </div>
        </div>

        {/* Labs */}
        <div>
          <div className="text-sm font-medium mb-1">Lab Tests</div>
          <div className="flex gap-2 mb-2">
            <TextInput
              value={labText}
              onChange={(e) => setLabText(e.target.value)}
              placeholder="Add a lab test (e.g., CBC)"
            />
            <ToolbarButton onClick={() => addList("labTests", setLabText, labText)}>Add</ToolbarButton>
          </div>
          <div>
            {(item.labTests || []).map((s, i) => (
              <span key={i} className="inline-flex items-center mr-2 mb-2">
                <Pill>{s}</Pill>
                <button className="ml-1 text-xs text-red-600" onClick={() => removeList("labTests", i)}>
                  ×
                </button>
              </span>
            ))}
          </div>
        </div>
      </SectionCard>

      {/* References */}
      <SectionCard title="References (Google & Notes)">
        <div className="space-y-3">
          {(item.references || []).map((r) => (
            <div key={r.id} className="flex items-center justify-between border rounded-2xl p-2">
              <div className="text-sm">
                <div className="font-medium">
                  {r.label}{" "}
                  {r.kind === "google" ? (
                    <span className="text-xs">(Google)</span>
                  ) : (
                    <span className="text-xs">(From You)</span>
                  )}
                </div>
                {r.kind === "google" ? (
                  <a className="underline break-all" href={r.url} target="_blank" rel="noreferrer">
                    {r.url}
                  </a>
                ) : (
                  <div className="text-gray-700 whitespace-pre-wrap">{r.note}</div>
                )}
              </div>
              <button className="text-sm text-red-600" onClick={() => removeRef(r.id)}>
                Delete
              </button>
            </div>
          ))}

          <div className="grid md:grid-cols-3 gap-2">
            <select
              value={refKind}
              onChange={(e) => setRefKind(e.target.value)}
              className="border rounded-2xl px-3 py-2"
            >
              <option value="google">Google Reference (URL)</option>
              <option value="note">From You (Note)</option>
            </select>
            <TextInput
              placeholder="Label"
              value={refLabel}
              onChange={(e) => setRefLabel(e.target.value)}
            />
            {refKind === "google" ? (
              <TextInput
                placeholder="https://..."
                value={refUrl}
                onChange={(e) => setRefUrl(e.target.value)}
              />
            ) : (
              <TextInput
                placeholder="Short note"
                value={refNote}
                onChange={(e) => setRefNote(e.target.value)}
              />
            )}
          </div>
          <div>
            <ToolbarButton onClick={addRef}>Add Reference</ToolbarButton>
            <a className="ml-2 text-sm underline" href={googleUrl} target="_blank" rel="noreferrer">
              Find references on Google ↗
            </a>
          </div>
        </div>
      </SectionCard>
    </div>
  );
}

// ---------- Main App ----------
export default function App() {
  const [unlocked, setUnlocked] = useState(false);
  const [data, setData] = useState(() => loadData() || initialSeed());
  const [query, setQuery] = useState("");
  const [editing, setEditing] = useState(null);

  useEffect(() => saveData(data), [data]);

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return data.diseases;
    return data.diseases.filter((d) =>
      [d.name, d.diagnosisNotes, d.treatment, ...(d.symptoms || []), ...(d.labTests || [])]
        .join(" ")
        .toLowerCase()
        .includes(q)
    );
  }, [query, data.diseases]);

  const addNew = () => {
    const n = { id: uid(), name: "New Disease", symptoms: [], labTests: [], diagnosisNotes: "", treatment: "", references: [] };
    setData({ ...data, diseases: [n, ...data.diseases] });
    setEditing(n);
  };
  const updateDisease = (next) => setData({ ...data, diseases: data.diseases.map((d) => (d.id === next.id ? next : d)) });
  const deleteDisease = (id) => {
    if (confirm("Delete this disease?")) {
      setData({ ...data, diseases: data.diseases.filter((d) => d.id !== id) });
      setEditing(null);
    }
  };

  const exportJson = () => {
    const blob = new Blob([JSON.stringify(data, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `EH-doctor-data-${new Date().toISOString().slice(0, 10)}.json`;
    a.click();
    URL.revokeObjectURL(url);
  };

  const importJson = () => {
    const inp = document.createElement("input");
    inp.type = "file";
    inp.accept = "application/json";
    inp.onchange = async () => {
      const file = inp.files?.[0];
      if (!file) return;
      const text = await file.text();
      try {
        const parsed = JSON.parse(text);
        if (!parsed?.diseases) throw new Error("Invalid file");
        setData(parsed);
        alert("Import successful.");
      } catch (e) {
        alert("Import


- 📖 [**Documentation & guides**](https://vite-pwa-org.netlify.app/)
- 👌 **Zero-Config**: sensible built-in default configs for common use cases
- 🔩 **Extensible**: expose the full ability to customize the behavior of the plugin
- 🦾 **Type Strong**: written in [TypeScript](https://www.typescriptlang.org/)
- 🔌 **Offline Support**: generate service worker with offline support (via Workbox)
- ⚡ **Fully tree shakable**: auto inject Web App Manifest
- 💬 **Prompt for new content**: built-in support for Vanilla JavaScript, Vue 3, React, Svelte, SolidJS and Preact
- ⚙️ **Stale-while-revalidate**: automatic reload when new content is available
- ✨ **Static assets handling**: configure static assets for offline support
- 🐞 **Development Support**: debug your custom service worker logic as you develop your application
- 🛠️ **Versatile**: integration with meta frameworks: [îles](https://github.com/ElMassimo/iles), [SvelteKit](https://github.com/sveltejs/kit), [VitePress](https://github.com/vuejs/vitepress), [Astro](https://github.com/withastro/astro), [Nuxt 3](https://github.com/nuxt/nuxt) and [Remix](https://github.com/remix-run/remix)
- 💥 **PWA Assets Generator**: generate all the PWA assets from a single command and a single source image
- 🚀 **PWA Assets Integration**: serving, generating and injecting PWA Assets on the fly in your application

## 📦 Install

> From v0.3, `@vite-pwa/vitepress` requires **Vite 5** and **VitePress 1.0.0-rc.26 or above**.

> Using any version older than v0.3 requires Vite 3.1.0+.

```bash
npm i @vite-pwa/vitepress -D

# yarn
yarn add @vite-pwa/vitepress -D

# pnpm
pnpm add @vite-pwa/vitepress -D
```

## 🦄 Usage

You will need to wrap your VitePress config with `withPwa`:

```ts
// .vitepress/config.ts
import { withPwa } from '@vite-pwa/vitepress'
import { defineConfig } from 'vitepress'

export default withPwa(defineConfig({
  /* your VitePress options */
  /* Vite PWA Options */
  pwa: {}
}))
```

Read the [📖 documentation](https://vite-pwa-org.netlify.app/frameworks/vitepress) for a complete guide on how to configure and use
this plugin.

## 👀 Full config

Check out the type declaration [src/types.ts](./src/types.ts) and the following links for more details.

- [Web app manifests](https://developer.mozilla.org/en-US/docs/Web/Manifest)
- [Workbox](https://developers.google.com/web/tools/workbox)

## 📄 License

[MIT](./LICENSE) License &copy; 2022-PRESENT [Anthony Fu](https://github.com/antfu)
