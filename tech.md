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

# xAI / Grok SDK
try:
    from xai_sdk import Client as XAIClient
    from xai_sdk.chat import user as xai_user, system as xai_system
    XAI_AVAILABLE = True
except ImportError:
    XAI_AVAILABLE = False

# ---------------------------
# FLORA THEMES CONFIGURATION
# ---------------------------
FLORA_THEMES = {
    "Cherry Blossom": {
        "primary": "#FFB7C5",
        "secondary": "#FFC0CB",
        "accent": "#FF69B4",
        "background": "#FFF5F7",
        "text": "#4A4A4A",
        "gradient": "linear-gradient(135deg, #FFB7C5 0%, #FFC0CB 100%)"
    },
    "Lavender Dream": {
        "primary": "#E6E6FA",
        "secondary": "#D8BFD8",
        "accent": "#9370DB",
        "background": "#F8F8FF",
        "text": "#4B0082",
        "gradient": "linear-gradient(135deg, #E6E6FA 0%, #D8BFD8 100%)"
    },
    "Sunflower Bright": {
        "primary": "#FFD700",
        "secondary": "#FFA500",
        "accent": "#FF8C00",
        "background": "#FFFACD",
        "text": "#8B4513",
        "gradient": "linear-gradient(135deg, #FFD700 0%, #FFA500 100%)"
    },
    "Ocean Breeze": {
        "primary": "#87CEEB",
        "secondary": "#4682B4",
        "accent": "#1E90FF",
        "background": "#F0F8FF",
        "text": "#2F4F4F",
        "gradient": "linear-gradient(135deg, #87CEEB 0%, #4682B4 100%)"
    },
    "Rose Garden": {
        "primary": "#FF6B9D",
        "secondary": "#C9184A",
        "accent": "#A4133C",
        "background": "#FFF0F3",
        "text": "#590D22",
        "gradient": "linear-gradient(135deg, #FF6B9D 0%, #C9184A 100%)"
    },
    "Mint Fresh": {
        "primary": "#98FF98",
        "secondary": "#00FA9A",
        "accent": "#00CED1",
        "background": "#F0FFF0",
        "text": "#2F4F4F",
        "gradient": "linear-gradient(135deg, #98FF98 0%, #00FA9A 100%)"
    },
    "Violet Twilight": {
        "primary": "#8A2BE2",
        "secondary": "#9400D3",
        "accent": "#9932CC",
        "background": "#F8F4FF",
        "text": "#4B0082",
        "gradient": "linear-gradient(135deg, #8A2BE2 0%, #9400D3 100%)"
    },
    "Peach Sorbet": {
        "primary": "#FFDAB9",
        "secondary": "#FFB6C1",
        "accent": "#FF7F50",
        "background": "#FFF5EE",
        "text": "#8B4513",
        "gradient": "linear-gradient(135deg, #FFDAB9 0%, #FFB6C1 100%)"
    },
    "Iris Elegance": {
        "primary": "#5D3FD3",
        "secondary": "#7B68EE",
        "accent": "#6A5ACD",
        "background": "#F5F5FF",
        "text": "#2E2E5F",
        "gradient": "linear-gradient(135deg, #5D3FD3 0%, #7B68EE 100%)"
    },
    "Tulip Paradise": {
        "primary": "#FF5470",
        "secondary": "#FF6B9D",
        "accent": "#C9184A",
        "background": "#FFF0F5",
        "text": "#590D22",
        "gradient": "linear-gradient(135deg, #FF5470 0%, #FF6B9D 100%)"
    },
    "Daisy Field": {
        "primary": "#FFFFE0",
        "secondary": "#FFFACD",
        "accent": "#FFD700",
        "background": "#FFFFF0",
        "text": "#6B5B4D",
        "gradient": "linear-gradient(135deg, #FFFFE0 0%, #FFFACD 100%)"
    },
    "Orchid Mystique": {
        "primary": "#DA70D6",
        "secondary": "#BA55D3",
        "accent": "#9932CC",
        "background": "#FFF0FA",
        "text": "#4B0082",
        "gradient": "linear-gradient(135deg, #DA70D6 0%, #BA55D3 100%)"
    },
    "Hydrangea Blue": {
        "primary": "#6495ED",
        "secondary": "#4169E1",
        "accent": "#0000CD",
        "background": "#F0F8FF",
        "text": "#191970",
        "gradient": "linear-gradient(135deg, #6495ED 0%, #4169E1 100%)"
    },
    "Marigold Sunset": {
        "primary": "#FF9500",
        "secondary": "#FF7F00",
        "accent": "#FF6347",
        "background": "#FFF8DC",
        "text": "#8B4513",
        "gradient": "linear-gradient(135deg, #FF9500 0%, #FF7F00 100%)"
    },
    "Lily White": {
        "primary": "#FFFFFF",
        "secondary": "#F5F5F5",
        "accent": "#E0E0E0",
        "background": "#FAFAFA",
        "text": "#333333",
        "gradient": "linear-gradient(135deg, #FFFFFF 0%, #F5F5F5 100%)"
    },
    "Magnolia Cream": {
        "primary": "#FFF8DC",
        "secondary": "#FAEBD7",
        "accent": "#FFE4B5",
        "background": "#FFFEF0",
        "text": "#8B7355",
        "gradient": "linear-gradient(135deg, #FFF8DC 0%, #FAEBD7 100%)"
    },
    "Poppy Red": {
        "primary": "#FF4500",
        "secondary": "#FF6347",
        "accent": "#DC143C",
        "background": "#FFF5F0",
        "text": "#8B0000",
        "gradient": "linear-gradient(135deg, #FF4500 0%, #FF6347 100%)"
    },
    "Jasmine Night": {
        "primary": "#2C3E50",
        "secondary": "#34495E",
        "accent": "#5D6D7E",
        "background": "#ECF0F1",
        "text": "#1C2833",
        "gradient": "linear-gradient(135deg, #2C3E50 0%, #34495E 100%)"
    },
    "Wisteria Grove": {
        "primary": "#C9A0DC",
        "secondary": "#B19CD9",
        "accent": "#9370DB",
        "background": "#F5F0FF",
        "text": "#4B0082",
        "gradient": "linear-gradient(135deg, #C9A0DC 0%, #B19CD9 100%)"
    },
    "Hibiscus Tropical": {
        "primary": "#FF1493",
        "secondary": "#FF69B4",
        "accent": "#C71585",
        "background": "#FFF0F8",
        "text": "#8B008B",
        "gradient": "linear-gradient(135deg, #FF1493 0%, #FF69B4 100%)"
    }
}

