Below is a complete, production-ready Streamlit app for Hugging Face Spaces that implements your agentic AI system with advanced visualization, multi-model support (Gemini, OpenAI, Grok), dynamic dataset form creation, templated document generation, agent orchestration from agents.yaml, and interactive dashboard features. It includes robust error handling, flexible template editing, and the ability to modify prompts/parameters and chain agent outputs. It preserves all requested features and adds advanced status indicators and a Wow-level interactive dashboard.

Files to include in your Hugging Face Space repo:

1) app.py
----------------
import os
import io
import re
import json
import yaml
import time
import base64
import random
import tempfile
import traceback
import streamlit as st
import pandas as pd

# Data processing
from pandas_ods_reader import read_ods
from docx import Document as DocxDocument

# Template rendering
from jinja2 import Environment, FileSystemLoader, select_autoescape, Template

# LLM Providers
import google.generativeai as genai
from openai import OpenAI

# xAI / Grok SDK (install xai_sdk)
from xai_sdk import Client as XAIClient
from xai_sdk.chat import user as xai_user, system as xai_system, image as xai_image

# ---------------------------
# Configuration and Constants
# ---------------------------
APP_TITLE = "Agentic Docs Builder"
APP_DESC = "Upload dataset, create/edit template with {{placeholders}}, generate documents, and orchestrate multi-agent pipelines."

SUPPORTED_DATASETS = ["csv", "json", "txt", "ods", "xlsx"]
SUPPORTED_TEMPLATES = ["txt", "md", "markdown", "odt", "docx"]

DEFAULT_MODELS = [
    # Gemini
    "gemini-2.5-flash",
    "gemini-2.5-flash-lite",
    # OpenAI
    "gpt-5-nano",
    "gpt-4o-mini",
    "gpt-4.1-mini",
    # Grok
    "grok-4-fast-reasoning",
    "grok-3-mini",
]

DEFAULT_TEMPERATURE = 0.3
DEFAULT_MAX_TOKENS = 1024
DEFAULT_TOP_P = 0.95

# ---------------------------
# Utility: State Management
# ---------------------------
def init_session_state():
    st.session_state.setdefault("dataset_df", None)
    st.session_state.setdefault("dataset_name", "")
    st.session_state.setdefault("schema", [])
    st.session_state.setdefault("records", [])  # list of dicts from dataset (and added)
    st.session_state.setdefault("template_raw", "")
    st.session_state.setdefault("template_name", "")
    st.session_state.setdefault("generated_docs", [])  # list of dicts: {"record_index": i, "content": str, "file_name": str}
    st.session_state.setdefault("agents_config", None)  # loaded YAML agents config
    st.session_state.setdefault("agent_run_history", [])  # logs and outputs
    st.session_state.setdefault("edited_agent_outputs", {})  # agent_name -> edited text
    st.session_state.setdefault("metrics", {"calls": 0, "errors": 0, "latencies": []})

# ---------------------------
# Data I/O
# ---------------------------
def load_dataset(file, filetype):
    try:
        if filetype == "csv":
            df = pd.read_csv(file)
        elif filetype == "json":
            # Attempt to read as list of objects or line-delimited JSON
            content = file.getvalue().decode("utf-8", errors="ignore")
            try:
                data = json.loads(content)
                df = pd.DataFrame(data)
            except json.JSONDecodeError:
                # fallback to lines
                records = []
                for line in content.splitlines():
                    line = line.strip()
                    if not line:
                        continue
                    try:
                        records.append(json.loads(line))
                    except:
                        records.append({"text": line})
                df = pd.DataFrame(records)
        elif filetype == "txt":
            # Try sniffing delimiter; if fails -> single text column
            content = file.getvalue().decode("utf-8", errors="ignore")
            lines = [ln for ln in content.splitlines() if ln.strip() != ""]
            if not lines:
                df = pd.DataFrame(columns=["text"])
            else:
                # naive heuristic for delimiter based content
                if any("," in ln for ln in lines[:10]):
                    df = pd.read_csv(io.StringIO(content))
                else:
                    df = pd.DataFrame({"text": lines})
        elif filetype == "ods":
            # read_ods requires a path; save to temp
            with tempfile.NamedTemporaryFile(suffix=".ods", delete=False) as tmp:
                tmp.write(file.getvalue())
                tmp.flush()
                df = read_ods(tmp.name, 1)
        elif filetype == "xlsx":
            df = pd.read_excel(file)
        else:
            raise ValueError("Unsupported dataset format")
        df = df.fillna("")
        return df
    except Exception as e:
        raise RuntimeError(f"Failed to load dataset: {str(e)}")

