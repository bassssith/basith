import streamlit as st
import pandas as pd
import os

# Files for storage
MASTER_FILE = "cutting_master.csv"
PROD_FILE = "production_log.csv"
ADVANCE_FILE = "advance_log.csv"

# Function to load/save data
def load_data(file, cols):
    if os.path.exists(file): return pd.read_csv(file)
    return pd.DataFrame(columns=cols)

st.set_page_config(page_title="Al-Noor Smart Fit ERP", layout="wide")
st.title("🧵 Advanced Tailor Production & Salary System")

# Load Data
master_df = load_data(MASTER_FILE, ['Cutting No', 'Total Pcs'])
prod_df = load_data(PROD_FILE, ['Date', 'Tailor Name', 'Cutting No', 'Pcs Done', 'Rate', 'Total'])
adv_df = load_data(ADVANCE_FILE, ['Date', 'Tailor Name', 'Advance Amount'])

tab1, tab2, tab3, tab4 = st.tabs(["📦 Master", "🪡 Entry", "💵 Advance", "📊 Salary Report"])

# --- TAB 1: MASTER (Limit Set panna) ---
with tab1:
    st.subheader("Set Piece Limit per Cutting")
    with st.form("master"):
        c_no = st.text_input("Cutting Number").upper()
        t_pcs = st.number_input("Total Pieces", min_value=1)
        if st.form_submit_button("Save Master"):
            new_m = pd.DataFrame([[c_no, t_pcs]], columns=master_df.columns)
            master_df = pd.concat([master_df, new_m], ignore_index=True)
            master_df.to_csv(MASTER_FILE, index=False)
            st.success(f"Limit set for {c_no}")

# --- TAB 2: PRODUCTION ENTRY (Validation-oda) ---
with tab2:
    st.subheader("Daily Work Entry")
    with st.form("prod", clear_on_submit=True):
        t_name = st.text_input("Tailor Name").capitalize()
        sel_c = st.selectbox("Select Cutting No", options=master_df['Cutting No'].unique())
        pcs = st.number_input("Pieces Done", min_value=1)
        rate = st.number_input("Rate per Piece", min_value=0.0)
        if st.form_submit_button("Submit Work"):
            # Validation Logic
            limit = master_df[master_df['Cutting No'] == sel_c]['Total Pcs'].values[0]
            done = prod_df[prod_df['Cutting No'] == sel_c]['Pcs Done'].sum()
            rem = limit - done
            
            if pcs > rem:
                st.error(f"❌ Limit Exceeded! Only {rem} pieces left in {sel_c}.")
            else:
                amt = pcs * rate
                new_p = pd.DataFrame([[pd.Timestamp.now().date(), t_name, sel_c, pcs, rate, amt]], columns=prod_df.columns)
                prod_df = pd.concat([prod_df, new_p], ignore_index=True)
                prod_df.to_csv(PROD_FILE, index=False)
                st.success(f"Saved! {rem-pcs} pieces remaining.")

# --- TAB 3: ADVANCE (Kaasu munnadi kudutha) ---
with tab3:
    st.subheader("Record Advance Payment")
    with st.form("advance"):
        a_name = st.text_input("Tailor Name").capitalize()
        a_amt = st.number_input("Advance Amount (₹)", min_value=0.0)
        if st.form_submit_button("Save Advance"):
            new_a = pd.DataFrame([[pd.Timestamp.now().date(), a_name, a_amt]], columns=adv_df.columns)
            adv_df = pd.concat([adv_df, new_a], ignore_index=True)
            adv_df.to_csv(ADVANCE_FILE, index=False)
            st.success(f"Advance of ₹{a_amt} recorded for {a_name}")

# --- TAB 4: SALARY REPORT (Net Payable) ---
with tab4:
    st.subheader("Individual Salary Summary")
    names = list(set(prod_df['Tailor Name'].unique()) | set(adv_df['Tailor Name'].unique()))
    sel_tailor = st.selectbox("Select Tailor", options=["All"] + names)
    
    if sel_tailor != "All":
        earned = prod_df[prod_df['Tailor Name'] == sel_tailor]['Total'].sum()
        taken = adv_df[adv_df['Tailor Name'] == sel_tailor]['Advance Amount'].sum()
        payable = earned - taken
        
        c1, c2, c3 = st.columns(3)
        c1.metric("Total Earned", f"₹{earned}")
        c2.metric("Advance Taken", f"₹{taken}")
        c3.metric("Net Payable", f"₹{payable}", delta_color="normal")
        
        st.write("Work History:")
        st.dataframe(prod_df[prod_df['Tailor Name'] == sel_tailor])