# ---------------------------
# Configuration and Constants
# ---------------------------
APP_TITLE = "🌸 Agentic Docs Builder Flora Edition"
APP_DESC = "Beautiful themed document generation with AI agents"

SUPPORTED_DATASETS = ["csv", "json", "txt", "ods", "xlsx"]
SUPPORTED_TEMPLATES = ["txt", "md", "markdown", "odt", "docx"]

DEFAULT_MODELS = [
    "gemini-2.0-flash-exp",
    "gemini-1.5-flash",
    "gpt-4o-mini",
    "gpt-4-turbo",
    "grok-beta",
]

DEFAULT_TEMPERATURE = 0.3
DEFAULT_MAX_TOKENS = 1024
DEFAULT_TOP_P = 0.95

# ---------------------------
# State Management
# ---------------------------
def init_session_state():
    defaults = {
        "theme": "Cherry Blossom",
        "dataset_df": None,
        "dataset_name": "",
        "schema": [],
        "records": [],
        "template_raw": "",
        "template_name": "",
        "generated_docs": [],
        "agents_config": None,
        "agent_run_history": [],
        "edited_agent_outputs": {},
        "metrics": {"calls": 0, "errors": 0, "latencies": [], "success_rate": 100.0}
    }
    for key, val in defaults.items():
        st.session_state.setdefault(key, val)