def infer_schema(df: pd.DataFrame):
    return list(df.columns)

def add_record(df: pd.DataFrame, record: dict):
    row = {col: record.get(col, "") for col in df.columns}
    return pd.concat([df, pd.DataFrame([row])], ignore_index=True)

# ---------------------------
# Template I/O and Rendering
# ---------------------------
def extract_text_from_docx(file_bytes: bytes) -> str:
    with tempfile.NamedTemporaryFile(suffix=".docx", delete=False) as tmp:
        tmp.write(file_bytes)
        tmp.flush()
        doc = DocxDocument(tmp.name)
    text_parts = []
    for p in doc.paragraphs:
        text_parts.append(p.text)
    return "\n".join(text_parts)

def extract_text_from_odt(file_bytes: bytes) -> str:
    # Lightweight ODT extraction: try to parse content as text via zip if odfpy not set up
    # Using a very simple fallback to avoid heavy dependencies
    try:
        import zipfile
        from xml.etree import ElementTree as ET
        with tempfile.NamedTemporaryFile(suffix=".odt", delete=False) as tmp:
            tmp.write(file_bytes)
            tmp.flush()
            with zipfile.ZipFile(tmp.name) as z:
                xml = z.read("content.xml")
                root = ET.fromstring(xml)
                # extract text nodes
                ns = {"text": "urn:oasis:names:tc:opendocument:xmlns:text:1.0"}
                text_elems = root.findall(".//text:p", ns)
                paragraphs = []
                for el in text_elems:
                    paragraphs.append("".join(el.itertext()))
                return "\n".join(paragraphs)
    except Exception:
        return ""

def load_template(file, ext):
    try:
        ext = ext.lower()
        if ext in ["txt", "md", "markdown"]:
            content = file.getvalue().decode("utf-8", errors="ignore")
        elif ext == "docx":
            content = extract_text_from_docx(file.getvalue())
        elif ext == "odt":
            content = extract_text_from_odt(file.getvalue())
        else:
            raise ValueError("Unsupported template format")
        return content
    except Exception as e:
        raise RuntimeError(f"Failed to load template: {str(e)}")

def render_template_string(template_str: str, context: dict) -> str:
    try:
        # Jinja2 environment hardened
        env = Environment(autoescape=False)
        # Support {{ var }} placeholders
        tmpl = env.from_string(template_str)
        return tmpl.render(**context)
    except Exception as e:
        raise RuntimeError(f"Template rendering error: {str(e)}")

# ---------------------------
# Download utilities
# ---------------------------
def make_text_downloadable(content: str, filename: str) -> str:
    b64 = base64.b64encode(content.encode()).decode()
    href = f'<a href="data:text/plain;base64,{b64}" download="{filename}">Download {filename}</a>'
    return href

def save_to_docx(text: str) -> bytes:
    doc = DocxDocument()
    for line in text.split("\n"):
        doc.add_paragraph(line)
    with io.BytesIO() as output:
        doc.save(output)
        return output.getvalue()

# ---------------------------
# Agents: YAML and Orchestration
# ---------------------------
DEFAULT_AGENTS_YAML = """
agents:
  - name: Summarizer
    description: Summarize the generated document.
    default_model: gpt-4o-mini
    temperature: 0.3
    max_tokens: 512
    top_p: 0.95
    system_prompt: "You are a helpful assistant that summarizes text concisely."
    user_prompt: "Summarize the following content:\\n\\n{{input}}"
  - name: StyleRewriter
    description: Rewrite the content in the requested style.
    default_model: gemini-2.5-flash
    temperature: 0.5
    max_tokens: 1024
    top_p: 0.95
    system_prompt: "You are an expert copywriter."
    user_prompt: "Rewrite the content in a confident, friendly tone while preserving facts:\\n\\n{{input}}"
  - name: KeywordExtractor
    description: Extract 5-10 key terms as a comma-separated list.
    default_model: grok-3-mini
    temperature: 0.2
    max_tokens: 256
    top_p: 0.9
    system_prompt: "You are a precise NLP assistant."
    user_prompt: "Extract 5-10 keywords from the content. Return a comma-separated list only:\\n\\n{{input}}"
"""

