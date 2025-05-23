import streamlit as st
import pandas as pd
import re

st.set_page_config(page_title="CSV Validator", layout="centered")
st.title("📄 Smart Import Template Validator")

# Helper functions
valid_transaction_types = ["deposit", "withdrawal", "tradeFee", "tradeAcquire", "tradeDispose"]

def is_valid_time_format(time_str):
    return bool(re.match(r"^(0[1-9]|1[0-2])/([0-2][0-9]|3[01])/\d{4} ([0-1][0-9]|2[0-3]):[0-5][0-9]:[0-5][0-9]$", str(time_str).strip()))

def validate_row(row, id_set, grouped_blockchain_ids):
    errors = []
    warnings = []

    if not row["id"] or pd.isna(row["id"]):
        errors.append("Missing 'id'")
    elif row["id"] in id_set:
        errors.append("Duplicate 'id'")
    else:
        id_set.add(row["id"])

    if not row["amount"] or pd.isna(row["amount"]):
        errors.append("Missing 'amount'")
    elif "," in str(row["amount"]):
        errors.append("'amount' should not contain commas")

    if not row["amountTicker"] or pd.isna(row["amountTicker"]):
        errors.append("Missing 'amountTicker'")

    if not row["time"] or pd.isna(row["time"]):
        errors.append("Missing 'time'")
    elif not is_valid_time_format(row["time"]):
        errors.append("Invalid 'time' format (should be MM/DD/YYYY HH:MM:SS)")

    if not row["transactionType"] or pd.isna(row["transactionType"]):
        errors.append("Missing 'transactionType'")
    elif row["transactionType"] not in valid_transaction_types:
        errors.append("Invalid 'transactionType'")

    if not row["accountId"] or pd.isna(row["accountId"]):
        errors.append("Missing 'accountId'")

    # taxExempt must always be FALSE
    if str(row["taxExempt"]).strip() != "FALSE":
        errors.append("Invalid 'taxExempt' (must be 'FALSE')")

    # Fee requires feeTicker
    if row.get("fee") and not pd.isna(row["fee"]):
        if not row.get("feeTicker") or pd.isna(row["feeTicker"]):
            errors.append("'feeTicker' is required when 'fee' is present")

    # Warnings
    blockchain_id = row["blockchainId"]
    transaction_type = str(row["transactionType"]).lower()
    if blockchain_id in grouped_blockchain_ids and transaction_type in ["deposit", "withdrawal"]:
        if not row["groupId"] or pd.isna(row["groupId"]):
            warnings.append("Missing 'groupId' for grouped deposit/withdrawal")

    if transaction_type in ["tradeacquire", "tradedispose", "tradefee"]:
        if not row["tradeId"] or pd.isna(row["tradeId"]):
            warnings.append("Missing 'tradeId' for trade transaction")

    return "; ".join(errors), "; ".join(warnings)

# Upload file
uploaded_file = st.file_uploader("Upload your import template (.csv)", type=["csv"])
if uploaded_file is not None:
    df = pd.read_csv(uploaded_file)

    if df.empty:
        st.warning("Uploaded file is empty.")
    else:
        id_set = set()
        df = df[pd.notna(df["id"])]  # Only validate rows where 'id' is not empty
        df["taxExempt"] = "FALSE"  # Force taxExempt to be FALSE

        # Identify blockchainIds that are shared and include deposit/withdrawal types
        deposit_withdrawal = df[df["transactionType"].isin(["deposit", "withdrawal"])]
        grouped_blockchain_ids = deposit_withdrawal["blockchainId"].value_counts()
        grouped_blockchain_ids = set(grouped_blockchain_ids[grouped_blockchain_ids > 1].index)

        validation_results = df.apply(lambda row: validate_row(row, id_set, grouped_blockchain_ids), axis=1)
        df["Validation Errors"] = [res[0] for res in validation_results]
        df["Validation Warnings"] = [res[1] for res in validation_results]

        st.success("Validation complete. See results below:")
        st.dataframe(df)

        # Download link
        csv = df.to_csv(index=False).encode('utf-8')
        st.download_button("📥 Download Validated CSV", data=csv, file_name="validated_template.csv", mime="text/csv")