# ---------------------------
# Theme Application
# ---------------------------
def apply_theme(theme_name):
    theme = FLORA_THEMES.get(theme_name, FLORA_THEMES["Cherry Blossom"])
    
    custom_css = f"""
    <style>
        /* Main Theme Colors */
        :root {{
            --primary-color: {theme['primary']};
            --secondary-color: {theme['secondary']};
            --accent-color: {theme['accent']};
            --background-color: {theme['background']};
            --text-color: {theme['text']};
        }}
        
        /* App Background */
        .stApp {{
            background: {theme['background']};
            background-image: {theme['gradient']};
            background-attachment: fixed;
        }}
        
        /* Headers */
        h1, h2, h3 {{
            color: {theme['text']} !important;
            font-family: 'Segoe UI', sans-serif;
        }}
        
        /* Cards and Containers */
        .stTabs [data-baseweb="tab-panel"] {{
            background: rgba(255, 255, 255, 0.9);
            border-radius: 15px;
            padding: 20px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }}
        
        /* Buttons */
        .stButton>button {{
            background: {theme['gradient']};
            color: white;
            border: none;
            border-radius: 10px;
            padding: 10px 24px;
            font-weight: 600;
            transition: all 0.3s;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
        }}
        
        .stButton>button:hover {{
            transform: translateY(-2px);
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
        }}
        
        /* Metrics */
        .metric-card {{
            background: white;
            border-radius: 12px;
            padding: 16px;
            box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
            border-left: 4px solid {theme['accent']};
            margin: 8px 0;
        }}
        
        /* Status Badges */
        .status-badge {{
            display: inline-block;
            padding: 6px 12px;
            border-radius: 20px;
            font-size: 12px;
            font-weight: 600;
            margin: 4px;
            animation: fadeIn 0.5s;
        }}
        
        @keyframes fadeIn {{
            from {{ opacity: 0; }}
            to {{ opacity: 1; }}
        }}
        
        /* Expanders */
        .streamlit-expanderHeader {{
            background: {theme['gradient']};
            color: white !important;
            border-radius: 8px;
            font-weight: 600;
        }}
        
        /* Sidebar */
        [data-testid="stSidebar"] {{
            background: linear-gradient(180deg, {theme['primary']} 0%, {theme['secondary']} 100%);
        }}
        
        /* Progress Bar */
        .stProgress > div > div > div > div {{
            background: {theme['gradient']};
        }}
        
        /* Dataframe */
        .stDataFrame {{
            border-radius: 10px;
            overflow: hidden;
            box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
        }}
        
        /* Text Areas */
        .stTextArea textarea {{
            border-radius: 8px;
            border: 2px solid {theme['primary']};
        }}
        
        /* Visualization Cards */
        .viz-card {{
            background: white;
            border-radius: 15px;
            padding: 20px;
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
            margin: 10px 0;
            border-top: 3px solid {theme['accent']};
        }}
    </style>
    """
    st.markdown(custom_css, unsafe_allow_html=True)

# ---------------------------
# Enhanced UI Components
# ---------------------------
def fancy_status_badge(label, status="ok"):
    colors = {
        "ok": "#22c55e",
        "warn": "#f59e0b",
        "err": "#ef4444",
        "pending": "#3b82f6",
        "running": "#8b5cf6"
    }
    color = colors.get(status, "#64748b")
    st.markdown(
        f'<span class="status-badge" style="background:{color};color:white;">{label}</span>',
        unsafe_allow_html=True
    )

def metrics_dashboard():
    m = st.session_state["metrics"]
    avg_latency = sum(m["latencies"]) / len(m["latencies"]) if m["latencies"] else 0
    
    col1, col2, col3, col4 = st.columns(4)
    
    with col1:
        st.markdown('<div class="metric-card">', unsafe_allow_html=True)
        st.metric("🚀 API Calls", m["calls"], delta=None)
        st.markdown('</div>', unsafe_allow_html=True)
    
    with col2:
        st.markdown('<div class="metric-card">', unsafe_allow_html=True)
        st.metric("⚡ Avg Latency", f"{avg_latency:.2f}s", delta=None)
        st.markdown('</div>', unsafe_allow_html=True)
    
    with col3:
        st.markdown('<div class="metric-card">', unsafe_allow_html=True)
        st.metric("❌ Errors", m["errors"], delta=None)
        st.markdown('</div>', unsafe_allow_html=True)
    
    with col4:
        st.markdown('<div class="metric-card">', unsafe_allow_html=True)
        success_rate = 100 * (1 - m["errors"] / max(m["calls"], 1))
        st.metric("✅ Success Rate", f"{success_rate:.1f}%", delta=None)
        st.markdown('</div>', unsafe_allow_html=True)