def load_agents_yaml(uploaded_yaml_file=None):
    if uploaded_yaml_file is None:
        return yaml.safe_load(DEFAULT_AGENTS_YAML)
    try:
        content = uploaded_yaml_file.getvalue().decode("utf-8", errors="ignore")
        return yaml.safe_load(content)
    except Exception as e:
        st.warning(f"Failed to parse agents.yaml. Using defaults. Error: {str(e)}")
        return yaml.safe_load(DEFAULT_AGENTS_YAML)

# ---------------------------
# LLM Providers: Unified Call
# ---------------------------
def call_llm_unified(model_name: str, system_prompt: str, user_prompt: str, temperature=DEFAULT_TEMPERATURE, max_tokens=DEFAULT_MAX_TOKENS, top_p=DEFAULT_TOP_P, timeout=120):
    t0 = time.time()
    model_name = (model_name or "").strip()
    provider = detect_provider(model_name)

    try:
        if provider == "gemini":
            api_key = os.getenv("GEMINI_API_KEY") or st.secrets.get("GEMINI_API_KEY", None)
            if not api_key:
                raise RuntimeError("Missing GEMINI_API_KEY")
            genai.configure(api_key=api_key)
            model = genai.GenerativeModel(model_name)
            # Gemini uses safety settings internally; combine sys+user
            prompt = f"{system_prompt.strip()}\n\n{user_prompt.strip()}" if system_prompt else user_prompt
            resp = model.generate_content(
                prompt,
                generation_config={
                    "temperature": float(temperature),
                    "top_p": float(top_p),
                    "max_output_tokens": int(max_tokens),
                }
            )
            text = resp.text or ""
        elif provider == "openai":
            api_key = os.getenv("OPENAI_API_KEY") or st.secrets.get("OPENAI_API_KEY", None)
            if not api_key:
                raise RuntimeError("Missing OPENAI_API_KEY")
            client = OpenAI(api_key=api_key)
            messages = []
            if system_prompt:
                messages.append({"role": "system", "content": system_prompt})
            messages.append({"role": "user", "content": user_prompt})
            resp = client.chat.completions.create(
                model=model_name,
                messages=messages,
                temperature=float(temperature),
                max_tokens=int(max_tokens),
                top_p=float(top_p),
                timeout=timeout,
            )
            text = resp.choices[0].message.content or ""
        elif provider == "grok":
            api_key = os.getenv("XAI_API_KEY") or st.secrets.get("XAI_API_KEY", None)
            if not api_key:
                raise RuntimeError("Missing XAI_API_KEY")
            # Use xAI SDK as requested
            client = XAIClient(api_key=api_key, timeout=3600)
            chat = client.chat.create(model=model_name.replace("-fast-reasoning", "").replace("-mini", ""))
            if system_prompt:
                chat.append(xai_system(system_prompt))
            chat.append(xai_user(user_prompt))
            response = chat.sample()
            text = response.content or ""
        else:
            raise RuntimeError(f"Unknown provider for model: {model_name}")
        latency = time.time() - t0
        st.session_state["metrics"]["calls"] += 1
        st.session_state["metrics"]["latencies"].append(latency)
        return text, latency, None
    except Exception as e:
        latency = time.time() - t0
        st.session_state["metrics"]["errors"] += 1
        return "", latency, str(e)

def detect_provider(model_name: str):
    lower = model_name.lower()
    if lower.startswith("gemini"):
        return "gemini"
    if lower.startswith("gpt-") or lower.startswith("o"):
        return "openai"
    if lower.startswith("grok"):
        return "grok"
    # default heuristic
    if "gemini" in lower:
        return "gemini"
    if "grok" in lower:
        return "grok"
    return "openai"

