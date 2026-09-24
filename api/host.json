const { app } = require("@azure/functions");
const { TableClient, odata } = require("@azure/data-tables");

const LIMITS = { weightKg: [20, 300], steps: [0, 100000], waterL: [0, 10], sleepHrs: [0, 24], calories: [0, 10000], heartRate: [30, 220] };
const NOTES = ["Morning run", "Gym workout", "Rest day", "Regular day", "Cardio", "Light workout", "Cycling"];
const ACTIVE = ["Morning run", "Gym workout", "Cardio", "Cycling"];
const DEMO = [ // id, name, age, height, goal, baseline, days
  ["alex", "Alex", 21, 175, "Maintain fitness", { w: 68, s: 8000, wa: 2.5, sl: 7.0, c: 2100, hr: 71 }, 30],
  ["sarah", "Sarah", 22, 162, "Improve sleep and hydration", { w: 56, s: 7500, wa: 2.2, sl: 6.8, c: 1900, hr: 74 }, 14],
  ["rahul", "Rahul", 24, 178, "Lose weight", { w: 82, s: 6500, wa: 2.4, sl: 6.6, c: 2400, hr: 76 }, 30],
];
const r1 = (x) => Math.round(x * 10) / 10;
const rnd = (a, b) => a + Math.random() * (b - a);
const clip = (v, lo, hi) => Math.max(lo, Math.min(hi, v));
const day = (offset) => new Date(Date.now() - offset * 864e5).toISOString().slice(0, 10);

// ---- tables (created automatically on first use) ----
let ready;
function tables() {
  const cs = process.env.STORAGE_CONNECTION;
  if (!cs) throw new Error("STORAGE_CONNECTION setting is missing");
  const P = TableClient.fromConnectionString(cs, "Profiles");
  const R = TableClient.fromConnectionString(cs, "HealthRecords");
  ready = ready || Promise.all([P.createTable(), R.createTable()]);
  return ready.then(() => ({ P, R }));
}

// ---- identity: Static Web Apps puts the logged-in user in this header ----
function userId(req) {
  const h = req.headers.get("x-ms-client-principal");
  if (!h) return null;
  return JSON.parse(Buffer.from(h, "base64").toString("utf8")).userId || null;
}

function route(name, methods, path, fn) {
  app.http(name, {
    methods, route: path, authLevel: "anonymous",
    handler: async (req, ctx) => {
      try {
        const u = userId(req);
        if (!u) return { status: 401, jsonBody: { error: "Not signed in" } };
        return await fn(req, u, await tables());
      } catch (e) {
        ctx.error(e);
        return { status: 500, jsonBody: { error: e.message } };
      }
    },
  });
}

async function owns(P, u, pid) {
  if (!pid) return false;
  try { await P.getEntity(u, pid); return true; } catch { return false; }
}

const out = (e) => ({ recordDate: e.rowKey, weightKg: e.weightKg, steps: e.steps, waterL: e.waterL,
  sleepHrs: e.sleepHrs, calories: e.calories, heartRate: e.heartRate, notes: e.notes || "" });

async function seed(u, { P, R }) {
  const jobs = [];
  for (const [pid, name, age, h, goal, b, days] of DEMO) {
    jobs.push(P.upsertEntity({ partitionKey: u, rowKey: pid, name, age, heightCm: h, goal, weightKg: b.w }));
    for (let i = 0; i < days; i++) {
      const note = NOTES[Math.floor(Math.random() * NOTES.length)], act = ACTIVE.includes(note);
      jobs.push(R.upsertEntity({
        partitionKey: `${u}_${pid}`, rowKey: day(days - 1 - i),
        weightKg: r1(b.w - i * 0.02 + rnd(-0.3, 0.3)),
        steps: Math.round(b.s * rnd(0.85, 1.15) * (act ? 1.15 : 0.9)),
        waterL: r1(clip(b.wa + rnd(-0.4, 0.4), 1.5, 4)),
        sleepHrs: r1(clip(b.sl + rnd(-0.6, 0.6), 5, 9)),
        calories: Math.round(b.c * rnd(0.93, 1.07) * (act ? 1.05 : 0.97)),
        heartRate: b.hr + Math.floor(rnd(-3, 4)), notes: note }));
    }
  }
  await Promise.all(jobs);
}

route("getProfiles", ["GET"], "profiles", async (req, u, t) => {
  const list = async () => {
    const a = [];
    for await (const e of t.P.listEntities({ queryOptions: { filter: odata`PartitionKey eq ${u}` } }))
      a.push({ profileId: e.rowKey, name: e.name, age: e.age, heightCm: e.heightCm, goal: e.goal, weightKg: e.weightKg });
    return a;
  };
  let a = await list();
  if (!a.length) { await seed(u, t); a = await list(); }   // first login: create demo profiles
  return { jsonBody: a };
});

route("getRecords", ["GET"], "records", async (req, u, t) => {
  const pid = req.query.get("profile");
  if (!(await owns(t.P, u, pid))) return { status: 403, jsonBody: { error: "Forbidden" } };
  const from = day(Math.min(365, Number(req.query.get("days")) || 30) - 1);
  const a = [];
  for await (const e of t.R.listEntities({ queryOptions: { filter: odata`PartitionKey eq ${u + "_" + pid} and RowKey ge ${from}` } }))
    a.push(out(e));
  return { jsonBody: a };
});

route("saveRecord", ["POST"], "records", async (req, u, t) => {
  const b = await req.json();
  for (const [k, [lo, hi]] of Object.entries(LIMITS)) {
    const v = Number(b[k]);
    if (b[k] === "" || b[k] == null || isNaN(v) || v < lo || v > hi)
      return { status: 400, jsonBody: { error: `${k} must be between ${lo} and ${hi}` } };
  }
  if (!/^\d{4}-\d{2}-\d{2}$/.test(b.recordDate || "") || b.recordDate > day(-1))
    return { status: 400, jsonBody: { error: "recordDate must be a valid date, not in the future" } };
  if (!(await owns(t.P, u, b.profileId))) return { status: 403, jsonBody: { error: "Forbidden" } };
  const e = { partitionKey: `${u}_${b.profileId}`, rowKey: b.recordDate, notes: String(b.notes || "").slice(0, 100) };
  for (const k of Object.keys(LIMITS)) e[k] = Number(b[k]);
  await t.R.upsertEntity(e);
  return { jsonBody: out(e) };
});

route("deleteRecord", ["DELETE"], "records/{date}", async (req, u, t) => {
  const pid = req.query.get("profile");
  if (!(await owns(t.P, u, pid))) return { status: 403, jsonBody: { error: "Forbidden" } };
  try { await t.R.deleteEntity(`${u}_${pid}`, req.params.date); } catch {}
  return { jsonBody: { deleted: true } };
});