def interactive_dashboard():
    st.markdown('<div class="viz-card">', unsafe_allow_html=True)
    st.subheader("📊 Interactive Dashboard")
    
    metrics_dashboard()
    
    latencies = st.session_state["metrics"]["latencies"]
    if latencies:
        st.markdown("### Response Time Trend")
        df_latency = pd.DataFrame({
            "Call #": range(1, len(latencies) + 1),
            "Latency (s)": latencies
        })
        st.line_chart(df_latency.set_index("Call #"))
        
        # Performance indicator
        recent_avg = sum(latencies[-5:]) / min(5, len(latencies))
        if recent_avg < 2:
            fancy_status_badge("⚡ Excellent Performance", "ok")
        elif recent_avg < 5:
            fancy_status_badge("✓ Good Performance", "pending")
        else:
            fancy_status_badge("⚠ Slow Performance", "warn")
    
    st.markdown('</div>', unsafe_allow_html=True)

# ---------------------------
# Data I/O Functions
# ---------------------------
def load_dataset(file, filetype):
    try:
        if filetype == "csv":
            df = pd.read_csv(file)
        elif filetype == "json":
            content = file.getvalue().decode("utf-8", errors="ignore")
            try:
                data = json.loads(content)
                df = pd.DataFrame(data if isinstance(data, list) else [data])
            except:
                records = [json.loads(line) for line in content.splitlines() if line.strip()]
                df = pd.DataFrame(records)
        elif filetype == "txt":
            content = file.getvalue().decode("utf-8", errors="ignore")
            lines = [ln for ln in content.splitlines() if ln.strip()]
            df = pd.DataFrame({"text": lines}) if lines else pd.DataFrame()
        elif filetype == "ods":
            with tempfile.NamedTemporaryFile(suffix=".ods", delete=False) as tmp:
                tmp.write(file.getvalue())
                tmp.flush()
                df = read_ods(tmp.name, 1)
        elif filetype == "xlsx":
            df = pd.read_excel(file)
        else:
            raise ValueError("Unsupported format")
        return df.fillna("")
    except Exception as e:
        raise RuntimeError(f"Failed to load dataset: {str(e)}")

def load_template(file, ext):
    try:
        ext = ext.lower()
        if ext in ["txt", "md", "markdown"]:
            return file.getvalue().decode("utf-8", errors="ignore")
        elif ext == "docx":
            with tempfile.NamedTemporaryFile(suffix=".docx", delete=False) as tmp:
                tmp.write(file.getvalue())
                tmp.flush()
                doc = DocxDocument(tmp.name)
                return "\n".join([p.text for p in doc.paragraphs])
        else:
            return file.getvalue().decode("utf-8", errors="ignore")
    except Exception as e:
        raise RuntimeError(f"Failed to load template: {str(e)}")

def render_template_string(template_str: str, context: dict) -> str:
    try:
        env = Environment(autoescape=False)
        tmpl = env.from_string(template_str)
        return tmpl.render(**context)
    except Exception as e:
        raise RuntimeError(f"Template error: {str(e)}")

# ---------------------------
# LLM Functions
# ---------------------------
def call_llm_unified(model_name, system_prompt, user_prompt, temperature=DEFAULT_TEMPERATURE, 
                     max_tokens=DEFAULT_MAX_TOKENS, top_p=DEFAULT_TOP_P):
    t0 = time.time()
    
    try:
        if "gemini" in model_name.lower():
            api_key = os.getenv("GEMINI_API_KEY")
            if not api_key:
                raise RuntimeError("GEMINI_API_KEY not found")
            genai.configure(api_key=api_key)
            model = genai.GenerativeModel(model_name)
            prompt = f"{system_prompt}\n\n{user_prompt}" if system_prompt else user_prompt
            resp = model.generate_content(
                prompt,
                generation_config={
                    "temperature": float(temperature),
                    "top_p": float(top_p),
                    "max_output_tokens": int(max_tokens),
                }
            )
            text = resp.text
            
        elif "gpt" in model_name.lower():
            api_key = os.getenv("OPENAI_API_KEY")
            if not api_key:
                raise RuntimeError("OPENAI_API_KEY not found")
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
                top_p=float(top_p)
            )
            text = resp.choices[0].message.content
            
        elif "grok" in model_name.lower() and XAI_AVAILABLE:
            api_key = os.getenv("XAI_API_KEY")
            if not api_key:
                raise RuntimeError("XAI_API_KEY not found")
            client = XAIClient(api_key=api_key)
            chat = client.chat.create(model="grok-beta")
            if system_prompt:
                chat.append(xai_system(system_prompt))
            chat.append(xai_user(user_prompt))
            response = chat.sample()
            text = response.content
        else:
            raise RuntimeError(f"Unsupported model: {model_name}")
            
        latency = time.time() - t0
        st.session_state["metrics"]["calls"] += 1
        st.session_state["metrics"]["latencies"].append(latency)
        return text, latency, None
        
    except Exception as e:
        latency = time.time() - t0
        st.session_state["metrics"]["errors"] += 1
        return "", latency, str(e)