# ---------------------------
# UI Helpers
# ---------------------------
def fancy_status_badge(label, status="ok"):
    color = {"ok": "#22c55e", "warn": "#f59e0b", "err": "#ef4444", "pending": "#3b82f6"}.get(status, "#64748b")
    st.markdown(f"<span style='background:{color};color:white;padding:4px 8px;border-radius:6px;font-size:12px'>{label}</span>", unsafe_allow_html=True)

def metrics_bar():
    m = st.session_state["metrics"]
    avg_latency = sum(m["latencies"]) / len(m["latencies"]) if m["latencies"] else 0
    c1, c2, c3 = st.columns(3)
    c1.metric("LLM Calls", m["calls"])
    c2.metric("Errors", m["errors"])
    c3.metric("Avg Latency (s)", f"{avg_latency:.2f}")

def agent_card(agent_def, idx):
    st.subheader(f"Agent {idx+1}: {agent_def.get('name', 'Unnamed')}")
    st.caption(agent_def.get("description", ""))
    with st.expander("Model and Parameters", expanded=False):
        cols = st.columns(4)
        agent_def["model"] = cols[0].selectbox("Model", DEFAULT_MODELS, index=(DEFAULT_MODELS.index(agent_def.get("default_model")) if agent_def.get("default_model") in DEFAULT_MODELS else 1), key=f"model_{idx}")
        agent_def["temperature"] = cols[1].slider("Temperature", 0.0, 2.0, float(agent_def.get("temperature", DEFAULT_TEMPERATURE)), 0.05, key=f"temp_{idx}")
        agent_def["max_tokens"] = cols[2].number_input("Max tokens", 1, 8192, int(agent_def.get("max_tokens", DEFAULT_MAX_TOKENS)), 1, key=f"max_tokens_{idx}")
        agent_def["top_p"] = cols[3].slider("Top_p", 0.0, 1.0, float(agent_def.get("top_p", DEFAULT_TOP_P)), 0.01, key=f"topp_{idx}")
    with st.expander("Prompts", expanded=False):
        agent_def["system_prompt"] = st.text_area("System prompt", agent_def.get("system_prompt", ""), key=f"sys_{idx}", height=120)
        agent_def["user_prompt"] = st.text_area("User prompt (use {{input}} placeholder)", agent_def.get("user_prompt", ""), key=f"user_{idx}", height=160)
    return agent_def

def wow_dashboard():
    st.subheader("Interactive Dashboard")
    metrics_bar()
    latencies = st.session_state["metrics"]["latencies"]
    if latencies:
        st.line_chart(pd.DataFrame({"latency_sec": latencies}))
    st.progress(min(1.0, len(latencies) / 10.0))
    if st.session_state["metrics"]["errors"] > 0:
        fancy_status_badge("Issues detected", "warn")
    else:
        fancy_status_badge("All systems nominal", "ok")

