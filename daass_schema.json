
import streamlit as st
import json, time, math, random
from pathlib import Path

st.set_page_config(page_title="DAASS — Digital Adaptive Autism Spectrum Screening", page_icon="🧭", layout="centered")

@st.cache_data(show_spinner=False)
def load_schema():
    with open("daass_schema.json", "r", encoding="utf-8") as f:
        return json.load(f)

schema = load_schema()

CORE_IDS = [i["id"] for i in schema["items"] if i["id"].startswith("Q") and i["domain"] in ["attention_focus","communication_context","sensory","social_emotional","routines"] and "mirror_of" not in i]
ADAPTIVE_IDS = [i["id"] for i in schema["items"] if i.get("mirror_of")]
INTEGRITY_IDS = [i["id"] for i in schema["items"] if i["domain"] == "integrity"]

ITEM_MAP = {i["id"]: i for i in schema["items"]}

def init_state():
    if "phase" not in st.session_state:
        st.session_state.phase = "intro"
        random.seed()
        st.session_state.order_core = random.sample(CORE_IDS, k=len(CORE_IDS))
        st.session_state.order_integrity = random.sample(INTEGRITY_IDS, k=len(INTEGRITY_IDS))
        st.session_state.order_adaptive = []
        st.session_state.idx = 0
        st.session_state.responses = {}
        st.session_state.times = {}
        st.session_state._t0 = None

def start_timer():
    st.session_state._t0 = time.perf_counter()

def stop_timer(item_id):
    t0 = st.session_state.get("_t0", None)
    if t0 is not None:
        dt_ms = int((time.perf_counter() - t0) * 1000)
        st.session_state.times[item_id] = st.session_state.times.get(item_id, 0) + dt_ms
    st.session_state._t0 = None

def render_item(item_id):
    item = ITEM_MAP[item_id]
    st.write(f"**{item['text']}**")
    val = st.session_state.responses.get(item_id, None)
    choice = st.radio("Select one:", options=[1,2,3,4,5], format_func=lambda x: {1:"Strongly disagree",2:"Disagree",3:"Neutral",4:"Agree",5:"Strongly agree"}[x], index=(val-1) if val else None, key=f"radio_{item_id}")
    st.session_state.responses[item_id] = choice

def compute_adaptive_list():
    r = st.session_state.responses
    def get(id): return r.get(id, 3)
    adaptive = []
    if get("Q1") + get("Q3") >= 8: adaptive.append("Q1a")
    if get("Q11") + get("Q12") >= 8: adaptive.append("Q11a")
    if get("Q21") + get("Q22") >= 8: adaptive.append("Q22a")
    if "Q7" in r: adaptive.append("Q7a")
    adaptive = [a for a in adaptive if a in ADAPTIVE_IDS]
    random.shuffle(adaptive)
    return adaptive

def score_all():
    s = schema
    r = st.session_state.responses
    times = st.session_state.times

    def score_item(item_id):
        item = ITEM_MAP[item_id]
        raw = r.get(item_id)
        if raw is None: return None
        if item.get("reverse", False):
            return 6 - raw
        return raw

    domains = {d: [] for d in s["scoring"]["domain_mapping"].keys()}
    for domain, ids in s["scoring"]["domain_mapping"].items():
        for qid in ids:
            if qid in r:
                sc = score_item(qid)
                if sc is not None:
                    domains[domain].append(sc)

    domain_max = s["scoring"]["computation"]["domain_max"]
    domain_scores = {}
    for d, vals in domains.items():
        if not vals:
            domain_scores[d] = 0.0
            continue
        max_per_item = 5.0
        domain_scores[d] = domain_max * (sum(vals) / (max_per_item * len(vals)))

    total_index = sum(domain_scores.values())

    conf = 1.0
    integ = s["scoring"]["integrity_logic"]["penalize_extremes"]
    penalty_per_extreme = float(integ["penalty_per_extreme"])
    extreme_values = set(integ["extreme_values"])
    extreme_count = 0
    for iid in s["scoring"]["integrity_items"]:
        v = r.get(iid)
        if v in extreme_values:
            extreme_count += 1
    conf -= penalty_per_extreme * extreme_count

    tlogic = s["scoring"]["integrity_logic"]["time_based_flags"]
    min_ms = int(tlogic["min_ms_per_item"])
    penalty_time = float(tlogic["penalty"])
    too_fast = [iid for iid, t in times.items() if t < min_ms]
    if len(times) > 0 and len(too_fast) / len(times) >= float(tlogic["rapid_threshold_pct"]):
        conf -= penalty_time

    mirrors = s["scoring"]["consistency_checks"]["mirrors"]
    allowable = int(s["scoring"]["consistency_checks"]["allowable_deviation"])
    per_violation = float(s["scoring"]["consistency_checks"]["penalty_per_violation"])
    violations = 0
    for a,b in mirrors:
        if a in r and b in r:
            if abs(r[a] - r[b]) > allowable:
                violations += 1
    conf = max(0.0, min(1.0, 1.0 - per_violation * violations + (conf - 1.0)))

    label = "Low"
    for lvl in s["scoring"]["computation"]["confidence_levels"]:
        if conf >= lvl["min_conf"]:
            label = lvl["label"]
            break

    cut_label = "Unscored"
    for cut in s["scoring"]["cutpoints"]:
        lo, hi = cut["range"]
        if total_index >= lo and total_index <= hi:
            cut_label = cut["label"]
            break

    return {
        "domain_scores": domain_scores,
        "total_index": round(total_index,2),
        "confidence": round(conf,2),
        "confidence_label": label,
        "class_label": cut_label,
        "extreme_count": extreme_count,
        "consistency_violations": violations,
        "too_fast_count": len(too_fast),
        "answered": len(r)
    }

