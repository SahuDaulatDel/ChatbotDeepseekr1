# ChatbotDeepseekr1
from flask import Flask, request, jsonify
from langchain_ollama import OllamaLLM
from langchain_community.utilities import SQLDatabase
import re
import time

app = Flask(__name__)

# -------------------------------
# ✅ ONLY SMART MODEL (ACCURATE) -- change this line for the llm change 
# -------------------------------
llm = OllamaLLM(model="llama3:8b", temperature=0)
# llm_smart = OllamaLLM(model="deepseek-r1") 

# -------------------------------
# ✅ DATABASE
# -------------------------------
db = SQLDatabase.from_uri(
    "mysql+pymysql://root:newcg%401234@localhost:3306/cggst"
)

# -------------------------------
# ✅ SQL CLEANER (FIXED)
# -------------------------------
def extract_sql(text):
    text = text.replace("```sql", "").replace("```", "").strip()

    # Start from SELECT
    start = text.upper().find("SELECT")
    if start == -1:
        return text

    text = text[start:]

    # Remove comments
    text = re.split(r"#|--", text)[0]

    # Keep only first query
    if ";" in text:
        text = text.split(";")[0] + ";"

    return text.strip()

# -------------------------------
# ✅ SQL SAFETY
# -------------------------------
def is_safe(sql):
    sql_upper = sql.upper()
    unsafe = ["DELETE", "DROP", "UPDATE", "INSERT", "ALTER", "TRUNCATE"]
    return sql_upper.startswith("SELECT") and not any(x in sql_upper for x in unsafe)

# -------------------------------
# ✅ SQL GENERATION (STRICT)
# -------------------------------
def generate_sql(query, tables):

    prompt = f"""
    You are a MySQL expert.

    STRICT RULES:
    - Return ONLY SQL query
    - NO explanation
    - NO comments
    - Query must start with SELECT
    - Use correct table and column names
    - Use ONLY the columns present in schema
    - dont use your assummed columns only use column which are exist in schema

    Tables: {tables}

    Question: {query}
    """

    raw_sql = llm.invoke(prompt)
    sql = extract_sql(raw_sql)

    print("🧠 Generated SQL:", sql)

    return sql
    

# -------------------------------
# 🚀 MAIN API
# -------------------------------
@app.route("/ask", methods=["POST"])
def ask():
    start_time = time.time()

    try:
        data = request.get_json()
        query = data.get("query")

        if not query:
            return jsonify({"error": "Query is required"})

        tables = db.get_usable_table_names()

        print("\n==============================")
        print("🧑 User Question:", query)

        # -------------------------------
        # STEP 1: GENERATE SQL
        # -------------------------------
        sql = generate_sql(query, tables)

        print("📌 Final SQL:", sql)

        if not is_safe(sql):
            return jsonify({"error": "Unsafe SQL generated", "sql": sql})

        # -------------------------------
        # STEP 2: EXECUTE
        # -------------------------------
        try:
            result = db.run(sql)

        except Exception as e:
            print("❌ SQL Error:", e)

            # 🔁 FIX (ONLY ONCE → avoid long wait)
            fix_prompt = f"""
            Fix this SQL:

            Error: {str(e)}
            SQL: {sql}

            Return ONLY corrected SELECT query.
            """

            fixed_sql = llm.invoke(fix_prompt)
            sql = extract_sql(fixed_sql)

            print("🧠 Fixed SQL:", sql)

            result = db.run(sql)

        print("📊 Result:", result)

        end_time = time.time()

        print("⏱ Time:", round(end_time - start_time, 2), "sec")
        print("==============================\n")

        return jsonify({
            "sql": sql,
            "data": result,
            "time_taken_sec": round(end_time - start_time, 2)
        })

    except Exception as e:
        return jsonify({"error": str(e)})

# -------------------------------
# 🚀 RUN SERVER
# -------------------------------
if __name__ == "__main__":
    print("🚀 Server started at http://127.0.0.1:5000")
    app.run(host="0.0.0.0", port=5000, debug=True)