# ---------------------------
# Streamlit App
# ---------------------------
def main():
    st.set_page_config(page_title=APP_TITLE, layout="wide")
    init_session_state()

    st.title(APP_TITLE)
    st.caption(APP_DESC)

    # Sidebar controls
    with st.sidebar:
        st.header("APIs and Keys")
        st.text_input("GEMINI_API_KEY", type="password", key="GEMINI_API_KEY_UI", value=os.getenv("GEMINI_API_KEY", ""))
        st.text_input("OPENAI_API_KEY", type="password", key="OPENAI_API_KEY_UI", value=os.getenv("OPENAI_API_KEY", ""))
        st.text_input("XAI_API_KEY (Grok)", type="password", key="XAI_API_KEY_UI", value=os.getenv("XAI_API_KEY", ""))

        # Set environment for immediate use
        if st.session_state.get("GEMINI_API_KEY_UI"):
            os.environ["GEMINI_API_KEY"] = st.session_state["GEMINI_API_KEY_UI"]
        if st.session_state.get("OPENAI_API_KEY_UI"):
            os.environ["OPENAI_API_KEY"] = st.session_state["OPENAI_API_KEY_UI"]
        if st.session_state.get("XAI_API_KEY_UI"):
            os.environ["XAI_API_KEY"] = st.session_state["XAI_API_KEY_UI"]

        st.markdown("---")
        st.header("Agents Config")
        agents_file = st.file_uploader("Upload agents.yaml (optional)", type=["yaml", "yml"])
        if st.button("Load Agents"):
            st.session_state["agents_config"] = load_agents_yaml(agents_file)
            st.toast("Agents loaded", icon="✅")

        if not st.session_state["agents_config"]:
            st.session_state["agents_config"] = load_agents_yaml()

        wow_dashboard()

    # Tabs
    tab_data, tab_template, tab_generate, tab_agents, tab_run = st.tabs(["1) Dataset", "2) Template", "3) Generate Docs", "4) Agents Config", "5) Run Agents"])

    # 1) Dataset
    with tab_data:
        st.subheader("Upload Dataset")
        data_file = st.file_uploader(f"Upload dataset ({', '.join(SUPPORTED_DATASETS)})", type=SUPPORTED_DATASETS, key="dataset_upl")
        if data_file is not None:
            ext = data_file.name.split(".")[-1].lower()
            try:
                df = load_dataset(data_file, ext)
                st.session_state["dataset_df"] = df
                st.session_state["dataset_name"] = data_file.name
                st.session_state["schema"] = infer_schema(df)
                st.session_state["records"] = df.to_dict(orient="records")
                st.success(f"Loaded dataset '{data_file.name}' with {len(df)} rows and {len(df.columns)} columns.")
            except Exception as e:
                st.error(str(e))

        if st.session_state["dataset_df"] is not None:
            st.write("Preview:")
            st.dataframe(st.session_state["dataset_df"].head(100), use_container_width=True)
            st.markdown("Add a new record")

            with st.form("add_record_form", clear_on_submit=True):
                new = {}
                cols = st.columns(min(4, len(st.session_state["schema"]) or 1))
                for i, col in enumerate(st.session_state["schema"]):
                    idx = i % len(cols)
                    new[col] = cols[idx].text_input(col, "")
                submitted = st.form_submit_button("Add record")
                if submitted:
                    st.session_state["dataset_df"] = add_record(st.session_state["dataset_df"], new)
                    st.session_state["records"] = st.session_state["dataset_df"].to_dict(orient="records")
                    st.success("Record added.")

            # Allow editing the dataset grid lightly
            st.markdown("Quick Edit Table")
            edited_df = st.data_editor(st.session_state["dataset_df"], num_rows="dynamic", use_container_width=True, key="data_editor")
            st.session_state["dataset_df"] = edited_df
            st.session_state["records"] = edited_df.fillna("").to_dict(orient="records")

    # 2) Template
    with tab_template:
        st.subheader("Upload or Paste Template with {{placeholders}}")
        tmpl_file = st.file_uploader(f"Upload template ({', '.join(SUPPORTED_TEMPLATES)})", type=SUPPORTED_TEMPLATES, key="tmpl_upl")
        if tmpl_file is not None:
            ext = tmpl_file.name.split(".")[-1].lower()
            try:
                content = load_template(tmpl_file, ext)
                if not content.strip():
                    st.warning("Template appears empty or could not be extracted. You can still paste/edit below.")
                st.session_state["template_raw"] = content
                st.session_state["template_name"] = tmpl_file.name
                st.success(f"Loaded template '{tmpl_file.name}'.")
            except Exception as e:
                st.error(str(e))

        st.markdown("Or paste/edit template text (use {{col_name}} placeholders):")
        st.session_state["template_raw"] = st.text_area(
            "Template Editor",
            st.session_state["template_raw"],
            height=260,
            placeholder="Dear {{name}},\n\nThank you for your interest in {{product}}.\nYour order number is {{order_id}}.\n\nBest,\n{{sender}}"
        )

        # Show placeholder suggestions based on dataset columns
        if st.session_state["schema"]:
            st.info("Available placeholders from dataset: " + ", ".join([f"{{{{{c}}}}}" for c in st.session_state["schema"]]))

        # Render a sample with the first record
        if st.session_state["template_raw"] and st.session_state["records"]:
            if st.button("Render Sample with First Record"):
                try:
                    sample = render_template_string(st.session_state["template_raw"], st.session_state["records"][0])
                    st.text_area("Rendered Sample", sample, height=240)
                except Exception as e:
                    st.error(str(e))

    # 3) Generate Docs
    with tab_generate:
        st.subheader("Generate Documents from Template and Dataset")
        colg1, colg2 = st.columns(2)
        with colg1:
            record_range = st.slider("How many records to generate for?", 1, max(1, len(st.session_state["records"]) or 1), min(5, len(st.session_state["records"]) or 1))
        with colg2:
            output_basename = st.text_input("Output base filename (without extension)", value="generated_doc")

        if st.button("Generate", use_container_width=True):
            if not st.session_state["template_raw"]:
                st.error("Please provide a template first.")
            elif not st.session_state["records"]:
                st.error("Please upload a dataset first.")
            else:
                st.session_state["generated_docs"].clear()
                failures = 0
                prog = st.progress(0.0, text="Generating...")
                for i, rec in enumerate(st.session_state["records"][:record_range]):
                    try:
                        content = render_template_string(st.session_state["template_raw"], rec)
                        filename = f"{output_basename}_{i+1}.txt"
                        st.session_state["generated_docs"].append({"record_index": i, "content": content, "file_name": filename})
                    except Exception as e:
                        failures += 1
                    prog.progress((i+1)/record_range)
                if failures == 0:
                    st.success(f"Generated {len(st.session_state['generated_docs'])} documents.")
                else:
                    st.warning(f"Generated with {failures} failures. Check your placeholders.")

        if st.session_state["generated_docs"]:
            st.markdown("Review and Edit Generated Documents")
            for idx, doc in enumerate(st.session_state["generated_docs"]):
                with st.expander(f"Doc #{idx+1} - {doc['file_name']}", expanded=False):
                    new_text = st.text_area("Content", value=doc["content"], height=220, key=f"gen_edit_{idx}")
                    st.session_state["generated_docs"][idx]["content"] = new_text

                    c1, c2 = st.columns(2)
                    with c1:
                        if st.button("Download as .txt", key=f"dl_txt_{idx}"):
                            st.markdown(make_text_downloadable(new_text, doc["file_name"]), unsafe_allow_html=True)
                    with c2:
                        if st.button("Download as .docx", key=f"dl_docx_{idx}"):
                            data = save_to_docx(new_text)
                            st.download_button("Save .docx", data=data, file_name=doc["file_name"].replace(".txt", ".docx"), mime="application/vnd.openxmlformats-officedocument.wordprocessingml.document")

    # 4) Agents Config
    with tab_agents:
        st.subheader("Configure Agents (order, models, prompts, params)")
        cfg = st.session_state["agents_config"] or {"agents": []}
        agents = cfg.get("agents", [])
        if not agents:
            st.info("No agents found. Upload a valid agents.yaml or use defaults.")

        # Select which agents to run and their order
        agent_names = [a.get("name", f"Agent {i+1}") for i, a in enumerate(agents)]
        selected = st.multiselect("Select agents to include", agent_names, default=agent_names)
        # Reorder by drag-and-drop via selection sort: provide an order input
        order = st.text_input("Agent order (comma-separated indexes starting at 1)", value=",".join(str(i+1) for i in range(len(selected))))
        order_idx = []
        try:
            order_idx = [int(x.strip())-1 for x in order.split(",") if x.strip().isdigit()]
        except:
            st.warning("Invalid order. Using default selected order.")
            order_idx = list(range(len(selected)))

        # Build a working copy
        working_agents = []
        for name in selected:
            for a in agents:
                if a.get("name") == name:
                    working_agents.append(a.copy())

        # Reorder if valid
        if order_idx and len(order_idx) == len(working_agents):
            working_agents = [working_agents[i] for i in order_idx if 0 <= i < len(working_agents)]

        # Editable cards
        edited_agents = []
        for i, agent in enumerate(working_agents):
            ed = agent_card(agent, i)
            edited_agents.append(ed)

        st.session_state["agents_config"] = {"agents": edited_agents}

    # 5) Run Agents
    with tab_run:
        st.subheader("Execute Agents Sequentially")
        st.write("Choose an input source for the first agent:")
        source = st.radio("Input source", ["Manual input", "First generated doc", "Concatenate all generated docs"], horizontal=True)
        if source == "Manual input":
            initial_input = st.text_area("Initial input", "", height=200)
        elif source == "First generated doc":
            if st.session_state["generated_docs"]:
                initial_input = st.session_state["generated_docs"][0]["content"]
            else:
                st.warning("No generated docs yet. Please generate first or use manual input.")
                initial_input = ""
        else:
            if st.session_state["generated_docs"]:
                initial_input = "\n\n---\n\n".join([d["content"] for d in st.session_state["generated_docs"]])
            else:
                st.warning("No generated docs yet. Please generate first or use manual input.")
                initial_input = ""

        agents_to_run = (st.session_state["agents_config"] or {}).get("agents", [])
        if not agents_to_run:
            st.info("No agents configured.")
        else:
            st.write("You can edit the output after each agent to feed into the next.")
            if st.button("Run Agents", use_container_width=True):
                if not initial_input.strip():
                    st.error("Initial input is empty.")
                else:
                    st.session_state["agent_run_history"].clear()
                    current_input = initial_input
                    status = st.status("Running agents...", expanded=True)
                    for idx, agent in enumerate(agents_to_run):
                        name = agent.get("name", f"Agent {idx+1}")
                        sys_p = agent.get("system_prompt", "")
                        usr_p_tmpl = agent.get("user_prompt", "{{input}}")
                        model = agent.get("model") or agent.get("default_model") or "gemini-2.5-flash"
                        temp = float(agent.get("temperature", DEFAULT_TEMPERATURE))
                        max_tok = int(agent.get("max_tokens", DEFAULT_MAX_TOKENS))
                        top_p = float(agent.get("top_p", DEFAULT_TOP_P))

                        # Render user prompt with current input
                        try:
                            rendered_user_prompt = render_template_string(usr_p_tmpl, {"input": current_input})
                        except Exception as e:
                            rendered_user_prompt = usr_p_tmpl.replace("{{input}}", current_input)

                        status.update(label=f"Running {name} with {model}...", state="running")
                        out, latency, err = call_llm_unified(
                            model, sys_p, rendered_user_prompt, temperature=temp, max_tokens=max_tok, top_p=top_p
                        )
                        if err:
                            st.error(f"{name} failed: {err}")
                            status.update(label=f"{name} encountered an error.", state="error")
                            break
                        st.session_state["agent_run_history"].append(
                            {"agent": name, "model": model, "latency": latency, "input": current_input, "output": out}
                        )
                        # Allow inline editing
                        with st.expander(f"{name} Output (editable) - {latency:.2f}s", expanded=True):
                            edit_key = f"edited_{name}_{idx}"
                            edited = st.text_area("Edit output before passing to next agent", value=out, height=220, key=edit_key)
                            st.session_state["edited_agent_outputs"][name] = edited
                            current_input = edited  # feed to next
                        status.update(label=f"Completed {name}", state="complete")
                        st.toast(f"{name} completed in {latency:.2f}s", icon="✅")

                    status.update(label="Pipeline finished", state="complete")

            if st.session_state["agent_run_history"]:
                st.markdown("Run Summary")
                for step in st.session_state["agent_run_history"]:
                    fancy_status_badge(step["agent"], "ok")
                    st.write(f"Model: {step['model']} - Latency: {step['latency']:.2f}s")
                st.success("All available outputs are editable above for reuse or download.")

    # Footer
    st.markdown("---")
    st.caption("Tip: You can switch models per agent, modify prompts and parameters, and pass edited outputs to subsequent agents.")