init_state()

st.title("DAASS — Digital Adaptive Autism Spectrum Screening")
st.caption("Self-administered ASD-trait screening. This is *not* a diagnosis.")

if st.session_state.phase == "intro":
    with st.expander("What this does (tap to expand)", expanded=False):
        st.markdown(
            "- Randomized core items, adaptive follow-ups, and integrity checks\n"
            "- Per-item timing for response integrity\n"
            "- Automatic scoring with confidence weighting\n"
        )
    if st.button("Start"):
        st.session_state.phase = "core"
        st.session_state.idx = 0
        start_timer()
    st.stop()

def next_item(curr_id):
    stop_timer(curr_id)
    st.session_state.idx += 1
    start_timer()

def prev_item():
    st.session_state.idx = max(0, st.session_state.idx - 1)
    start_timer()

if st.session_state.phase == "core":
    order = st.session_state.order_core
    if st.session_state.idx >= len(order):
        st.session_state.order_adaptive = compute_adaptive_list()
        st.session_state.phase = "adaptive" if st.session_state.order_adaptive else "integrity"
        st.session_state.idx = 0
        start_timer()
        st.rerun()

    curr_id = order[st.session_state.idx]
    st.subheader(f"Core item {st.session_state.idx+1} of {len(order)}")
    render_item(curr_id)
    cols = st.columns(2)
    with cols[0]:
        if st.session_state.idx > 0 and st.button("Back"):
            prev_item()
            st.rerun()
    with cols[1]:
        if st.button("Next ▶"):
            if curr_id not in st.session_state.responses:
                st.warning("Please select a response to continue.")
            else:
                next_item(curr_id)
                st.rerun()
    st.stop()

if st.session_state.phase == "adaptive":
    order = st.session_state.order_adaptive
    if st.session_state.idx >= len(order):
        st.session_state.phase = "integrity"
        st.session_state.idx = 0
        start_timer()
        st.rerun()

    curr_id = order[st.session_state.idx]
    st.subheader(f"Adaptive item {st.session_state.idx+1} of {len(order)}")
    render_item(curr_id)
    cols = st.columns(2)
    with cols[0]:
        if st.session_state.idx > 0 and st.button("Back"):
            prev_item()
            st.rerun()
    with cols[1]:
        if st.button("Next ▶"):
            next_item(curr_id)
            st.rerun()
    st.stop()

if st.session_state.phase == "integrity":
    order = st.session_state.order_integrity
    if st.session_state.idx >= len(order):
        st.session_state.phase = "results"
        st.rerun()
    else:
        curr_id = order[st.session_state.idx]
        st.subheader(f"Integrity item {st.session_state.idx+1} of {len(order)}")
        render_item(curr_id)
        if st.button("Next ▶"):
            next_item(curr_id)
            st.rerun()
    st.stop()

if st.session_state.phase == "results":
    results = score_all()
    st.success("Completed. Here are your results:")
    st.metric("Total ASD-likelihood Index (0–100)", results["total_index"])
    st.metric("Confidence", f"{results['confidence']} ({results['confidence_label']})")
    st.metric("Class", results["class_label"])

    st.subheader("Domain scores (0–20 each)")
    for d, v in results["domain_scores"].items():
        st.write(f"- **{d.replace('_',' ').title()}**: {v:.2f}")

    with st.expander("Integrity diagnostics"):
        st.write(f"- Extreme-response count (integrity items): {results['extreme_count']}")
        st.write(f"- Consistency violations (mirrors): {results['consistency_violations']}")
        st.write(f"- Too-fast responses (< {schema['scoring']['integrity_logic']['time_based_flags']['min_ms_per_item']} ms): {results['too_fast_count']}")
        st.write(f"- Items answered: {results['answered']}")

    st.caption("This tool is a screening instrument and does not provide a diagnosis. Consider consulting a qualified clinician for further evaluation.")
    if st.button("Restart"):
        for k in list(st.session_state.keys()):
            del st.session_state[k]
        init_state()
        st.rerun()