# ---------------------------
# Agents Configuration
# ---------------------------
DEFAULT_AGENTS_YAML = """
agents:
  - name: Summarizer
    description: Concise summary generator
    default_model: gemini-1.5-flash
    temperature: 0.3
    max_tokens: 512
    top_p: 0.95
    system_prompt: "You are a helpful assistant that summarizes text concisely."
    user_prompt: "Summarize:\\n\\n{{input}}"
    
  - name: StyleRewriter
    description: Style transformation expert
    default_model: gpt-4o-mini
    temperature: 0.5
    max_tokens: 1024
    top_p: 0.95
    system_prompt: "You are an expert copywriter."
    user_prompt: "Rewrite in a friendly tone:\\n\\n{{input}}"
"""

def load_agents_yaml(uploaded=None):
    if uploaded is None:
        return yaml.safe_load(DEFAULT_AGENTS_YAML)
    try:
        content = uploaded.getvalue().decode("utf-8")
        return yaml.safe_load(content)
    except:
        return yaml.safe_load(DEFAULT_AGENTS_YAML)

# ---------------------------
# Main Application
# ---------------------------
def main():
    st.set_page_config(
        page_title="Flora Agentic Builder",
        page_icon="🌸",
        layout="wide",
        initial_sidebar_state="expanded"
    )
    
    init_session_state()
    
    # Sidebar - Theme selector and config
    with st.sidebar:
        st.title("🌸 Flora Themes")
        selected_theme = st.selectbox(
            "Choose Your Theme",
            options=list(FLORA_THEMES.keys()),
            index=list(FLORA_THEMES.keys()).index(st.session_state["theme"])
        )
        
        if selected_theme != st.session_state["theme"]:
            st.session_state["theme"] = selected_theme
            st.rerun()
        
        st.markdown("---")
        
        st.header("🔑 API Keys")
        gemini_key = st.text_input("Gemini API Key", type="password", value=os.getenv("GEMINI_API_KEY", ""))
        openai_key = st.text_input("OpenAI API Key", type="password", value=os.getenv("OPENAI_API_KEY", ""))
        xai_key = st.text_input("xAI API Key", type="password", value=os.getenv("XAI_API_KEY", ""))
        
        if gemini_key:
            os.environ["GEMINI_API_KEY"] = gemini_key
        if openai_key:
            os.environ["OPENAI_API_KEY"] = openai_key
        if xai_key:
            os.environ["XAI_API_KEY"] = xai_key
        
        st.markdown("---")
        
        st.header("🤖 Agents")
        agents_file = st.file_uploader("Upload agents.yaml", type=["yaml", "yml"])
        if st.button("Load Agents Config"):
            st.session_state["agents_config"] = load_agents_yaml(agents_file)
            st.success("✓ Agents loaded!")
        
        if not st.session_state["agents_config"]:
            st.session_state["agents_config"] = load_agents_yaml()
        
        st.markdown("---")
        interactive_dashboard()
    
    # Apply theme
    apply_theme(st.session_state["theme"])
    
    # Main header
    st.title(APP_TITLE)
    st.caption(APP_DESC)
    
    # Tabs
    tabs = st.tabs(["📊 Dataset", "📝 Template", "🚀 Generate", "🤖 Agents", "▶️ Run Pipeline"])
    
    # Tab 1: Dataset
    with tabs[0]:
        st.header("📊 Dataset Management")
        
        col1, col2 = st.columns([2, 1])
        
        with col1:
            data_file = st.file_uploader(
                "Upload Dataset",
                type=SUPPORTED_DATASETS,
                help="Supports CSV, JSON, TXT, ODS, XLSX"
            )
            
            if data_file:
                ext = data_file.name.split(".")[-1].lower()
                try:
                    df = load_dataset(data_file, ext)
                    st.session_state["dataset_df"] = df
                    st.session_state["dataset_name"] = data_file.name
                    st.session_state["schema"] = list(df.columns)
                    st.session_state["records"] = df.to_dict(orient="records")
                    fancy_status_badge(f"✓ Loaded {len(df)} records", "ok")
                except Exception as e:
                    fancy_status_badge(f"✗ Error: {str(e)}", "err")
        
        with col2:
            if st.session_state["dataset_df"] is not None:
                st.metric("Total Records", len(st.session_state["dataset_df"]))
                st.metric("Columns", len(st.session_state["schema"]))
        
        if st.session_state["dataset_df"] is not None:
            st.markdown("### Preview")
            st.dataframe(st.session_state["dataset_df"], use_container_width=True)
            
            with st.expander("✏️ Edit Dataset", expanded=False):
                edited_df = st.data_editor(
                    st.session_state["dataset_df"],
                    num_rows="dynamic",
                    use_container_width=True
                )
                st.session_state["dataset_df"] = edited_df
                st.session_state["records"] = edited_df.to_dict(orient="records")
    
    # Tab 2: Template
    with tabs[1]:
        st.header("📝 Template Editor")
        
        tmpl_file = st.file_uploader("Upload Template", type=SUPPORTED_TEMPLATES)
        
        if tmpl_file:
            ext = tmpl_file.name.split(".")[-1].lower()
            try:
                content = load_template(tmpl_file, ext)
                st.session_state["template_raw"] = content
                st.session_state["template_name"] = tmpl_file.name
                fancy_status_badge(f"✓ Loaded {tmpl_file.name}", "ok")
            except Exception as e:
                fancy_status_badge(f"✗ Error: {str(e)}", "err")
        
        st.session_state["template_raw"] = st.text_area(
            "Template Content (use {{variable}} placeholders)",
            st.session_state["template_raw"],
            height=300,
            placeholder="Dear {{name}},\n\nThank you for {{action}}.\n\nBest regards"
        )
        
        if st.session_state["schema"]:
            st.info("💡 Available: " + ", ".join([f"{{{{{c}}}}}" for c in st.session_state["schema"]]))
        
        if st.button("🔍 Preview with First Record") and st.session_state["records"]:
            try:
                preview = render_template_string(
                    st.session_state["template_raw"],
                    st.session_state["records"][0]
                )
                st.text_area("Preview", preview, height=200)
            except Exception as e:
                st.error(f"Preview error: {str(e)}")
    
    # Tab 3: Generate
    with tabs[2]:
        st.header("🚀 Generate Documents")
        
        if not st.session_state["template_raw"] or not st.session_state["records"]:
            st.warning("⚠️ Please load dataset and template first")
        else:
            col1, col2 = st.columns(2)
            
            with col1:
                num_records = st.slider(
                    "Records to generate",
                    1,
                    len(st.session_state["records"]),
                    min(5, len(st.session_state["records"]))
                )
            
            with col2:
                basename = st.text_input("Output filename", "document")
            
            if st.button("✨ Generate Documents", use_container_width=True):
                st.session_state["generated_docs"].clear()
                progress_bar = st.progress(0.0)
                status_text = st.empty()
                
                for i, rec in enumerate(st.session_state["records"][:num_records]):
                    try:
                        content = render_template_string(st.session_state["template_raw"], rec)
                        st.session_state["generated_docs"].append({
                            "record_index": i,
                            "content": content,
                            "file_name": f"{basename}_{i+1}.txt"
                        })
                        status_text.text(f"Generated {i+1}/{num_records}...")
                        progress_bar.progress((i + 1) / num_records)
                    except Exception as e:
                        st.error(f"Error in record {i+1}: {str(e)}")
                
                fancy_status_badge(f"✓ Generated {len(st.session_state['generated_docs'])} docs", "ok")
            
            if st.session_state["generated_docs"]:
                st.markdown("### 📄 Generated Documents")
                
                for idx, doc in enumerate(st.session_state["generated_docs"]):
                    with st.expander(f"📄 {doc['file_name']}", expanded=False):
                        new_content = st.text_area(
                            "Edit content",
                            doc["content"],
                            height=200,
                            key=f"doc_edit_{idx}"
                        )
                        st.session_state["generated_docs"][idx]["content"] = new_content
                        
                        col1, col2, col3 = st.columns(3)
                        with col1:
                            st.download_button(
                                "⬇️ Download TXT",
                                data=new_content,
                                file_name=doc["file_name"],
                                mime="text/plain"
                            )
                        with col2:
                            if st.button("📋 Copy", key=f"copy_{idx}"):
                                st.toast("Copied to clipboard!", icon="✓")
                        with col3:
                            if st.button("🗑️ Delete", key=f"del_{idx}"):
                                st.session_state["generated_docs"].pop(idx)
                                st.rerun()
    
    # Tab 4: Agents
    with tabs[3]:
        st.header("🤖 Configure AI Agents")
        
        cfg = st.session_state.get("agents_config", {"agents": []})
        agents = cfg.get("agents", [])
        
        if not agents:
            st.info("No agents configured. Upload agents.yaml or use defaults.")
        else:
            st.markdown(f"**{len(agents)} agents loaded**")
            
            for idx, agent in enumerate(agents):
                with st.expander(f"🤖 {agent.get('name', f'Agent {idx+1}')}", expanded=False):
                    st.caption(agent.get("description", "No description"))
                    
                    col1, col2, col3, col4 = st.columns(4)
                    
                    with col1:
                        model_options = DEFAULT_MODELS
                        current_model = agent.get("default_model", model_options[0])
                        if current_model not in model_options:
                            model_options.append(current_model)
                        agent["model"] = st.selectbox(
                            "Model",
                            model_options,
                            index=model_options.index(current_model),
                            key=f"model_{idx}"
                        )
                    
                    with col2:
                        agent["temperature"] = st.slider(
                            "Temperature",
                            0.0, 2.0,
                            float(agent.get("temperature", 0.3)),
                            0.05,
                            key=f"temp_{idx}"
                        )
                    
                    with col3:
                        agent["max_tokens"] = st.number_input(
                            "Max Tokens",
                            100, 4096,
                            int(agent.get("max_tokens", 1024)),
                            key=f"maxtok_{idx}"
                        )
                    
                    with col4:
                        agent["top_p"] = st.slider(
                            "Top P",
                            0.0, 1.0,
                            float(agent.get("top_p", 0.95)),
                            0.01,
                            key=f"topp_{idx}"
                        )
                    
                    agent["system_prompt"] = st.text_area(
                        "System Prompt",
                        agent.get("system_prompt", ""),
                        height=100,
                        key=f"sys_{idx}"
                    )
                    
                    agent["user_prompt"] = st.text_area(
                        "User Prompt (use {{input}} placeholder)",
                        agent.get("user_prompt", "{{input}}"),
                        height=120,
                        key=f"user_{idx}"
                    )
    
    # Tab 5: Run Pipeline
    with tabs[4]:
        st.header("▶️ Run Agent Pipeline")
        
        agents_to_run = (st.session_state.get("agents_config") or {}).get("agents", [])
        
        if not agents_to_run:
            st.warning("⚠️ No agents configured")
        else:
            st.markdown(f"**Pipeline: {len(agents_to_run)} agents**")
            
            # Input source
            st.subheader("📥 Input Source")
            input_source = st.radio(
                "Select input",
                ["Manual Text", "First Generated Doc", "All Generated Docs"],
                horizontal=True
            )
            
            if input_source == "Manual Text":
                initial_input = st.text_area("Input text", height=200, placeholder="Enter your text here...")
            elif input_source == "First Generated Doc":
                if st.session_state["generated_docs"]:
                    initial_input = st.session_state["generated_docs"][0]["content"]
                    st.text_area("Input preview", initial_input[:500] + "...", height=150, disabled=True)
                else:
                    st.warning("No generated documents available")
                    initial_input = ""
            else:
                if st.session_state["generated_docs"]:
                    initial_input = "\n\n---\n\n".join([d["content"] for d in st.session_state["generated_docs"]])
                    st.text_area("Input preview", initial_input[:500] + "...", height=150, disabled=True)
                else:
                    st.warning("No generated documents available")
                    initial_input = ""
            
            st.markdown("---")
            
            # Run button
            if st.button("🚀 Execute Pipeline", use_container_width=True, type="primary"):
                if not initial_input.strip():
                    st.error("❌ Input is empty")
                else:
                    st.session_state["agent_run_history"].clear()
                    current_input = initial_input
                    
                    # Create a container for real-time updates
                    pipeline_container = st.container()
                    
                    with pipeline_container:
                        for idx, agent in enumerate(agents_to_run):
                            agent_name = agent.get("name", f"Agent {idx+1}")
                            
                            # Show agent card
                            with st.status(f"Running {agent_name}...", expanded=True) as status:
                                st.write(f"🤖 Model: {agent.get('model', 'N/A')}")
                                
                                # Prepare prompts
                                sys_prompt = agent.get("system_prompt", "")
                                user_prompt_template = agent.get("user_prompt", "{{input}}")
                                
                                try:
                                    user_prompt = render_template_string(
                                        user_prompt_template,
                                        {"input": current_input}
                                    )
                                except:
                                    user_prompt = user_prompt_template.replace("{{input}}", current_input)
                                
                                # Call LLM
                                model = agent.get("model", agent.get("default_model", "gemini-1.5-flash"))
                                temp = float(agent.get("temperature", 0.3))
                                max_tok = int(agent.get("max_tokens", 1024))
                                top_p = float(agent.get("top_p", 0.95))
                                
                                output, latency, error = call_llm_unified(
                                    model, sys_prompt, user_prompt, temp, max_tok, top_p
                                )
                                
                                if error:
                                    status.update(label=f"❌ {agent_name} failed", state="error")
                                    st.error(f"Error: {error}")
                                    break
                                else:
                                    status.update(label=f"✅ {agent_name} completed in {latency:.2f}s", state="complete")
                                    
                                    # Store history
                                    st.session_state["agent_run_history"].append({
                                        "agent": agent_name,
                                        "model": model,
                                        "latency": latency,
                                        "input": current_input[:200] + "..." if len(current_input) > 200 else current_input,
                                        "output": output
                                    })
                                    
                                    # Show output with edit capability
                                    st.markdown(f"**Output ({len(output)} chars):**")
                                    edited_output = st.text_area(
                                        "Edit before passing to next agent",
                                        output,
                                        height=200,
                                        key=f"pipeline_edit_{idx}"
                                    )
                                    
                                    current_input = edited_output
                                    
                                    # Performance indicator
                                    if latency < 2:
                                        fancy_status_badge("⚡ Fast", "ok")
                                    elif latency < 5:
                                        fancy_status_badge("✓ Normal", "pending")
                                    else:
                                        fancy_status_badge("🐌 Slow", "warn")
                        
                        st.success("🎉 Pipeline completed!")
            
            # Show history
            if st.session_state["agent_run_history"]:
                st.markdown("---")
                st.subheader("📊 Execution Summary")
                
                for i, step in enumerate(st.session_state["agent_run_history"]):
                    with st.expander(f"Step {i+1}: {step['agent']} ({step['latency']:.2f}s)", expanded=False):
                        col1, col2 = st.columns(2)
                        
                        with col1:
                            st.markdown("**Input:**")
                            st.text(step["input"])
                        
                        with col2:
                            st.markdown("**Output:**")
                            st.text_area("Result", step["output"], height=150, key=f"history_{i}", disabled=True)
                        
                        st.download_button(
                            "⬇️ Download Output",
                            data=step["output"],
                            file_name=f"{step['agent']}_output.txt",
                            mime="text/plain",
                            key=f"dl_history_{i}"
                        )
    
    # Footer
    st.markdown("---")
    theme = FLORA_THEMES[st.session_state["theme"]]
    st.markdown(
        f'<div style="text-align:center;padding:20px;color:{theme["text"]}">'
        f'<p>🌸 Flora Edition by Agentic Docs Builder | Theme: <strong>{st.session_state["theme"]}</strong></p>'
        f'</div>',
        unsafe_allow_html=True
    )

if __name__ == "__main__":
    main()