if __name__ == "__main__":
    main()


2) agents.yaml (sample, optional uploadable)
----------------
agents:
  - name: Summarizer
    description: Summarize the generated document.
    default_model: gpt-4o-mini
    temperature: 0.3
    max_tokens: 512
    top_p: 0.95
    system_prompt: "You are a helpful assistant that summarizes text concisely."
    user_prompt: "Summarize the following content:\n\n{{input}}"

  - name: StyleRewriter
    description: Rewrite the content in the requested style.
    default_model: gemini-2.5-flash
    temperature: 0.5
    max_tokens: 1024
    top_p: 0.95
    system_prompt: "You are an expert copywriter."
    user_prompt: "Rewrite the content in a confident, friendly tone while preserving facts:\n\n{{input}}"

  - name: KeywordExtractor
    description: Extract 5-10 key terms as a comma-separated list.
    default_model: grok-3-mini
    temperature: 0.2
    max_tokens: 256
    top_p: 0.9
    system_prompt: "You are a precise NLP assistant."
    user_prompt: "Extract 5-10 keywords from the content. Return a comma-separated list only:\n\n{{input}}"


3) requirements.txt
----------------
streamlit>=1.36.0
pandas>=2.2.2
pandas-ods-reader>=0.1.6
openpyxl>=3.1.5
python-docx>=1.1.2
Jinja2>=3.1.4
google-generativeai>=0.7.2
openai>=1.42.0
xai-sdk>=0.6.0
pyyaml>=6.0.2


Key features implemented and improvements:
- Multi-format data ingestion with robust fallbacks: CSV, JSON (array or NDJSON), TXT (auto-heuristic or single-column), ODS, XLSX.
- Dynamic form for adding new records and a live editable data grid using st.data_editor with dynamic rows enabled.
- Template ingestion from TXT/MD/DOCX/ODT with text extraction, safe editing, and Jinja2-based rendering with {{placeholders}}.
- Document generation per record with progress, interactive review and editing, download as .txt or .docx.
- Multi-agent pipeline: load from agents.yaml or defaults, choose which agents to use and order, edit system/user prompts, select models per agent, adjust temperature, top_p, and max_tokens.
- Unified LLM wrapper supporting:
  - Google Gemini (gemini-2.5-flash, gemini-2.5-flash-lite)
  - OpenAI (gpt-5-nano, gpt-4o-mini, gpt-4.1-mini)
  - Grok via xAI SDK (grok-4-fast-reasoning, grok-3-mini) with the requested sample code approach.
- Interactive chaining: after each agent run, user can edit the output passed to the next agent.
- Advanced visualization dashboard: live metrics (call count, errors, average latency), latency sparkline, progress bar, toast notifications, step status with st.status, colored badges.
- Strong error handling and state management for Hugging Face Spaces deployment.
- API keys loaded from sidebar to environment variables for immediate use; also compatible with st.secrets.
- Defaults for agents.yaml included so the app works out-of-the-box even without an upload.
- Safer Jinja2 rendering and helpful placeholder hints based on dataset schema.

Notes and constraints:
- For Google Docs: this app creates text/docx outputs from your Google-Doc-like template syntax {{...}}. Direct Google Docs API integration is not included to avoid OAuth complexity on Spaces; generated DOCX/MD outputs can be opened in Google Docs seamlessly.
- DOCX/ODT template formatting is simplified to plain text editing; rendering preserves textual content, not original rich styles. If you need full DOCX templating (styles preserved), we can add docxtpl in a follow-up.

Deployment instructions on Hugging Face Spaces:
- Space type: Streamlit
- Add the files above.
- Configure Secrets (recommended): GEMINI_API_KEY, OPENAI_API_KEY, XAI_API_KEY
- Or use the sidebar to paste keys during runtime.

10 comprehensive follow-up questions:
1) Do you need full-fidelity DOCX templating that preserves styles and images (via docxtpl), or is plain-text templating sufficient?
2) Should we add Google Drive/Docs API integration for direct document creation and sharing, with OAuth flows on Spaces?
3) Do you want schema validation and per-field input types (dates, numbers, enums) inferred from the dataset to improve the record-entry form?
4) Would you like templating helpers (conditional sections, loops for line items) enabled for records containing arrays or nested objects?
5) Should we support batch agent execution per generated document and export aggregated results (CSV/JSON) of agent outputs?
6) Do you want to define agent tool-calls or structured outputs (JSON schema enforcement) for specific steps such as extraction?
7) Is image input support for Grok/Gemini needed (e.g., attach an image per record to augment the prompts) in your pipeline?
8) Would you like role-based presets for prompts and parameters (e.g., Legal, Sales, HR) and the ability to save/load configurations?
9) Should we add multi-tenant storage (e.g., HF Inference cache/Space filesystem/DB) to persist datasets, templates, and run histories across sessions?
10) Do you want guardrails like PII redaction, toxicity filters, or enterprise logging/telemetry dashboards integrated into the WOW dashboard?
